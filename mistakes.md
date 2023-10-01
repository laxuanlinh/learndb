## Sequence of query execution in database
### Check for syntax error
- Just regular SQL syntax error
### Check for context error
- Whether the table or the schema exists
### Check the share pool
- When an SQL query is execute, the database breaks it down and figure out the execution plan, this takes time and resource.
- The database then stores the execution plan in a `cache` 
- The next time another query is excuted, it checks the `cache` to see if the execution plan already exists.
- How ever the database uses a hash function to determine the query so if 2 almost same queries are executed, the database still sees them as 2 different queries.
```sql
--these are considered 2 different queries
--and the database parse the execution plan again.
SELECT * FROM USERS WHERE first_name = 'Linh'
SELECT * FROM USERS WHERE first_name = 'Long'
```
- Each query after being hashed has a unique `SQL_ID` and `hash value` (Oracle)
- How ever after parsing, the 2 similar queries will have the same `plan hash value` because their execution plans are the same, as expected
### Parse the query to execution plan.
- If the execution plan already exists then the database just need to get it from the `share pool`, this is a `soft parse`
- Else it has to figure out the execution plan from the beginning and this is a `hard parse`

### Mistake
- When a system consecutively executes many queries that have almost the same syntax, the database still has to figure out the execution plan for each query which costs time.
```sql
SELECT * FROM users WHERE user_id = 1;
SELECT * FROM users WHERE user_id = 2;
SELECT * FROM users WHERE user_id = 3;
...
```
- We can fix this by passing named parameters instead
```sql
SELECT * FROM users WHERE user_id = $id
```

## Abusing HINT
- `Hints` is a feature to override execution plans that the optimizer might choose for queries.
- There are many types of `hint` to optimize but one of the most popular one is `index hint` in which we force the optimizer to use indexes instead of tables
```sql
--it's kinda difficult to replicate in PostgeSQL
SELECT * FROM emp e WHERE e.salary < 500000;
CREATE INDEX IDX_EMP_SALARY ON emp (salary);
--hint
SELECT /*+ INDEX(e, IDX_EMP_SALARY) */ * FROM emp e WHERE e.salary < 5000000
```
- How ever sometimes it takes much more time to scan index than full table scan and we should not abuse hint to force the optimizer
- If we think the execution plan chosen by the optimizer is not optimal, we should find other ways to provide the optimizer with better inputs to come up with better execution plan.

## Statistics Gathering Strategy
- To come up with an execution plan, the database has to collect statistics about tables (size, number of rows, columns, blocks), index, and system, we can ignore system.
- Most of the mistakes are made on databse objects.
- Database has a job to gather statistics periodically.
- By default in Oracle this job runs once from aroud 10PM - 2AM, this is only suitable for smaller system.
- SQL Server also has options to collect statistics
- PostgreSQL updates statistics when some of the commands like `VACUUM`, `ANALYZE` and `CREATE INDEX`
### Case study: Jobs that updates objects after gather statistic job runs
- A bank has 2 tables to store transactions, table DAILY stores data of a day.
- By the end of the day, all data of DAILY is dumped to table HISTORY, this helps keeps the size of DAILY table always small. 
- How ever the job to dump data runs from 3-4AM. 
- At 2AM, the gather statistics job runs and collects the size of DAILY table as 2mil records. 
- In the morning, the database still treats DAILY table as a 2-mil-record table even though it was emptied at 4AM

### Case study: Very large objects (tables, indexes)
- Some tables are `20TB` in size
- By default, the gather statistics job only has `4 hours` to collect `(10PM-2AM)`
- Due to the large size of these objects, the job cannot make it on time and fails.
- The statistics that were gathered before become obsolete.
- The optimizer creates wrong execution plans based on the obsolete statistics that were gathered a long time ago when the tables were not large
- We must make sure that the statistics collected are the latest.

