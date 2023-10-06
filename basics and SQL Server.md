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
- 