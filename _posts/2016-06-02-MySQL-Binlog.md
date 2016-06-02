---
layout: post
title:  "MySQL二进制日志（binlog）"
date:   2016-06-02 12:05:18 +0800
categories: MySQL
tags: MySQL InnoDB Binlog
---

* content
{:toc}


二进制日志（binary log）记录了对MySQL数据库执行更改的所有操作，但是==不包括`SELECT`和`SHOW`这类操作==，因为这类操作对数据本身并没有修改。然而，若操作本身并没有导致数据发生变化，那么该操作可能也会写入二进制日志。例如：






```sql
tqdb@localhost.[tqdb] 16:00:48> update t set b = 'dbtan' where a = 6;
Query OK, 0 rows affected (0.00 sec)
Rows matched: 0  Changed: 0  Warnings: 0

tqdb@localhost.[tqdb] 16:01:06> show master status \G
*************************** 1. row ***************************
             File: mysql-bin.000129
         Position: 530
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)

tqdb@localhost.[tqdb] 16:01:22> show binlog events in 'mysql-bin.000129' \G
*************************** 1. row ***************************
   Log_name: mysql-bin.000129
        Pos: 4
 Event_type: Format_desc
  Server_id: 1
End_log_pos: 120
       Info: Server ver: 5.6.25-enterprise-commercial-advanced-log, Binlog ver: 4
*************************** 2. row ***************************
   Log_name: mysql-bin.000129
        Pos: 120
 Event_type: Query
  Server_id: 1
End_log_pos: 192
       Info: BEGIN
*************************** 3. row ***************************
   Log_name: mysql-bin.000129
        Pos: 192
 Event_type: Table_map
  Server_id: 1
End_log_pos: 239
       Info: table_id: 70 (tqdb.t)
*************************** 4. row ***************************
   Log_name: mysql-bin.000129
        Pos: 239
 Event_type: Update_rows
  Server_id: 1
End_log_pos: 294
       Info: table_id: 70 flags: STMT_END_F
*************************** 5. row ***************************
   Log_name: mysql-bin.000129
        Pos: 294
 Event_type: Xid
  Server_id: 1
End_log_pos: 325
       Info: COMMIT /* xid=6 */
*************************** 6. row ***************************
   Log_name: mysql-bin.000129
        Pos: 325
 Event_type: Query
  Server_id: 1
End_log_pos: 397
       Info: BEGIN
*************************** 7. row ***************************
   Log_name: mysql-bin.000129
        Pos: 397
 Event_type: Table_map
  Server_id: 1
End_log_pos: 444
       Info: table_id: 70 (tqdb.t)
*************************** 8. row ***************************
   Log_name: mysql-bin.000129
        Pos: 444
 Event_type: Update_rows
  Server_id: 1
End_log_pos: 499
       Info: table_id: 70 flags: STMT_END_F
*************************** 9. row ***************************
   Log_name: mysql-bin.000129
        Pos: 499
 Event_type: Xid
  Server_id: 1
End_log_pos: 530
       Info: COMMIT /* xid=8 */
9 rows in set (0.00 sec)
```

从上述例子中可以看到，MySQL数据库首先进行`UPDATE`操作，从返回的结果看到`Changed: 0`，这意味着该操作并没有导致数据的变化。但是通过`SHOW BINLOG EVENTS`可以看出二进制日志中的确进行了记录。

如果用户想记录SELECT和SHOW操作，那只能使用查询日志，而不是二进制日志。此外，二进制日志还包括了执行数据库更改操作的时间等其他额外信息。总的来说，二进制日志主要有以下几种作用：

- **恢复（recovery）**：某些数据的恢复需要二进制日志。例如，在一个数据库全备文件恢复后，用户可以通过二进制日志进行`point-in-time`的恢复。
- **复制（replication）**：其原理与恢复类似，通过复制和执行二进制日志使一台远程的MySQL数据库（一般称为slave或者standby）与一台MySQL数据库（一般称为master或者primary）进行实时同步。
- **审计（audit）**：用户可以通过二进制日志中的信息来进行审计，判断是否有对数据库进行注入攻击。

通过配置参数`log-bin [=name]`可以启动二进制日志。如果不指定name，则默认二进制日志文件名为主机名，后缀名为二进制日志的序列号，所在目录为数据库所在目录（`datadir`），如：