## Data types
- Wrong data types could lead to database has wrong execution plans.
- Example: we have an Employees table
```sql
CREATE TABLE employees (
    id serial primary key,
    first_name varchar(100),
    --wrong type
    salary varchar(10)
)
```
- If we create an index based on the salary, the queries with where salary condition will still not scan from the index
```sql
CREATE INDEX idx_emp_salary ON employees (salary);
--this will still scan the whole table
SELECT * FROM employees WHERE salary = 40000;
```
- This is because the salary is varchar and has to be parsed to number while the index itself only has salary column as varchar, thus the optimizer cannot query from there.
- `Solution 1`: Update the data type of the salary column, however this could affect the operation of the database, especially if it's critical.
- `Solution 2`: Query with a varchar, this however doesn't work if we want to use math operations like > or <
```sql
SELECT * FROM employees WHERE salary = '50000';
```
- `Solution 3`: Create index from parse function. Since the database in the background has to parse salary to number anyway, we can create an index from this function.
```sql
CREATE INDEX idx_emp_salary ON employees (CAST(salary AS INT));
--remember to cast since we create the index from cast function.
SELECT * FROM employees WHERE CAST(salary AS INT) = 5000;
```

## Parallelism on too many objects
- Parallelism is a feature to enable multicores for a single query.
- In Oracle, to enable, we can add PARALLEL along with the number of CPU cores to improve performance.
```sql
ALTER TABLE employees PARALLEL 4;
---or for a session, table, index, create table...
ALTER SESSION ENABLE PARALLEL DML;
```
- In PostgreSQL, parallelism is enabled by default with number of workers = 2, we can increase the number per gather using `SET max_parallel_workers_per_gather=4`
- In SQL Server we can check and update `MAXDOP`.
- While this significantly improves performance, if abused, it could lead to issue when multiple sessions access the same parallel-enabled table and cause the CPU to spike.
### Case study 1: Index parallel
- In many databases, indexes are used frequently and there are a lot of indexes for each column.
- When rebuilding indexes with parallel to speed up the rebuild process, this parallel is also set to the indexes by accident
```sql
--Oracle
ALTER INDEX idx_emp_salary REBUILD PARALLEL 16;
```
- After rebuidling, the index `idx_emp_salary` will have PARALLEL = 16 even though we never set it.
- The correct way to rebuild parallel an index is to reset after it's complete
```sql
ALTER INDEX idx_emp_salary NOPARALLEL;
```

## Unusable, invisible indexes 
- Unusable are broken indexes that cannot be used and must be rebuilt
- There are multiple reasons for an index to break, it could be because 
    - SQL loader fails to update the index because it runs out of space 
    - Failure midway when building index
    - Unique key has duplicate
    - ...
- Invisible are indexes that the database cannot see so it doesn't consider when constructing execution plan.
- As databases are passed down for generations of developers, there is a risk that at some time an index is set to invisible

## No partitions or wrong partitioning for large tables
- When the database is not changed but becomes slower over time, it could be due to the data getting larger but we don't have proper partitioning.
- When we manually create partititions based on a certain criteria, the data is inserted and searched in the parititions corresponding to the query's range, how ever if it doesn't find the partition, it has to insert to the `partition max`
- Overtime if we forget to create new parittions, this partition max becomes larger and partitioning is useless.
- We need to check or better yet, automate the partitioning process so we never forget about it.

## Improper creation of temp tables
- Sometimes we create temp tables for calculating purposes but we create it improperly using standing CREATE TABLE command then use DELETE after done with it.
- All databases support CREATE TEMP TABLE and the system will delete automatically after the table is no longer used.
- TEMP TABLE can be indexed and created by separated sessions.
- In SQL Server, can create temporary tables using `SELECT INTO #table_name` or `CREATE TABLE #table_name`, note the `#` before table name
- In PostgreSQL, can create using `CREATE TEMP TABLE table_name`
- Temp tables are automatically deleted after we disconnect sessions, `ON COMMIT DROP` or we can manually `DROP TABLE`

