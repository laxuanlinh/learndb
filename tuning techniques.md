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

### Order by
- `Order by` is a `costly` operation
- If the index that being has already sorted the `Order by` column then the system doesn't have to sort again
- If `Order by` descending, the system simply can just reverse the scan and doesn't cost any more than `Order by` ascending

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

### Index Full scan and Fast full scan