```sql
tqdb@localhost.[tqdb] 16:24:33> show variables like 'datadir';
+---------------+--------------+
| Variable_name | Value        |
+---------------+--------------+
| datadir       | /MySQL_DATA/ |
+---------------+--------------+
1 row in set (0.00 sec)

tqdb@localhost.[tqdb] 16:24:52> system ls -lh /MySQL_DATA
total 5.6G
drwx------ 2 mysql mysql 4.0K Jul  9  2015 abc
-rw-rw---- 1 mysql mysql   56 Jul  8  2015 auto.cnf
-rw-rw---- 1 mysql mysql 1.0G May 12 16:00 ibdata1
-rw-rw---- 1 mysql mysql 256M May 12 16:00 ib_logfile0
-rw-rw---- 1 mysql mysql 256M May 11 19:19 ib_logfile1
-rw-rw---- 1 mysql mysql 256M May 11 19:20 ib_logfile2
drwx------ 2 mysql mysql 4.0K May  3 17:24 mysql
-rw-rw---- 1 mysql mysql 467M Apr 19 08:47 mysql-bin.000122
-rw-rw---- 1 mysql mysql 1.1G Apr 26 18:05 mysql-bin.000123
-rw-rw---- 1 mysql mysql  120 Apr 26 18:34 mysql-bin.000124
-rw-rw---- 1 mysql mysql  863 May  3 18:01 mysql-bin.000125
-rw-rw---- 1 mysql mysql 1.9G May  9 15:04 mysql-bin.000126
-rw-rw---- 1 mysql mysql  737 May 11 16:29 mysql-bin.000127
-rw-rw---- 1 mysql mysql 422M May 12 14:06 mysql-bin.000128
-rw-rw---- 1 mysql mysql  530 May 12 16:00 mysql-bin.000129
-rw-rw---- 1 mysql mysql  152 May 12 14:06 mysql-bin.index
srwxrwxrwx 1 mysql mysql    0 May 12 14:06 mysql.sock
drwx------ 2 mysql mysql 4.0K Nov 20 17:30 new_apply_1113
drwx------ 2 mysql mysql 4.0K Apr 26 18:02 new_apply_online
drwx------ 2 mysql mysql 4.0K Jul  8  2015 performance_schema
drwx------ 2 mysql mysql 4.0K May  5 18:27 sakila
drwx------ 2 mysql mysql 4.0K Apr 15 17:27 scene
drwxr-xr-x 2 mysql mysql 4.0K Jan 20 18:28 test
-rw-r----- 1 mysql mysql  22K May 12 14:06 test-178.err
-rw-rw---- 1 mysql mysql    1 Apr 26 18:15 test-178.log
-rw-rw---- 1 mysql mysql    5 May 12 14:06 test-178.pid
-rw-rw---- 1 mysql mysql 5.9K May 12 14:06 test-178-slow.log
drwx------ 2 mysql mysql 4.0K May 11 19:07 tqdb
drwx------ 2 mysql mysql 4.0K Apr 21 13:59 wxmall
tqdb@localhost.[tqdb] 16:25:47> system cat /etc/my.cnf | grep log-bin
log-bin=mysql-bin
```

这里的`mysql-bin.000122`即为二进制文件，我们在配置文件指定了名字，所以没有用默认的文件名。`mysql-bin.index`为二进制日志的索引文件，用来存储过往产生的二进制日志序号，在通常情况下，不建议手工修改这个文件。

二进制日志文件在默认情况下并没有启动，需要手动指定参数来启动。可能有人会质疑，开启这个选项是否会对数据库整体性能有所影响。不错，开启这个选项的确会影响性能，但是性能的损失十分有限。根据MySQL官方手册中的测试表明，开启二进制日志会使性能下降1％。但考虑到可以使用复制（`replication`）和`point-in-time`的恢复，这些性能损失绝对是可以且应该被接受的。

以下配置文件的参数影响着二进制日志记录的信息会行为：

- [x] max_binlog_size
- [x] binlog_cache_size
- [x] sync_binlog
- [x] binlog-do-db
- [x] binlog-ignore-db
- [x] log-slave-update
- [x] binlog-format

参数`max_binlog_size`指定了单个二进制日志文件的最的值，如果超过该值，则产生新的二进制文件，后缀名`+1`，并记录到`.index文件`。从MySQL 5.0 开始的默认值为1073741824，代表1G（在之前的版本中`max_binlog_size`默认大小为1.1G）。

