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