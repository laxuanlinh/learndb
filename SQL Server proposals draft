# SQL Server Optimization Proposal

Currently our database instance is being managed by 1 admin team, while this serves the purpose of security and consistency, it also poses some issues in term of performance for other projects who are using the database. We need to have a mechanism to implement some changes for specific needs of some teams in case their tables might get too large or they need some certain settings, while also to raise the awareness of developers about the challenges when the data grows.

## Filegroup 
By default SQL Server has a PRIMARY Filegroup which contains system tables and configuration. However currently all of user's data (schemas, tables, indexes) is in the same Filegroup as the system's data.
```sql
select * from sys.filegroups
```

This is not ideal because all schemas are now stored on the same disk which limits the I/O performance, especially when we execute complex `JOIN` queries from different tables at the same time.
https://learn.microsoft.com/en-us/answers/questions/960862/why-we-use-filegroups-what-are-the-benefits

I propose we move user's data to different Filegroups other than PRIMARY. Ideally we should have the following filegroups:s
- PRIMARY: contains only system's data and config, no one should ever touch this Filegroup. 
- SECONDARY/ User-defined filegroup: contains all tables and indexes, we can put it on a faster drive for better performance.
- Memory-optimized filegroup: can consider this option for tables that are frequently accessed, the In-memory feature of SQL Server can help improve performance greatly, for example this could be a Filegroup for all indexes.
- FILESTREAM filegroup: better performance for non-row tables such as LOB. Currently one of our biggest tables is `messenger.mail_attachment` which is a LOB table with a size of 6GB in UAT DB (I don't know how large it is in Prod) (https://www.mssqltips.com/sqlservertip/1489/using-filestream-to-store-blobs-in-the-ntfs-file-system-in-sql-server-2008/)


## Index
One of the problems with our database design is that we never use Non-clustered indexes to improve query performance. The SQL Server automatically creates a Clustered index for each table so we can avoid the HEAP problem but it can only get us so far.
The tables with the most rows in UAT is `document.doc_account` with over 29 millions rows. I tested this table with a simple query
```sql
select * from documents.doc_account where id_acc = 3000050;
```
This takes over 8 minutes (without cache) to complete. The execution plan is full scan the whole table with `Number of Rows Read: 29477328`, `Operator Cost: 132.921`
I then try creating a Non-clustered index for it 
```sql
create index idx_doc_account on documents.doc_account (id_acc);
```
And execute the query from `documents.doc_account` again. This time the execution plan does an index scan and an ID lookup. It only takes less than a second and the `Number of Rows Read: 1432`, `Operator Cost: 0.0035857`, meaning it's `37,069 times` more efficient and have to scan `10,292 times` less rows (combine both index scan and ID lookup).
For smaller tables like `omt.pay_txn` with only 14,729 rows, the execution plans still favor full table scan over Non-clustered index scan because the data is relatively small and ID lookup might be slower, but as it grows, especially when OMT EMEA goes online, full table scan will one day be very costly.
The benefits of indexes are clear and I think OMT team and other teams should be encouranged to use indexes to improve the query performance. 
To pick which tables, which columns to create indexes on, we should first identify which queries are most frequently executed
```sql
--top 50 most frequently executed queries
SELECT TOP 50
QueryState.execution_count
,OBJECT_NAME(objectid)
,query_text = SUBSTRING(
qt.text,
QueryState.statement_start_offset/2,
(CASE WHEN QueryState.statement_end_offset = -1
THEN len(convert(nvarchar(max), qt.text)) * 2
ELSE QueryState.statement_end_offset
END - QueryState.statement_start_offset)/2)
,qt.dbid
,dbname = db_name(qt.dbid)
,qt.objectid
FROM sys.dm_exec_query_stats QueryState
CROSS APPLY sys.dm_exec_sql_text(QueryState.sql_handle) as qt
ORDER BY QueryState.execution_count DESC
```
We also need to identify the most expensive queries using `Activity Monitor` and enable `Query Store` (I'm not sure if we don't enable Query Store or I'm not allowed to view it)
https://www.sqlshack.com/how-to-identify-slow-running-queries-in-sql-server/

As a best practice for developers, we should only select the columns we need so that we can create covering indexes which will avoid table lookup and greatly improve performance. For example, in the case of OMT
```sql
select id_login, cd_risk_status, cd_currency, id_acc, am_txn from omt.pay_txn where id_txn = :id_txn
```
We can create an index with composite key which returns result immediately without going back to the table for look up.
```sql
create index idx_pay_txn on omt.pay_txn (id_txn, id_login, cd_risk_status, cd_currency, id_acc, am_txn)
```

## Partition and Data lifecycle
Partitioning is important to manage growth and from what I can tell from the UAT database, we don't have this set up yet. I propose each project should analyze which tables can be partitioned and which data range can be put away on a slower disk for archive/readonly purposes. This can free up a lot of space and improve performance.
Take `omt.pay_txn` table as an example, since data is inserted chronogically with a transaction date column (`dt_txn`), we can partition it yearly. The data from more than 3 years ago can be put to archive disks and set to READONLY mode (optionally). 
```sql
select pt.cd_currency, sum(pt.am_txn) as am_sum from pay_txn pt where pt.id_login = (:id_login) and datediff(dd, GetDate(), pt.dt_txn) = 0 and (pt.cd_txn_type <> 'B' or (pt.cd_txn_type = 'B' and pt.is_cross_spn = 'Y')) group by pt.cd_currency
```
Queries like this can benefit greatly from partitioning since `dt_txn` is one of the WHERE conditions, even without an index.
To take full advantage of partitions, we also need to remember to create local indexes for them instead of using global indexes.

Since each database has to be configured to add Filegroups for different years and the script to partition each table need to be scheduled to execute, this strategy can only succeed with the help from admin team. It also requires planning and collaboration between admin and other project teams to come up with list of tables to partition and data range.

## DOP
I don't know how many CPU cores available for our instance because I don't have the permissions to run this query
```sql
select cpu_count from sys.dm_os_sys_info
```
But I can see we're setting `MAXDOP` to 4, I'm guessing this is to prevent an expensive query from eating up the entire CPU. Considering we're allocating 54GB of RAM in UAT environment, I'm guessing our server's CPU is no less than 16 cores, meaning 4 is probably not an optimal number (I could be wrong since I'm not very experienced with DB administration). 
I propose we should consider to follow the practices of setting MAXDOP in this official article
https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/configure-the-max-degree-of-parallelism-server-configuration-option?view=sql-server-ver16