当使用事务的表存储引擎（如InnoDB存储引擎）时，所有未提交（uncommitted）的二进制日志会被记录到一个缓存中去，等该事务提交（committed）时直接将缓冲中的二进制日志写入二进制日志文件，而该缓冲的大小由`binlog_cache_size`决定，默认大小为`32K`。此外，`binlog_cache_size`是基于会话（`session`）的，也就是说，当一个线程开始一个事务时，MySQL会自动分配一个大小为`binlog_cache_size`的缓存，因此该值的设置需要相当小心，不能设置过大。当一个事务的记录大于设定的`binlog_cache_size`时，MySQL会把缓冲中的日志写入一个临时文件中，因此该值又不能设得太小。通过`SHOW GLOBAL STATUS`命令查看`binlog_cache_use`、`binlog_cache_disk_use`的状态，可以判断当前`binlog_cache_size`的设置是否合适。`binlog_cache_use`记录了使用缓冲写二进制日志的**次数**，`binlog_cache_disk_use`记录了使用临时文件写二进制日志的**次数**。现在来看一个数据的状态：

```sql
tqdb@localhost.[tqdb] 14:45:26> show variables like 'binlog_cache_size%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| binlog_cache_size | 32768 |
+-------------------+-------+
1 row in set (0.01 sec)

tqdb@localhost.[tqdb] 14:45:57> show global status like 'binlog_cache%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Binlog_cache_disk_use | 999   |
| Binlog_cache_use      | 2997  |
+-----------------------+-------+
2 rows in set (0.00 sec)
```

使用缓冲次数为2997，临时文件使用次数为999。是由于我们之前在这个测试库里对一个有1300W＋的大表进行过多次大数据量的操作。例如：update

```sql
tqdb@localhost.[tqdb] 14:57:10> select count(*) from test_table;
+----------+
| count(*) |
+----------+
| 13930919 |
+----------+
1 row in set (6.71 sec)

tqdb@localhost.[tqdb] 14:58:08> update test_table set path = '/dbtan';
Query OK, 13930919 rows affected (4 min 0.47 sec)
Rows matched: 13930919  Changed: 13930919  Warnings: 0

-- 再来看，使用缓冲次数、临时文件使用次数都增长（+1）了。
tqdb@localhost.[tqdb] 14:56:52> show global status like 'binlog_cache%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Binlog_cache_disk_use | 1000  |
| Binlog_cache_use      | 2998  |
+-----------------------+-------+
2 rows in set (0.02 sec)
```

在默认情况下，二进制日志并不是在每次写的时候同步的磁盘（用户可以理解为缓冲写）。因此，当数据库所在的操作系统发生宕机时，可能会有最后一部分数据没有写入二进制文件中，这会给恢复和复制带来问题。参数`sync_binlog=[N]`表示每写缓冲多少次就同步到磁盘。如果将N设为1，即`sync_binlog=1`表示采用同步写磁盘的方式来写二进制日志，这时写操作不使用操作系统的缓冲来写二进制日志。sync_binlog的默认值为0，如果使用Innodb存储引擎进行复制，并且想得到最大的高可用性，建议将该值设为ON。不过该值为ON时，确时会对数据库IO系统带来一定的影响。

但是，即使将sync_binlog设为1，还是会有一种情况导致问题的发生。当使用InnoDB存储引擎时，在一个事务发出COMMIT动作之前，由于sync_binlog为1，因此会将二进制日志立即写入磁盘。如果这时已经写入了二进制日志，但是提交还没有发生，并且此时发生了宕机，那么在MySQL数据库下次启动时，由于COMMIT操作并没有发生，这个事务会被回滚掉。但是二进制日志已经记录了该事务信息，不能被回滚。这个问题可以通过将参数`innodb_support_xa`设置为1来解决，虽然innodb_support_xa与XA事务有关，但它同时也确保了二进制日志和InnoDB存储引擎数据文件的同步。

参数`binlog-do-db`和`binlog-ignore-db`表示需要写入或者忽略写入哪些库的日志。默认为空，表示需要同步所有库的日志到二进制日志。

如果当前数据库是复制中的slave角色，则它不会将master取得并执行的二进制日志写入自己的二进制日志文件中去。如果需要写入，要设置`log-slave-update`。如果需要搭建`master-->slave-->slave`架构的复制，则必须设置该参数。

