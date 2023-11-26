# Architecture

## Logical architecture
- `Cluster`: a collection of databases in a server
- `Database`: we can create multiple databases for different purposes
- `Schema`: is a logical unit or a namespace that is a collection of tables, views, functions, indexes...
- `Object`: table, index, trigger, function ... each object has a unique identifier. To get list of database
```sql
select * from pg_database;
```
- To get list of objects
```sql
select * from pg_class where reffilenode=;
```

## Physical architecture
- Each database cluster has a base directory, this contains all files of the database.
- In the data/base directory, there are directories of each database with their corresponding oid
- The data/global contains all common objects used to serve the system.
- There are 3 types of files: connection config, metadata and database config.

### Connection config files: which IPs can access, authentication types ...
- There are 2 important connection config files: `pg_hba.conf` and `pg_ident.conf`
- pg_hba.conf: usually when firstly installed, a database cluster can only be accessed from local, to be accessible from other IPs, we need to configure pg_hba.conf file.
- pg_ident.conf: in case we use an external authentication method other than username/password, we need to use this file to configure.

### Metadata: 
- PG_VERSION: the version of postgre
- current_logfiles: when encountering issue, we need to use this to find the current log file of the database
- postmaster: the processes started by postgre when the database clustered starts, usually we don't have to deal with this.
 
### Database configuration files
- postgresql.conf: the MOST IMPORTANT file, it contains the configuration of the whole database like shared memory, number of connections.
- postgresql.auto.conf: when we run command ALTER SYSTEM to configure, this file will store the parameters. If both conf contain the same parameters, the auto.conf file takes precedence. This file is edited by the system and we MUST NOT manually edit it.

### Tablespaces
- Tablespaces are collections of objects in postgres, it's a physical way to bundle objects
- By default, Postgre creates 2 tablespaces pg_default and pg_global
- The pg_global contains all shared objects used by the system, for example, the table pg_tables has all tables of all schemas.
- The pg_default contains all user's data.
- We should create our own tablespaces to store our data for each business area.
- Doing so by creating a new directory anywhere, the create new tablespace with the path to this directory.
```sql
create tablespace myspace owner postgres location 'data/myspace'
```
- When we create new database, we can specify the tablespace or the directory we want to store our data in.
```sql
create database linh_db tablespace myspace;
create table foo(i int) tablespace myspace;
```
- In Linux or BSD, make sure the postgres user has access to the tablespace directory using `sudo chown postgres`

# Interacting with databases

## Table
- Select data from table
- Create a new table.
- Create table from another table
```sql
create table countries_new as select * from countries
```
- Join, inner join, outer join, left, right outer join, full outer join.
- Select join with left rows not match right rows
```sql
--return countries that don't match any regions
select * from countries c left join regions r on c.region_id = r.region_id where r.region_id is null
```
- Insert, delete, update.

## Tuning
- Execution plan: to get execution plan of each query, add explain at the beginning.
```sql 
explain select * from employees
--"Seq Scan on employees  (cost=0.00..3.07 rows=107 width=72)"
```
- In this case the system scans the whole table, 107 rows at the cost of 3.07. Tuning is to minimize the execution costs
```sql
explain select sum(salary) from employees where job_id = 'IT_PROG';
"  ->  Seq Scan on employees  (cost=0.00..3.34 rows=5 width=5)"
```
- The biggest cost of this query is in the sequential scan of the whole table, we can improve the performance by creating a new index with index key being the job_id
```sql
create index idx_employees_job on employees(job_id)
```
- Now the performance is increased because instead of using sequential scanning, the system uses index scanning.
- However there are cases when the system chooses sequential scanning over index because it's faster.

## Backup and Recovery

### Backup
- The simplest way is to export a table to csv files 
```sql
copy employees to '/Users/laxuanlinh/Downloads/employees-bk.csv' delimiter ',' csv header;
```
- We can also backup using sql format which can retain table structure and data.
- We can use pgAdmin or pg_dump to generate dump sql files

### Recovery
- We can import the csv files
```sql
create table employees
copy employees from '/Users/laxuanlinh/Downloads/employees-bk.csv' with (format csv)
```
- Or we can restore from the sql file.

### Check if a table doesn't have enough indexes
- To check if a table doesn't have enough indexes, we need to see if its queries have `seq scan` in their execution plans
```sql
select * from pg_stat_all_tables where schemaname = 'public' and seq_scan > 0;
```