## Tablespace and Filegroup
- In PostgreSQL, to check tablespace
```sql
\db
        List of tablespaces
    Name    |   Owner    | Location 
------------+------------+----------
 pg_default | laxuanlinh | 
 pg_global  | laxuanlinh | 
(2 rows)
```
- Oracle
```sql
SELECT DISTINCT owner, tablespace_name from dba_segments order by tablespace_name

```
- In SQL Server 
```sql
SELECT o.[name], o.[type], i.[name], i.[index_id], f.[name] FROM sys.indexes i
INNER JOIN sys.filegroups f
ON i.data_space_id = f.data_space_id
INNER JOIN sys.all_objects o
ON i.[object_id] = o.[object_id] WHERE i.data_space_id = f.data_space_id
AND o.type = 'U' -- User Created Tables
GO
```
- We need to check who owns these tablespaces, for example, user should not own the SYSTEM tablespace or there shouldn't be a generic DATA space where all data is stored.
- Tables, indexes should be stored in different specified tablespaces.
- To create tablespace in Postgres
```sql
CREATE TABLESPACE 'tablespace_name' OWNER 'owner' LOCATION 'directory';
```
- To move a table to a new space for both Oracle and PostgreSQL
```sql
ALTER TABLE 'table_name' TABLESPACE 'tablespace_name'
--or create table in a tablespace
CREATE TABLE 'table_name' TABLESPACE 'tablespace_name';
```
- For index, we need to rebuild in Oracle
```sql
ALTER INDEX 'idx_name' REBUILD TABLESPACE 'tablespace_name' | PARALLEL | NOLOGGING | ONLINE;
```
- For PostgreSQL it's a bit similar
```sql
ALTER INDEX 'idx_name' SET TABLESPACE tablespace_name;
```
- For large objects (`LOB`), they're not stored the same as tables so we need to move them separately, we can create a separate tablespace for `LOB`
- For partitioned tables, we need to list out all partitions and move them using name to tablespaces.
- We can create tablespace `HISTORY` to store older partitions while newer are stored in `DAILY` tablespace.

## Database lifecycle management
- We need to have a comprehensive database lifecycle management strategy.
- There are a few types of data in a database
    - Config data (features, entitlements)
    - Master data (users, accounts)
    - Transaction data
- Transaction data is more likely to grow fast so we need to know which data range is frequently updated, which is less updated and which is read only.
- For example, table transaction, data in the last 3 months is considered `HOT` so its partitions are stored in better I/O disks.
- Partitions in the last 3 years can be stored in slower disks
- Partitions from 10 years ago can be set to `READ ONLY`, can be backed up only once and exclude from daily backup because it's never touched again.

## Database fragmentation
- Fragementation in database is when we do update or delete and create spaces within the table.
- As we insert new data to Oracle, High Water Mark moves upward to claim new data block instead of inserting to the spaces, this leads to the size of table increases and wasted space.
- We need to routinely check the fragmentation of tables and indexes in database to make sure they're not too fragmented.
- To fix the fragmentation:
    - For table we can move table to the same `TABLESPACE` to automatically reclaim space.
    - Export then import table
    - Or can `ALTER TABLE SHRINK` which takes longer but not prone to locks if there are `DML` operations
- For indexes, we can `ALTER INDEX REBUILD`.

## Not analyze index effectiveness
- Create too many indexes but some of them are never used, this leads too `SELECT` queries are not faster but `DML` are slower.
- If we find an index that is never used, we can set it to `INVISIBLE` for a while and monitor the system's performance.
- If we are confident that the index does not affect performance, we can backup the index creation script, then `DROP` it

## Edit Store Procedures Online
- Editing Store procedures online is dangerous and could lead to the whole system to go down.
- We should only edit store procedures when the system is not accessed by any sessions, basically we need to take the system down for maintainance.

## Put user and system data in the same tablespace
- We should put system config data and app data in separate tablespace to avoid conflicts.

## Resource conflict
- For systems that has multiple databases and apps on the same servers, when a component has an issue with resource, for example, uses up too much CPU or IO, could lead to other components to hang.
- We should separate databases and apps into different servers.

## Abuse DB Links
- When a database needs to query data from another database, we can create a DB link.
- This might serve well if we only have a few links but can cause issue when the number of links grow.
- The execution plans are not optimal because the database doesn't know the statistics of other databases.
- It's also more difficult to rollback if a transaction fails.
- When there are many database links, we should consider other technologies like data synchronization.

## Sequence cache
- Sequence is a feature to generate unique integer
```sql
CREATE SEQUENCE testing1;
SELECT NEXTVAL('testing1'); --1
SELECT NEXTVAL('testing1'); --2
SELECT NEXTVAL('testing1'); --3
```
- Sequence cache is a memory to store the 
- By default cache can store 20 values but it can also be set to 0 which is bad.
- We should check any of the sequences has a cache size = 0 and increase it.

## Underutilized reserve resources
- We usually have standby databases that synchronized from main database but it's only used when the database goes down.
- We should utilize the standby database as read-only database for reporting.