`binlog_format`参数十分重要，它影响了记录二进制日志的格式。在MySQL 5.1版本之前，没有这个参数。所有二进制文件的格式都是基于SQL语句（`statement`）级别的，因此基于这个格式的二进制日志文件的复制（`Replication`）和`Oracle的逻辑Standby有点相似`。同时，对于复制是有一定要求的。如在主服务器运行`rand`、`uuid`等函数，又或者使用触发器等操作，这些都可能会导致主从服务器上表中数据不一致（not sync）。另一个影响是，会发现InnoDB存储引擎的默认事务隔离级别是`REPEATABLE READ`。这其实也是因为二进制日志文件格式的关系，如果使用`READ COMMITTED`的事务隔离级别（大多数数据库，如Oracle，Microsoft SQL Server数据库的默认事务隔离级别），会出现类似丢失更新的现象，从而出现主从数据库上的数据不一致。

MySQL 5.1开始引入了`binlog_format`参数，该参数可设的值有`STATEMENT`、`ROW`和`MIXED`。

1.  **STATEMENT**格式和之前的MySQL版本一样，二进制日志文件记录的是日志的逻辑SQL语句。
2.  在**ROW**格式下，二进制日志记录的不再是简单的SQL语句了，而是记录表的行更改情况。基于ROW格式的复制`类似于Oracle的物理Standby`（当然，还是有些区别）。同时，对上述提及的STATEMENT格式下复制的问题予以解决。从MySQL 5.1版本开始，如果设置了`binlog_format为ROW`，可以将InnoDB的事务隔离级别设为`READ COMMITTED`，以获得更好的并发行。
3.  在**MIXED**格式下，MySQL默认采用`STATEMENT`格式进行二进制日志文件的记录，但是在一些情况下会使用`ROW`格式，可能的情况有：
      1. 表的存储引擎为`NDB`，这时对表的DML操作都会以ROW格式记录。
      2. 使用了`UUID()`、`USER()`、`CURRENT_USER()`、`FOUND_ROW()`、`ROW_COUNT()`等不确定函数。
      3. 使用了`INSERT DELAY`语句。
      4. 使用了用户定义函数（`UDF`）。
      5. 使用了临时表（`temporary table`）。

`binlog_format`是动态参数，因此可以在数据库运行环境下进行更改，例如我们可以将当前会话的`binlog_format`设为`ROW`，如：

```sql
tqdb@localhost.[tqdb] 17:43:46> set @@session.binlog_format='ROW';
Query OK, 0 rows affected (0.00 sec)

tqdb@localhost.[tqdb] 17:44:10> select @@session.binlog_format;
+-------------------------+
| @@session.binlog_format |
+-------------------------+
| ROW                     |
+-------------------------+
1 row in set (0.00 sec)
```

当然，也可以将全局的`binlog_format`设置为想要的格式，不过通常这个操作会带来问题，运行时要确保更改后不会对复制带来影响。如：

```sql
tqdb@localhost.[tqdb] 17:48:50> set global binlog_format = 'ROW';
Query OK, 0 rows affected (0.00 sec)

tqdb@localhost.[tqdb] 17:49:37> select @@global.binlog_format;
+------------------------+
| @@global.binlog_format |
+------------------------+
| ROW                    |
+------------------------+
1 row in set (0.00 sec)
```

在通常情况下，我们将参数`binlog_format`设置为`ROW`，这可以为数据库的恢复和复制带来更好的可靠性。但是不能忽略的一点是，这会带来二进制文件大小的增加，有些语句下的ROW格式可能需要更大的容量。比如我们有两张一样的表，大小都为100W，分别UPDATE操作，观察二进制日志大小的变化：

```sql
tqdb@localhost.[tqdb] 19:09:02> set @@session.binlog_format = 'STATEMENT';
Query OK, 0 rows affected (0.00 sec)

tqdb@localhost.[tqdb] 19:10:44> select @@session.binlog_format;
+-------------------------+
| @@session.binlog_format |
+-------------------------+
| STATEMENT               |
+-------------------------+
1 row in set (0.00 sec)

tqdb@localhost.[tqdb] 19:10:48> show master status \G
*************************** 1. row ***************************
             File: mysql-bin.000128
         Position: 112702410
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)

tqdb@localhost.[tqdb] 19:10:51> update test1 set path = '/tq';
Query OK, 1000000 rows affected (14.92 sec)
Rows matched: 1000000  Changed: 1000000  Warnings: 0

tqdb@localhost.[tqdb] 19:12:04> show master status \G
*************************** 1. row ***************************
             File: mysql-bin.000128
         Position: 112702623
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)
```

