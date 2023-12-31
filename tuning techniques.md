# Tuning Technique

## Index
- [Basics about Index](./basics%20and%20SQL%20Server.md##Index)
- There are many types of indexes:
  - Btree: most popular
  - Bitmap 
- For Btree indexes, index range scan works as follow:
  - If the where condition is greater operator, the system will find the value then scan all the right leaves, because in Btree, right is always greater than left
  - If where condition is lesser operator, the system will find the value and scan all left leaves
  - Because leaves are connected with 2-way link, the system doesn't have to jump back to the branch to scan the next leaf
- Modern databases use CBO (Cost Based Optimizer) to decide which execution plan works best.
- Usually if a query has result larger than `10%` of overall table size then scanning index and key look up will cost more than just full table scan.
- The most ideal situation is result size is only about `1-2%`

### Leading column
- When we create indexes, the first column is the most important column, also called leading column
- This will dictate how the system decide to scan the index or table.
- For example the following query
  ```sql
  select * from employees where salary > 10000 and name = 'Linh';
  ```
- If we have an index with composite key (salary and name), everything works perfectly, the system will scan the index.
- However if the query only looks for `name` then it's very likely that the system only `index skip scan` or scan the whole table instead
  ```sql
  ---skip scan or table scan
  select * from employees where name = 'Linh';
  ```
### Index skip scan
- It's a feature of Oracle database that allows the optimizer to scan an index without the leading column specified in `WHERE` condition
- Oracle does this by first get the distinct values of leading columns in the index, then it combine these distinct values with other conditions in `WHERE` 
- Example we have the following index
  ```sql
  create index idx_emp_name_salary_age on employees (name, salary, age)
  ```
- And a query
  ```sql
  select * from employees where salary = 50000 and age = 30
  ```
- Even though the `WHERE` conditions don't have name, the optimizer can still use this index by getting all distinct values of name, then combine with value 50000 and 30
| Name  | Salary | Age |
| :---: | :----: | :-: |
|  An   | 50000  | 30  |
| Binh  | 50000  | 30  |
| Chinh | 50000  | 30  |
- If we have 3 combinations, the optimizer will scan the index 3 times
- To decide whether to `skip scan` or `scan table`, the optizer relies on the `statistics`
- This techninque only works when the `leading column` has `few distinct values` and it's still `slower` than `normal index scan`
- However in some specific cases, this could still be more efficient than `table scan` and it helps reduce the number of indexes

### How to pick columns
- We need to see which queries are used frequently and their `WHERE` conditions
- The column that have the `highest number of distinct values` should be chosen as the leading column because it will filter out `highest number of unsatisfied rows`
  ```sql
  --10k 
  select count(distinct name) from employees;
  --30k
  select count(distinct salary) from employees;
  --salary should be the leading column
  ```
### Function-based index
- When a query has WHERE conditions that use function like uppercase, the index has to have the key with the same function as well
  ```sql
  select * from employees where uppercase(name) = 'LINH';
  ```
- Then we need a function-based index in order for the optimizer to scan the index
  ```sql
  create index idx_upper_name_emp on employees (uppercase(name));
  ```

### Index full scan and Fast full scan
- `Fast full scan` is similar to table fullscan but on indexes instead.
- Normally `Fast full scan` features multi block or parallel scan
- The Fast full scan is utilized when the index already has all column that the query needs but the leading column is not in WHERE conditions, for example
  ```sql
  select salary, first_name from employees where salary > 10000
  ```
- If the index has `salary` as the leading column then the optimizer will do an `index range scan`
- If the index has `first_name` as the leading column, the optimizer can still scan the index but this time it has to do a `Fast full scan`
- Therefore the leading column will decide what type of index scan it is.
- We should always try to eliminate table scan because index scans are always faster
- The optimizer uses skip scan when the selected columns doesn't contain some columns that the index doesn't have
- If the index has all selected columns from the query then the optimizer does a fast full scan
  ```sql
  --create index with last_name as leading column
  create index idx_last_name_salary on employees (last_name, salary);

  --skip first_name and scan the index for last_name and salary then a table scan to get first_name
  select first_name, last_name, salary from employees where salary > 10000;

  --the index already has last_name and salary but salary is not the leading column so it does a fast full scan
  select last_name, salary from employees where salary > 10000;
  ```
- `Index full scan`, unlike `fast full scan`, is singly scan
- In case of `HAVING` and `GROUP BY`, indexes can still help improve performance but this depends on the costs and on how the data is clustered.
  ```sql
  --this could use index, depends on how salary is clustered
  select salary, count(*) from employees group by salary having count(*) > 3;
  ```

### Order by
- `Order by` is a `costly` operation
- If the index that being has already sorted the `Order by` column then the system doesn't have to sort again
- If `Order by` descending, the system simply can just reverse the scan and doesn't cost any more than `Order by` ascending
- We need to include the Order by column in indexes to improve performance
  ```sql
  select first_name, birthday, salary from employees where salary > 10000 order by birthday;

  --the following index helps improve ORDER BY's performance
  create index idx_emp on employees (salary, birthday, first_name)
  ```
### When to create indexes?
- `Primary / Unique key`: most databases create indexes on primary / unique constraint by default, this helps improve performance of key lookup
- `Foreign key`: databases don't create indexes on foreign keys by default but if we ever delete data from parent tables then creating indexes for foreign keys in child tables is a must. This prevents locks when multiple sessions execute DML like `UPDATE` or `DELETE`
- `Distinct`: indexes can also help improve performance on SELECT DISTINCT 
- `Covering index`: we can consider selected columns to be included as well if they're queried frequently and we need to select the leading column carefully, based on the `WHERE`, `ORDER BY` and `GROUP BY` condition

### Descending index
- When we `ORDER BY` multiple columns in descending order, the system will have to sort the result after scanning index
  ```sql
  --by default last_name is sorted in ascending order
  select * from employees where salary > 10000 order by first_name, last_name desc
  ```
- To prevent this, we can create descending index
  ```sql
  create index idx_emp on employees (salary, first_name, last_name desc)
  ```
- The order of `ORDER BY` columns also dictates whether the system has to sort the result

### Reverse key index
- This index helps solve the problem with hot objects, when multiple sessions doing `INSERT` to an index
- `Reverse key index` reverses the ID 123 -> 321, 124 -> 421, 461 -> 164 ...
- This helps spread the rows evenly between leaves and avoid multiple sessions accessing the same index block
  ```sql
  --Oracle only
  alter index idx_emp rebuild reverse
  ```
- However reverse indexes cannot do range scan as the IDs are unordered
  ```sql
  --full table scan
  select * from employees where salary > 10000
  --can index scan
  select * from employees where salary = 10000
  ```
### Bitmap index
- Bitmap index is suitable when there are not many unique values and the schema is designed as a `star schema`
- When we create a bitmap index on the central table, each unique value of other tables that have referenced columns in the central table is a column in the index
- For example we have table `employees`, `jobs`, `gender`
  | employees        | jobs             | genders             |
  | ---------------- | ---------------- | ------------------- |
  | ID int PK        | ID int PK        | ID int PK           |
  | name varchar     | job_name varchar | gender_name varchar |
  | job_id int FK    |                  |                     |
  | gender_id int FK |                  |                     |
- And create a bitmap index on `employees` and column `job_id`, each unique value is assigned to a column, if a row matches a value, the cell is 1, else it's 0.
  | ID  | developer Bitmap | DBA Bitmap | manager Bitmap |
  | --- | ---------------- | ---------- | -------------- |
  | 1   | 0                | 1          | 0              |
  | 2   | 1                | 0          | 0              |
  | 3   | 0                | 1          | 0              |
- Now the system can use AND and OR operator to find the satisfied rows.
- Bitmap index is suitable for data warehouse, reporting or star schema and the `unique values` are `< 1% `of total values
- The bitmap indexes should only be used with DML queries.

### Partition index
- When we parititon a table, we should create local indexes for each partition instead of using a global partition.
- We can subpartition on a partition by other columns
```sql
--Oracle
CREATE TABLE ts (id INT, purchased DATE)
    PARTITION BY RANGE( YEAR(purchased) )
    SUBPARTITION BY HASH( TO_DAYS(purchased) )
    SUBPARTITIONS 2 (
        PARTITION p0 VALUES LESS THAN (1990),
        PARTITION p1 VALUES LESS THAN (2000),
        PARTITION p2 VALUES LESS THAN MAXVALUE
    );
```
### Operators that prevent Index scan
- When we use `NOT IN` in the query, the optimizer might do a table scan instead of index scan because it believes this is too complicated
  ```sql
  select * from emp where salary not in (5000)
  ```
- To avoid this, we can avoid using `NOT IN` and use `IN` instead, this happens a lot when we query against `status` column
  ```sql
  --use this
  select * from txn where txn_status in ('PENDING', 'REJECTED');
  --instead of this 
  select * from txn where txn_status not in ('COMPLETED');
  ```
- Another case of index is not used is when we use `LIKE` with a prefix wildcard like `%` and `_`
  ```sql
  select * from emp where first_name like '%Linh';
  select * from emp where first_name like '_inh';
  ```
- This also happens if we use `IS NULL` because indexes cannot store null values
  ```sql
  select * from emp where first_name is null;
  ```
- We can fix this by pairing the nullable column with a value as the composite index key
  ```sql
  create index idx_first_name_emp on emp (first_name, 1)
  ```

### Index clustering factor
- When we use ORDER BY in a query, the behavior of optimizer can be different depending on the ORDER BY column, it could scan table or index
  ```sql
  --table scan
  select * from emp order by salary;
  --index scan
  select * from emp order by id;
  ```
- This is because tables only order data in inserting order so the key lookup cost could be huge, this is the `clustering factor`
- We can fix this by creating a new table and order data by a column
  ```sql
  create emp_new as select * from emp order by salary
  ```
### Index when delete table records
- When we delete table records, indexes don't delete their leaves but only make them empty.
  ```sql
  delete * from emp;
  analyze index idx_name validate structure;
  select height, lf_rows, lf_blks, del_lf_rows from INDEX_STATS;
  ```
  | HEIGHT | LF_ROWS | LF_BLKS | DEL_LF_ROWS |
  | ------ | ------- | ------- | ----------- |
  | 2      | 1000    | 4       | 1000        |
- This is to ensure the B-tree remains balanced
- If we delete some rows and update the leading rows then the index has to add more leaves and data blocks to store new values
  ```sql
  update emp set name = name || ' new suffix';
  analyze index idx_name validate structure;
  select height, lf_rows, lf_blks, del_lf_rows from INDEX_STATS;
  ```
| HEIGHT | LF_ROWS | LF_BLKS | DEL_LF_ROWS |
| ------ | ------- | ------- | ----------- |
| 2      | 1590    | 7       | 590         |
- The more we update the rows (update or delete), the more blocks are used even though the number of rows is unchanged
- This leads to fragmented indexes which decrease the performance of the indexes.
- We can fix this by REBUILD or SHRINK indexes
- REBUILD requires more temp space and resources, especially with large indexes, however during REBUILD process, the system updates the latest statistics and we can update the properties of the index like PARALLEL
- SHRINK requires less resources but it doesn't update statistics

## Partition
- Partition key is the most important factor
- There are 2 main types of partitioning methods
  - Regular partitioning: list, range, hash
  - Composite partitioning: combination of other types
### Range partitioning
```sql
create table sale_range 
(salesman_id number(5), 
salesman_name varchar(30), 
sales_amount number(10), 
sales_date date) 
partition by range(sales_date)
(
  partition sales_jan2000 values less than (to_date('02/01/2000', 'DD/MM/YYYY')),
  partition sales_feb2000 values less than (to_date('02/02/2000', 'DD/MM/YYYY')),
  partition sales_mar2000 values less than (to_date('02/03/2000', 'DD/MM/YYYY'))
)
```
- If we run out of parititions, we can `add paritition`
- Or we can use `partition max`

### Hash partitioning
- Instead of having growing number of partitions by range, we can divide into fixed number of equally distributed partitions using hash partitioning

### List partitioning
- Similar to range partitioning but instead of choosing a range of values, we choose the explicit values
```sql
create table employees (
  id int,
  first_name varchar(30),
  store_id int
)
partition by list(store_id) (
  partition pNorth values in (1, 3, 5, 7),
  partition pEast values in (2, 4, 6, 8)
  partition pWest values in (9, 10, 11)
)
```

### Composite partitioning or Subpartition
- We can combine other partitioning methods into 1 partition key
- Subpartition is when we partition a partition.
- This help with performance when queries on multiple columns.
- For example if we have a transaction table 
  ```sql
  create table txn (
    txn_id int primary key,
    branch varchar(30),
    txn_date datetime
  )
  ```
- We can partition by `branch` then by `txn_date` and improve performance of queries such as
  ```sql
  select * from txn where branch = 'Hanoi' and txn_date > '01-01-2023 00:00:000'
  ```
- For queries that only has 1 column, subpartitioning can still improve performance.

### Mistakes when using partition
- If the optimizer has to spend time converting data type in the queries, it will takes significant more time despite it still uses partitions
  ```sql
  select * from sales where time_id between '01-JAN-00' to '01-DEC-00' --takes much longer
  select * from sales where time_id between '01-JAN-2000' to '01-DEC-2000' --takes less time
  ```
- The same can happen if we use functions on the partition keys
  ```sql
  select * from sales where to_date(time_id, 'dd-mm-yyyy') between to_date ('01-04-2000', 'dd-mm-yyyy') and to_date ('01-04-2001', 'dd-mm-yyyy')
  ```
- If we update a record with a new value in the partition key column, it will take time to move this record from a partition block to another, we can mitigate this by deleting and adding a new record. However if we have to constantly update partition key columns then there are problems with our parititoning strategy

## Statistics
- The optimizer needs to collect the object, database and system statistics to calculate the best execution plans
- For object, it usually collects information about:
  - Size
  - Number of data block
  - Number of rows
  - Data distribution
  - Null values
  - Number of distinct values
- For system, it usually collects:
  - CPU
  - RAM
  - OS
  - Drive
- For database, we have other parameters. If we change a parameter in database, it might change the execution plans of the entire database
- We need to check if the optimizer has all the information it needs about the database such as size and number of data blocks.
- If a query which runs fine, suddenly runs slowly, first we need to check if the object statistics or database parameters have been changed, then the system statistics.
- The database parameters changes are usually in Alert logs (Postgres)
- We can also check the execution plan hash value to see when this plan is generated
- For object statistics, we need to check if they're the latest and correct (number of rows), check if the optimizer has statistics about `WHERE` and `JOIN` columns (distict values, null)

## Locks
- `Lock conflicts` usually happen with DML commands. 
- If insert query is slow then it's likely due to `Lock conflicts`
- If we have 2 sessions insert to the same tables at the same time
```sql
insert into emp values (1, 'Linh');
insert into emp values (1, 'Linh but session 2');
```
- The second command could be locked until the first command either commits or rollbacks because there is a unique constraint
- Triggers could also lead to locks, if a DML command triggers another DML command to a different table and that table is locked then the original DML command is locked as well, so we need to check if our tables are using triggers

### Multiple session lock case study
- Given a table with 250 records, accessed by 250 sessions, each session updates 1 different record, however they're all slow.
  | Row | Session        |
  | --- | -------------- |
  | 1   | Update row 1   |
  | 2   | Update row 2   |
  | ... | ...            |
  | 250 | Update row 250 |
- This happens because these 250 records are all stored in a `data block`
- When a transaction updates a row, the database allocate part of that same `data block` to manage the update, this happens to all other transactions which could lead to the `data block` runs out of space for update
- If we configure each data block to be 8K and it only has 200 slots for 200 transactions, then the other 50 have to wait.
- One way we can solve this is to increase data block size to 16K or 32K
- But a better way is to split the data to multiple `data blocks`

### Check database locks
- We can use the following query to list all locking queries
```sql
SELECT
'alter system kill session ''' || SID || ',' || s.serial# || ',@'||inst_id||''';' as "Kill Scripts" ,sid,username,serial#,process,NVL (sql_id, 0),
blocking_session,wait_class,event,seconds_in_wait
FROM gv$session s WHERE blocking_session_status = 'VALID'
OR sid IN (SELECT blocking_session
FROM gv$session WHERE blocking_session_status = 'VALID');
```
- The first column also provides the kill session command, or we can use the PID to kill the process at OS level.
- To show the lock history, for example 1 transaction locks other transactions, we can use the blocking session tree
  ```sql
  SELECT level,
        LPAD(' ', (level-1)*2, ' ') || NVL(s.username, '(oracle)') AS username,
        s.osuser,
        s.sid,
        s.serial#,
        s.lockwait,
        s.status,
        s.module,
        s.machine,
        s.program,
        TO_CHAR(s.logon_Time,'DD-MON-YYYY HH24:MI:SS') AS logon_time
  FROM   v$session s
  WHERE  level > 1
  OR     EXISTS (SELECT 1
                FROM   v$session
                WHERE  blocking_session = s.sid)
  CONNECT BY PRIOR s.sid = s.blocking_session
  START WITH s.blocking_session IS NULL;
  ```

## Fragmentation
- Table fragmentation happens when we delete data from tables
- When we insert data into table, it reaches a limit called `High water mark`
- However when we delete data, the `High water mark` doesn't go down and if we insert more data, instead of appending new data block, it scans the whole table to look for free data blocks, this leads to slowness
- One of the easiest way to defragmentation is to move table space
- When we move a table to another tablespace to the same tablespace, the system automatically defratments
- However this method requires downtime
- We also needs to rebuild indexes, preferably with `PARALLEL` enabled.

## Join
- Join order doesn't make a difference in performance, it only requires the system to parse the query again.
- When we `JOIN` multiple tables, the optimizer actually does the physical joins
  - `Nested loop join`
  - `Hash join`
  - `Sort merge join`
### Nested loop join
- When JOIN 2 tables, the optimizer chooses 1 table to be the `Driving table`
- For each element of the `driving table`, it scans the other table (`Inner table`) to find the matched rows, hence `nested loop`
- We can optimize JOIN queries using indexes and we need to select the leading columns correctly
  ```sql
  select * from emp e
  join dept d
  on e.dpt_id = d.dpt_id and e.salary > 500
  ```
- In this case the optimizer does the followings:
  1. Loop `dept` index to get `dpt_id`
  2. Access full `emp` table to find the correct `dpt_id` and `salary` and the rest of the information
  3. Access `dept` table to get the rest of the information
- The `Full access` to `emp` table is the most costly and we can improve it by creating index on the `dpt_id` and `salary` column
- If we simply create index on just 1 column `dpt_id` then the optimizer does the followings:
  1. Loop `dept` index to get `dpt_id`
  2. Scan the `emp` index and use `dpt_id` to filter the correct rows
  3. Access `emp` table to find the rows with `salary > 500` and the rest of the information
  4. Access `dept` table to find the rest of the information
- We can still improve this by creating index on both `dept_id` and `salary`
- Since accessing `emp` table to find rows with `salary > 500` is the most costly, we will make `salary` the leading column
  ```sql
  create index idx_emp_salary_dptid on emp (salary, dpt_id);
  ```
- Now the optimizer does the followings
  1. Loop `dept` index to get `dpt_id`
  2. Scan `emp` index, use `salary` to filter the correct `salary` rows, then find the correct `dpt_id` rows
  3. Access `emp` table with ID lookup to find the rest of the information
  4. Access `dept` table to find the rest of the information
- This costs so much less because most of the lookup and filter work is done by indexes already
- However if we choose dpt_id before salary
  ```sql
  create index idx_emp_dptid_salary on emp (salary, dpt_id)
  ```
- Now instead of range scan to filter `salary` in `emp` index, it has to `skip scan` to find these rows first, then find the `dpt_id` later, this is still fast but not as optimal
- However there is still a way to make this faster, as we can see, we still have to access both tables by ID lookup to find the rest of the information, we can improve this by selecting specific columns and create indexes on these columns
  ```sql
  select e.first_name, d.dept_name from emp e
  join dept d
  on e.dpt_id = d.dpt_id and e.salary > 500
  ```
- For `emp`, we need to create index on `salary`, `dpt_id` and `first_name`
  ```sql
  create index idx_emp on emp(salary, dpt_id, first_name)
  ```
- For `dept`, we need to create index on `dpt_id` and `dept_name` so it doesn't have to ID look up the `dept` table to get the rest of the data.
- Because `dpt_id` is the primary key, the combination of `dpt_id` and `dept_name` is also unique, so we can create a unique index here, which makes it faster
  ```sql
  create unique index idx_dept on dept(dpt_id, dept_name)
  ```
- So now the optimizer does the followings:
  1. Range scan `dept` index to get `dpt_id` to scan and `dept_name`
  2. Range scan `emp` index, filter using `salary` and `dpt_id`, then it also gets `first_name`

### Hash join
- Used on large tables and data is unordered
- It can only be used with equal `WHERE` condition
- The database first selects a table and creates a hashed table in memory
- Then it loops through the second table and hash each value to find the matched hashed value
- This algorithm is very RAM consuming so it's usually used in systems with lots of RAM
- Hash join can shorten the first table because duplicate values will be hashed into 1 row so some times it's faster than nested loop join
- However because it has to create a hash table at the beginning, it's usually slower to retrieve the first element compare to nested loop join.

### Merge sort join
- The idea here is to sort both table first then use merge sort to filter out smaller or greater values so we don't have to scan the whole table
- Give 2 tables
| Table A | Table B |
| ------- | ------- |
| 3       | 2       |
| 5       | 7       |
| 1       | 2       |
| 5       | 8       |
| 2       | 4       |
| 3       | 7       |
- We first sort them
| Table A_temp | Table B_temp |
| ------------ | ------------ |
| 1            | 2            |
| 2            | 2            |
| 3            | 4            |
| 3            | 7            |
| 5            | 7            |
| 5            | 8            |
- Then we can compare the first element of `Table A_temp` to first element of `Table B_temp`
- Because 1 < 2, we stop looking and move to the next element in `Table A_temp` because we know for sure there is no value in `Table B_temp` equal to 1
- Then we move to 2, we find 2 rows in `Table B_temp` that are also 2
- When moving to the next row in `Table B_temp`, we find 4, which is > 2 so we stop and check the next row of `Table A_temp`
- Repeat and we have filtered out a lot of rows in both `Table A_temp` and `Table B_temp`