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