可以看到，在`binlog_format`格式为`STATEMENT`的情况下，执行UPDATE语句后二进制日志大小中增加了`213字节`（112702623 - 112702410）。如果是使用ROW格式，同样对test2表进行操作，可以看到：

```sql
tqdb@localhost.[tqdb] 19:19:19> set @@session.binlog_format = 'ROW';
Query OK, 0 rows affected (0.00 sec)

tqdb@localhost.[tqdb] 19:19:30> select @@session.binlog_format;
+-------------------------+
| @@session.binlog_format |
+-------------------------+
| ROW                     |
+-------------------------+
1 row in set (0.00 sec)

tqdb@localhost.[tqdb] 19:19:35> show master status \G
*************************** 1. row ***************************
             File: mysql-bin.000128
         Position: 332108535
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)

tqdb@localhost.[tqdb] 19:19:40> update test2 set path = '/tq';
Query OK, 1000000 rows affected (18.30 sec)
Rows matched: 1000000  Changed: 1000000  Warnings: 0

tqdb@localhost.[tqdb] 19:20:09> show master status \G
*************************** 1. row ***************************
             File: mysql-bin.000128
         Position: 441811491
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)
```
```shell
[mysql@test-178: /MySQL_DATA/tqdb]$ ll -th test*
-rw-rw---- 1 mysql mysql  92M May 11 19:20 test2.ibd
-rw-rw---- 1 mysql mysql  92M May 11 19:12 test1.ibd
-rw-rw---- 1 mysql mysql 8.8K May 11 19:07 test2.frm
-rw-rw---- 1 mysql mysql 8.8K May 11 19:06 test1.frm
```

这时会惊讶地发现，同样的操作在`ROW`格式下竟然需要`109702956字节`（441811491 - 332108535），二进制日志文件的大小差不多增加了100MB，要知道test2表的大小也不超过92MB。而且执行时间也有所增加（这里我设置了`sync_binlog=1`）。这是因为这时MySQL数据库不再将逻辑的SQL操作记录到二进制日志中，而是记录对于每行的更改。

上面的这个例子告诉我们，将参数`binlog_format`设置为`ROW`，会对磁盘空间要求有一定的增加。而由于复制是传输二进制日志方式实现的，因此复制的网络开销也有所增加。

二进制日志文件的文件格式为二进制（好像有点废话），不能像错误日志文件、慢查询日志文件那样用cat、head、tail等命令来查看。要查看二进制日志文件的内容，必须通过MySQL提供的工具`mysqlbinlog`。对于`STATEMENT`格式的二进制文件，在使用`mysqlbinlog`后，看到的就是执行的逻辑SQL语加，如：

```shell
[mysql@test-178: /MySQL_DATA]$ mysqlbinlog --no-defaults --start-position=112702410 mysql-bin.000128 | less
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!40019 SET @@session.max_insert_delayed_threads=0*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#160511 16:29:49 server id 1  end_log_pos 120 CRC32 0x5bf6b7d8  Start: binlog v 4, server v 5.6.25-enterprise-commercial-advanced-log created 160511 16:29:49 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;
BINLOG '
fe0yVw8BAAAAdAAAAHgAAAABAAQANS42LjI1LWVudGVycHJpc2UtY29tbWVyY2lhbC1hZHZhbmNl
ZC1sb2cAAAAAAAAAAAB97TJXEzgNAAgAEgAEBAQEEgAAXAAEGggAAAAICAgCAAAACgoKGRkAAdi3
9ls=
'/*!*/;
# at 112702410
#160511 19:11:49 server id 1  end_log_pos 112702489 CRC32 0x1f67544f    Query   thread_id=1     exec_time=15    error_code=0
SET TIMESTAMP=1462965109/*!*/;
SET @@session.pseudo_thread_id=1/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1075838976/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8mb4 *//*!*/;
SET @@session.character_set_client=45,@@session.collation_connection=45,@@session.collation_server=45/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 112702489
#160511 19:11:49 server id 1  end_log_pos 112702592 CRC32 0x90bb1964    Query   thread_id=1     exec_time=15    error_code=0
use `tqdb`/*!*/;
SET TIMESTAMP=1462965109/*!*/;
update test1 set path = '/tq'
/*!*/;
# at 112702592
#160511 19:11:49 server id 1  end_log_pos 112702623 CRC32 0x22fb745f    Xid = 42
COMMIT/*!*/;
# at 112702623
#160511 19:16:41 server id 1  end_log_pos 112702695 CRC32 0x0a1b10d4    Query   thread_id=1     exec_time=0     error_code=0
SET TIMESTAMP=1462965401/*!*/;
```

