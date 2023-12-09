# How to read AWR

- First when we read an AWR report is to check the database version, OS, hardware and RAC configuration (yes/no)
- Then we need to see the snapshot time (begin and end snap)
- The DB Time is very important because it shows the load of datbase during the snap time.
- For example:
| Begin Snap | 02-Jul-19 15:30-27 |
| End Snap   | 02-Jul-19 16:00-41 |
| Elapsed    | 30.23(mins)        |
| DB Time    | 2,426.25(mins)     |
- The DB Time indicates high load because if we divide 2426 by 30 ~ 80 times.
- In Oracle DB Time = CPU time + IO time + Non-idle wait time.
- If DB Time =< Elapsed Time then the database has low load
- Sudden DB Time surges also mean we have more users or more transactions, it could be because of hardware slow down or client apps slow down the SQL

## Load Profile
- When looking at Load Profile, the system automatically divide DB Time/Elapsed time for us, we need to see how much of the quotent is the CPU Time.
- Ideally, most of the DB Time/Elapsed should be CPU Time, if the CPU Time is only less than 50% of the DB Time then we have problems.
- In this case the CPU Time is only 5% of the DB Time 
| DB Time(s)  | 80.3 |
| CPU Time(s) | 4.3  |
- We also need to see Logical Reads (from memory) and Physical Reads (from disks), ideally the system should read mostly from RAM.
- The Parse number should be low because this show how the Shared Pool is working and if the system has to hard parse SQL queries many times.

## Instance Efficiency Percentages
- This shows how the memory is being used, ideally every number should be 100%
- 100% is not certainly good but if the number is below like 95%, 96% then it's certainly bad.

## Top 5 Timed Foreground Events
- This is to show the waits of the system.
- In our case, it shows that the DB CPU is 5.38% (as calculated before) so most of the wait is from somewhere else.
| Event                   | Wait    | Times  | Avg wait(ms) | %DB Time | Wait class |
| ----------------------- | ------- | ------ | ------------ | -------- | ---------- |
| direct path read        | 410,410 | 88,576 | 216          | 60.85    | User I/O   |
| log file sync           | 13,249  | 13,035 | 984          | 8.95     | Commit     |
| DB CPU                  |         |        |              | 5.38     |            |
| db file sequential read | 15,527  | 2,199  | 142          | 1.51     | User I/0   |
| read by other sessions  | 1,882   | 644    | 342          | 0.44     | User I/O   |

- `direct path read` is when the database bypasses buffer cache and reads directly from the disk, which leads to high physical reads
- After identifying the problem, we can skip to the `IO stats` section to find that the `Direct Reads` is 820GB while the `Buffer Cache Reads` is only 1.7GB

## IO Stats by Function Summary
- This shows the data amount read from disk (physical) and from Buffer Cache (RAM)
- Ideally Buffer Cache Reads should be greater than Direct Reads

## File IO Stats
- After checking the data amount read of disk, we need to check the response time of each file from disk
- This section shows us the file name and their average response time.
- If the response time `Av Rd(ms)` is greater than 20ms then it only worsens the problem 