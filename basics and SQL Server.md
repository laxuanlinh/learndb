## Memory architecture
- In SQL Server, execution plans are stored in `Procedure Cache`
- Queried data is stored in `Buffer Pool`
- Changes to data are logged in `Log Buffer`(`Log Cache`)
- These 3 components have big impact on the performance of SQL Server databases

### Buffer Pool
- The size of `Buffer Pool` dictates how much temporary data it can store thus the performance of the query. 
- If the `Buffer Pool` is not big enough, we can use `Buffer Pool Extension`
- It's recommended to use SSD to extend the `Buffer Pool`, with size 4-8x the RAM.

### Total allocated memory
- In the later versions of SQL Server, the allocated memory is automatically allocated to other components.
- We need to pay attention to the total allocated memory (`Database` -> `Properties` -> `Memory`)
- By default the maximum memory is very large, we should set it to 75-85% of the RAM.

## Data files
- Data is written to disk on `data files`
- `.MDF` is `Master Database files` which store internal config used by system, this file is required to start database
- MDF by default also stores user's data which is not optimal
- `.NDF` is `Secondary Database Files`
- `SQL Server` allows to group data into `Filegroups` 

### Filegroup
- By default, there is `Primary Filegroup` which has `Master database files` (`.MDF`)
- We can create new filegroups to store user's data and it's easier to manage data lifecyle
- Having tables on different filegroups also improves performance when we have to join them, the I/O on 2 different disk partitions is faster.

## Transaction Log
- Changes of data in `Log Buffer` are written to `Transaction Log`
- `Transaction Log` logs the change before writing the change to data files or `Write-ahead logging (WAL)`
- Transaction Log is made of multiple smaller `Virtual Log File` (VLF)
- VLF have only 2 states:
  - Active: it's still being logged
  - Inactive: it's done logged and put away.
- When system writes to logs but there is no available log block to write, it will check `Autogrowth` setting and create a new log block.
- But if `Autogrowth` is not enabled, transaction logs is full and the transactions stop so we always enable `Autogrowth`.
- To enable `Autogrowth`, go to `System Datanbase` -> `tempdb` -> `properties` 
- However we cannot just grow transaction logs forever but also need to envict `Inactive` logs and this is configured in `Database Recovery Mode`
- There are 3 modes of `Database Recovery Mode`:
  - `Simple`: when system reaches a checkpoint, it automatically TRUNCATE Inactive logs, the size of growth is not large
  - `Full`: Inactive logs have to be backed up before the system automatically TRUNCATE, if we forget to backup, the system won't TRUNACATE and it keeps growing in size
  - `Bulk-logged`: Similar to `Full`, the only difference is it minimalize logs for commands like `BULK INSERT`

### Filegroup and Transaction Log I/O
- We should split the I/O of Primary filegroup, user's filegroups and `Transaction Log` by locating them into different disk drives
- The `Transaction Log` should be prioritized with the `fastest` drives
- We also need to have a backup strategy to avoid Transaction Logs getting too big.

## Table
- There are 2 types of tables:
  - `Heap`: unordered tables, without a clustered index
  - `Clustered index`: when a table is created, a `clustered index` might be created as well and the table is ordered by this index
- When we create a `clustered index` on a `Heap` table, the index itself becomes a new ordered table and all inserts are ordered as well. The original `Heap` no longer exists
- This is also the reason why there is only 1 `clustered index` per table since the `clustered index` is the table itself
- In SQL Server, tables are considered `partitions`
- Indexes should be partitioned the same as tables.
- If we partition a table by date column, we should partition the its indexes by the same date column

## Index
- There are 2 types of index
  - Clustered: each table can only have 1 clustered index
  - Non clustered: can by created on different columns of a table.
- All indexes are sorted using `B-tree` `(Self balanced tree)`
- A `B-tree` index has a root, intermediate levels and `leaf levels`
- `Leaf levels` are actual tables while the other tables are just intermediate to group leaves
- `Leaf levels` are connected with `double links`, this helps scan from 1 `leaf` to another without going back to the parent branch.
- When the `execution plan` does an `index scan`, meaning it will scan all `leaves`, only exception not to scan all `leaves` is when we `SELECT TOP`
- When we add a `WHERE` condition, the `execution plan` does an `index seek`, meaning it only searches for those condition then stops

### Clustered vs Non-clustered index
- `Clustered index`'s leaves contain all data of a row. 
- `Non-clustered index`'s leaves only contain `index key` and the `ID`.
- After finding the record according to the `WHERE` condition, the `Non-clustered index` uses the `ID` to look up in `Clustered index` for full information of the row
- The `ID lookup` process consumes a lot of resource and sometimes even more than the `index seek`, we can improve this by store all information of rows in the `Non-clustered index` so it doesn't have to lookup in the `Clustered index`
- `Covering index` is a regular `Non-clustered index` but it provides all data required by a query without looking up in the `Clustered index`
- To create a `covering index`, we need to use `composite index key` with all columns that a query needs
  ```sql
  CREATE UNIQUE INDEX idx_user_fn_ln ON schema1.users (first_name DESC, last_name ASC);
  ```