通过SQL语句`update test1 set path = '/tq'`可以看到，二进制日志的记录采用SQL语句的方式。在这种情况下，`mysqlbinlog`和`Oracle LogMiner`类似。但是如果这时使用ROW格式的记录方式，会发现mysqlbinlog的结果变得“不可读”（`unreadable`），如：

```shell
[mysql@test-178: /MySQL_DATA]$ mysqlbinlog --no-defaults --start-position=332108535 mysql-bin.000128 | less
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!40019 SET @@session.max_insert_delayed_threads=0*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#160511 16:29:49 server id 1  end_log_pos 120 CRC32 0x5bf6b7d8  Start: binlog v 4, server v 5.6.25-enterprise-commercial-advanced-log created 160511 16:29:49 at startup
ROLLBACK/*!*/;
BINLOG '
fe0yVw8BAAAAdAAAAHgAAAAAAAQANS42LjI1LWVudGVycHJpc2UtY29tbWVyY2lhbC1hZHZhbmNl
ZC1sb2cAAAAAAAAAAAB97TJXEzgNAAgAEgAEBAQEEgAAXAAEGggAAAAICAgCAAAACgoKGRkAAdi3
9ls=
'/*!*/;
# at 332108535
#160511 19:19:51 server id 1  end_log_pos 332108607 CRC32 0xcb9dd533    Query   thread_id=1     exec_time=0     error_code=0
SET TIMESTAMP=1462965591/*!*/;
SET @@session.pseudo_thread_id=1/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1075838976/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8mb4 *//*!*/;
SET @@session.character_set_client=45,@@session.collation_connection=45,@@session.collation_server=45/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 332108607
#160511 19:19:51 server id 1  end_log_pos 332108672 CRC32 0x6a6188a5    Table_map: `tqdb`.`test2` mapped to number 73
# at 332108672
#160511 19:19:51 server id 1  end_log_pos 332116833 CRC32 0x92c6353e    Update_rows: table id 73
# at 332116833
#160511 19:19:51 server id 1  end_log_pos 332124909 CRC32 0xee6c81de    Update_rows: table id 73
# at 332124909
#160511 19:19:51 server id 1  end_log_pos 332133112 CRC32 0xc0eef064    Update_rows: table id 73
# at 332133112
#160511 19:19:51 server id 1  end_log_pos 332141289 CRC32 0x5cba6ee7    Update_rows: table id 73
# at 332141289
#160511 19:19:51 server id 1  end_log_pos 332149381 CRC32 0xca1760fa    Update_rows: table id 73
# at 332149381
...skipping...
ADE0MDEyNTUxNzE3NTEuanBngABcJpmS+AAAmZeVZcoAPUIPAJMZAQABMwMvdHERADE0MDEyNTUx
NzE3NTEuanBngABcJpmS+AAAmZeVZcoAPkIPAJMZAQABMwYvZGJ0YW4RADE0MDEyNTUxNzUzNzcu
anBngABcPJmS+AAAmZeVZcoAPkIPAJMZAQABMwMvdHERADE0MDEyNTUxNzUzNzcuanBngABcPJmS
+AAAmZeVZcoAP0IPAJMZAQABMwYvZGJ0YW4RADE0MDEyNTUxNzU5NzYuanBngABZMpmS+AAAmZeV
ZcoAP0IPAJMZAQABMwMvdHERADE0MDEyNTUxNzU5NzYuanBngABZMpmS+AAAmZeVZcoAQEIPAJMZ
AQABMwYvZGJ0YW4RADE0MDEyNTUxNzY1OTYuanBngABZNZmS+AAAmZeVZcoAQEIPAJMZAQABMwMv
dHERADE0MDEyNTUxNzY1OTYuanBngABZNZmS+AAAmZeVZcpkZMax
'/*!*/;
# at 441811460
#160511 19:19:51 server id 1  end_log_pos 441811491 CRC32 0x694b3b81    Xid = 59
COMMIT/*!*/;
DELIMITER ;
# End of log file
ROLLBACK /* added by mysqlbinlog */;
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

这里看不到执行的SQL语句，反而是一大串用户不可读的字符。其实只要加上参数`-v`或者`-vv`就能清楚地看到执行的具体信息了。`-vv`会比`-v`多显示出更新的类型。加上`-vv`选项，可以得到：

```shell
[mysql@test-178: /MySQL_DATA]$ mysqlbinlog --no-defaults -vv --start-position=332108535 mysql-bin.000128 | less
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!40019 SET @@session.max_insert_delayed_threads=0*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#160511 16:29:49 server id 1  end_log_pos 120 CRC32 0x5bf6b7d8  Start: binlog v 4, server v 5.6.25-enterprise-commercial-advanced-log created 160511 16:29:49 at startup
ROLLBACK/*!*/;
BINLOG '
fe0yVw8BAAAAdAAAAHgAAAAAAAQANS42LjI1LWVudGVycHJpc2UtY29tbWVyY2lhbC1hZHZhbmNl
ZC1sb2cAAAAAAAAAAAB97TJXEzgNAAgAEgAEBAQEEgAAXAAEGggAAAAICAgCAAAACgoKGRkAAdi3
9ls=
'/*!*/;
# at 332108535
#160511 19:19:51 server id 1  end_log_pos 332108607 CRC32 0xcb9dd533    Query   thread_id=1     exec_time=0     error_code=0
SET TIMESTAMP=1462965591/*!*/;
SET @@session.pseudo_thread_id=1/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1075838976/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8mb4 *//*!*/;
SET @@session.character_set_client=45,@@session.collation_connection=45,@@session.collation_server=45/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 332108607
#160511 19:19:51 server id 1  end_log_pos 332108672 CRC32 0x6a6188a5    Table_map: `tqdb`.`test2` mapped to number 73
# at 332108672
#160511 19:19:51 server id 1  end_log_pos 332116833 CRC32 0x92c6353e    Update_rows: table id 73
# at 332116833
#160511 19:19:51 server id 1  end_log_pos 332124909 CRC32 0xee6c81de    Update_rows: table id 73
# at 332124909
#160511 19:19:51 server id 1  end_log_pos 332133112 CRC32 0xc0eef064    Update_rows: table id 73
# at 332133112
#160511 19:19:51 server id 1  end_log_pos 332141289 CRC32 0x5cba6ee7    Update_rows: table id 73
# at 332141289
#160511 19:19:51 server id 1  end_log_pos 332149381 CRC32 0xca1760fa    Update_rows: table id 73
# at 332149381
...skipping...
### UPDATE `tqdb`.`test2`
### WHERE
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2=1 /* INT meta=0 nullable=0 is_null=0 */
###   @3='4' /* VARSTRING(8) meta=8 nullable=0 is_null=0 */
###   @4='/dbtan' /* VARSTRING(200) meta=200 nullable=0 is_null=0 */
###   @5='bd843399-8368-4185-805b-20177afb086c.png' /* VARSTRING(400) meta=400 nullable=1 is_null=0 */
###   @6=56.66 /* DECIMAL(7,2) meta=1794 nullable=1 is_null=0 */
###   @7='2012-02-28 00:00:00' /* DATETIME(0) meta=0 nullable=0 is_null=0 */
###   @8='2015-11-10 22:23:10' /* DATETIME(0) meta=0 nullable=0 is_null=0 */
### SET
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2=1 /* INT meta=0 nullable=0 is_null=0 */
###   @3='4' /* VARSTRING(8) meta=8 nullable=0 is_null=0 */
###   @4='/tq' /* VARSTRING(200) meta=200 nullable=0 is_null=0 */
###   @5='bd843399-8368-4185-805b-20177afb086c.png' /* VARSTRING(400) meta=400 nullable=1 is_null=0 */
###   @6=56.66 /* DECIMAL(7,2) meta=1794 nullable=1 is_null=0 */
###   @7='2012-02-28 00:00:00' /* DATETIME(0) meta=0 nullable=0 is_null=0 */
###   @8='2015-11-10 22:23:10' /* DATETIME(0) meta=0 nullable=0 is_null=0 */
### UPDATE `tqdb`.`test2`
### WHERE
###   @1=2 /* INT meta=0 nullable=0 is_null=0 */
###   @2=1 /* INT meta=0 nullable=0 is_null=0 */
###   @3='10' /* VARSTRING(8) meta=8 nullable=0 is_null=0 */
###   @4='/dbtan' /* VARSTRING(200) meta=200 nullable=0 is_null=0 */
###   @5='e72656c4-d1b6-4af2-b61c-b40803a56213.png' /* VARSTRING(400) meta=400 nullable=1 is_null=0 */
###   @6=56.66 /* DECIMAL(7,2) meta=1794 nullable=1 is_null=0 */
###   @7='2012-02-28 00:00:00' /* DATETIME(0) meta=0 nullable=0 is_null=0 */
###   @8='2015-11-10 22:23:10' /* DATETIME(0) meta=0 nullable=0 is_null=0 */
### SET
###   @1=2 /* INT meta=0 nullable=0 is_null=0 */
###   @2=1 /* INT meta=0 nullable=0 is_null=0 */
###   @3='10' /* VARSTRING(8) meta=8 nullable=0 is_null=0 */
###   @4='/tq' /* VARSTRING(200) meta=200 nullable=0 is_null=0 */
###   @5='e72656c4-d1b6-4af2-b61c-b40803a56213.png' /* VARSTRING(400) meta=400 nullable=1 is_null=0 */
###   @6=56.66 /* DECIMAL(7,2) meta=1794 nullable=1 is_null=0 */
###   @7='2012-02-28 00:00:00' /* DATETIME(0) meta=0 nullable=0 is_null=0 */
###   @8='2015-11-10 22:23:10' /* DATETIME(0) meta=0 nullable=0 is_null=0 */
### UPDATE `tqdb`.`test2`
...skipping...
###   @8='2015-11-10 22:23:10' /* DATETIME(0) meta=0 nullable=0 is_null=0 */
### UPDATE `tqdb`.`test2`
### WHERE
###   @1=1000000 /* INT meta=0 nullable=0 is_null=0 */
###   @2=72083 /* INT meta=0 nullable=0 is_null=0 */
###   @3='3' /* VARSTRING(8) meta=8 nullable=0 is_null=0 */
###   @4='/dbtan' /* VARSTRING(200) meta=200 nullable=0 is_null=0 */
###   @5='1401255176596.jpg' /* VARSTRING(400) meta=400 nullable=1 is_null=0 */
###   @6=89.53 /* DECIMAL(7,2) meta=1794 nullable=1 is_null=0 */
###   @7='2014-05-28 00:00:00' /* DATETIME(0) meta=0 nullable=0 is_null=0 */
###   @8='2015-11-10 22:23:10' /* DATETIME(0) meta=0 nullable=0 is_null=0 */
### SET
###   @1=1000000 /* INT meta=0 nullable=0 is_null=0 */
###   @2=72083 /* INT meta=0 nullable=0 is_null=0 */
###   @3='3' /* VARSTRING(8) meta=8 nullable=0 is_null=0 */
###   @4='/tq' /* VARSTRING(200) meta=200 nullable=0 is_null=0 */
###   @5='1401255176596.jpg' /* VARSTRING(400) meta=400 nullable=1 is_null=0 */
###   @6=89.53 /* DECIMAL(7,2) meta=1794 nullable=1 is_null=0 */
###   @7='2014-05-28 00:00:00' /* DATETIME(0) meta=0 nullable=0 is_null=0 */
###   @8='2015-11-10 22:23:10' /* DATETIME(0) meta=0 nullable=0 is_null=0 */
# at 441811460
#160511 19:19:51 server id 1  end_log_pos 441811491 CRC32 0x694b3b81    Xid = 59
COMMIT/*!*/;
DELIMITER ;
# End of log file
ROLLBACK /* added by mysqlbinlog */;
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

现在`mysqlbinlog`向我们解释了它具体做的事情。可以看到，一条简单的`update test2 set path = '/tq'`语句记录了对于整个行更改的信息，这也解释了为什么前面更新了100W行的数据，在ROW格式下，二进制日志文件会增加100MB。

-- The End --
