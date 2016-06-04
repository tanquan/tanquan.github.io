---
layout: post
title:  "MySQL InnoDB锁和死锁"
date:   2016-05-31 17:40:18 +0800
categories: MySQL
tags: MySQL InnoDB Lock
author: TanQuan
---

* content
{:toc}


**Revision    V1.1**

| No.  | Date       | Author/Modifier | Comments |
| ---- | ---------- | --------------- | -------- |
| 1.0  | 2016-04-18 | 谈权              | 初稿       |
| 1.1  | 2016-05-01 | 谈权              | 更新目录结构   |

---------

在使用MySQL的业务中，经常会碰到各种MySQL的死锁。一直以来，我们接触比较多的是Oracle数据库，而大家正在逐步开始使用MySQL数据库，都对MySQL的死锁不甚了解，趁这次机会，好好学习一下MySQL的死锁。我们的死锁的讨论是在InnoDB引擎基础上的。






## 1. MySQL索引

### 1.1 聚簇索引(Clustered Indexes)

InnoDB存储引擎的数据组织方式，是聚簇索引表：完整的记录，存储在主键索引中，通过主键索引，就可以获取记录所有的列。

每个InnoDB的表有一个特殊的索引称之为聚簇索引，每行的数据就是存储在聚簇索引中。通常，聚簇索引和主键同义。

当你在你的表上面定义一个主键时，InnoDB将其作为聚簇索引。建议为你的表都创建一个主键。如果没有唯一并且非空的一列或者多列（用来做你的主键），那么可以创建一个自动填充的自增列（比如ID）

如果你的表没有定义主键，MySQL会将第一个所有列都非空的UNIQUE索引作为聚簇索引。

如果你的表不存在这样的UNIQUE索引（见上），InnoDB内部会自动隐式生成一个包含行ID的列并在其上面建立聚簇索引。这一列按行ID排序。行ID是一个`6-byte`的严格单调自增的字段。因此，按照行在物理上是按照插入顺序排序的。

聚簇索引是如何加速查询的呢？通过聚簇索引访问一行非常快，这是因为在聚簇索引上搜索会直接定位到包含你需要的行的数据所在的页上(page).

参考: [http://dev.MySQL.com/doc/refman/5.6/en/innodb-index-types.html](http://dev.MySQL.com/doc/refman/5.6/en/innodb-index-types.html)

### 1.2 二级索引(Secondary Indexes)

除了聚簇索引其他索引都是二级索引。==在InnoDB中每个二级索引记录都包含了**这一行的主键列**和**当前这个二级索引包含的列**==。InnoDB使用二级索引中包含的主键取索引这一行对应的聚簇索引，进而找到这一行完整的数据。

如果主键很长，则二级索引会占有更多的空间，因此建议使用短的列做主键。

![InnoDB锁和死锁_PrimaryKey_SecondaryKey](http://7xsk51.com2.z0.glb.clouddn.com/InnoDB%E9%94%81%E5%92%8C%E6%AD%BB%E9%94%81_PrimaryKey_SecondaryKey.png)

![InnoDB锁和死锁_PrimaryKey_SecondaryKey_1](http://7xsk51.com2.z0.glb.clouddn.com/InnoDB%E9%94%81%E5%92%8C%E6%AD%BB%E9%94%81_PrimaryKey_SecondaryKey_1.png)

## 2. MySQL锁

Innodb具备表锁和行锁，其中表锁是MySQL提供的，跟存储引擎无关；行锁是Innodb存储引擎实现。

### 2.1 共享锁和排他锁

```
1. 共享锁(S)
允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。
2. 排他锁(X)
允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写锁。
```

另外，为了允许行锁和表锁共存，实现多粒度锁机制，InnoDB还有两种内部使用的意向锁(Intention Locks)，这两种意向锁都是表锁。

```
1. 意向共享锁(IS)
事务打算给数据行加行共享锁，事务在给一个数据行加共享锁前必须先取得该表的IS锁。
2. 意向排他锁(IX)
事务打算给数据行加行排他锁，事务在给一个数据行加排他锁前必须先取得该表的IX锁。
```

上面这四种锁的兼容性Conflict表示冲突不能共存，Compatible表示可以共存:


|          | *X*      | *IX*       | *S*        | *IS*       |
| :------: | -------- | ---------- | ---------- | ---------- |
| ***X***  | Conflict | Conflict   | Conflict   | Conflict   |
| ***IX*** | Conflict | Compatible | Conflict   | Compatible |
| ***S***  | Conflict | Conflict   | Compatible | Compatible |
| ***IS*** | Conflict | Compatible | Compatible | Compatible |

### 2.2 什么时候会加锁

```sql
1. 共享锁(S)
    SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE
2. 排他锁(X)
    SELECT * FROM table_name WHERE ... FOR UPDATE
```

参考: [http://dev.MySQL.com/doc/refman/5.6/en/innodb-lock-modes.html](http://dev.MySQL.com/doc/refman/5.6/en/innodb-lock-modes.html) 其中有「Deadlock Example」

### 2.3 当前请求锁

使用`show engine innodb status`命令查看当前请求锁的信息。

```sql
mysql> show engine innodb status \G;
```

可以从`information_schema.INNODB_LOCKS`表中查看锁的信息。

```sql
mysql> select * from information_schema.INNODB_LOCKS \G;
```


### 2.4 锁的算法(Record Lock，Gap Lock，Next-Key Lock)

InnoDB有三种类型的行锁：`record locks`，`gap locks`和`next-key locks`

索引锁是在单个索引记录上的锁；区间锁是两个索引记录之间的锁，或者第一个索引之前的锁，或者最后一个索引之后的锁；Next-Key锁是索引锁和该索引之前的gap锁的结合。

```
1. 索引锁(Record Lock)
索引锁总是锁定索引(可能多条)，即使表上面没有索引(这种情况，InnoDB会隐式的用自增id创建一个聚簇索引)。一级索引只对一级索引加锁，二级索引对二级索引和对应的一级索引加锁。注意记录锁锁的是索引记录，不是具体的数据记录。

2. 区间锁(Gap Lock)
锁定索引记录间隙的锁，确保索引记录的间隙不变，间隙锁是针对事务隔离等级是可重复读(Repeatable Read)或以上级别而言的。

间隙锁一般是针对非唯一索引而言的

3. Next-Key Lock
默认情况，InnoDB使用REPEATABLE READ事物隔离级别，并且innodb_locks_unsafe_for_binlog这个系统设置无效。这时InnoDB使用next-key锁来做搜索(searches)和索引扫描(index scans)，以此来防止幻读。
(参考:http://dev.MySQL.com/doc/refman/5.6/en/innodb-next-key-locking.html)
```

### 2.5 区间锁

区间锁的一个简单例子：

```sql
tqdb@localhost.[tqdb] 18:45:41> CREATE TABLE t7 (
    ->   id int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
    ->   num int(11) DEFAULT NULL,
    ->   PRIMARY KEY (id),
    ->   KEY n (num)
    -> ) ENGINE=InnoDB ;
Query OK, 0 rows affected (0.01 sec)

tqdb@localhost.[tqdb] 18:45:43> insert into t7(num) values(1), (7), (4), (9);
Query OK, 4 rows affected (0.00 sec)
Records: 4  Duplicates: 0  Warnings: 0
4 rows in set (0.00 sec)

tqdb@localhost.[tqdb] 18:48:07> select * from t7;
+----+------+
| id | num  |
+----+------+
|  1 |    1 |
|  3 |    4 |
|  2 |    7 |
|  4 |    9 |
+----+------+
4 rows in set (0.00 sec)
```

表中现在有4条记录，其中普通索引(二级索引n)生成了5个Gap：

`(-∞, 1), (1, 4), (4, 7), (7, 9), (9, +∞)`

现在Session A以共享锁获取num=4的数据，Session B想要插入数据，就有可能造成锁等待导致超时从而重启事务，因为Session A以共享锁获取num=4的数据，会产生gap锁将区间`(1, 4)`和区间`(4, 7)`锁住，因此这两个区间的插入会失败：

![gap_lock](http://7xsk51.com2.z0.glb.clouddn.com/InnoDB%E9%94%81%E5%92%8C%E6%AD%BB%E9%94%81_gap_lock.png)

间隙锁在InnoDB的作用就是防止其它事务的插入操作，以此来达到防止幻读的发生。另外，在上面的例子中，我们选择的是一个普通（非唯一）索引字段来测试的，这不是随便选的，因为如果InnoDB扫描的是一个主键、或是一个唯一索引的话，那InnoDB只会采用行锁方式来加锁，而不会使用Next-Key Lock的方式，也就是说不会对索引之间的间隙加锁。

要禁止间隙锁的话，可以把隔离级别降为读已提交(Read-Comitted)，或者开启参数`innodb_locks_unsafe_for_binlog=1`。

参考: [http://www.dbtan.com/2015/10/mysql-using-repeatable-read-as-the-default-isolation-level.html](http://www.dbtan.com/2015/10/mysql-using-repeatable-read-as-the-default-isolation-level.html)

[http://dev.MySQL.com/doc/refman/5.6/en/innodb-record-level-locks.html](http://dev.MySQL.com/doc/refman/5.6/en/innodb-record-level-locks.html)

## 3. snapshot read 和 current read

MySQL的两种read方式：

> - 快照读(snapshot read)
>
> 快照读：读取的是记录的可见版本(有可能是历史版本)，不用加锁；
>
> 通常，简单的select操作，属于快照读，不加锁，比如：
> ```sql
> select * from table where ?
> ```
> - 当前读(current read或者lock read)
>
> 当前读：读取的是记录的最新版本，并且，当前读返回的记录，都会加上锁，保证其他事务不会再并发修改这条记录。
>
> 特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁。比如：
> ```sql
> select * from table where ? lock in share mode
> select * from table where ? for update
> insert into table values (...)
> update table set ? where ?
> delete from table where ?
> ```
> 所有以上的语句，都属于当前读，读取记录的最新版本。并且，读取之后，还需要保证其他并发事务不能修改当前记录，对读取记录加锁。其中，除了第一条语句，对读取记录加S锁 (共享锁)外，其他的操作，都加的是X锁(排它锁)。

总之一句话：有加锁的查询都认为是当前读。

快照读大大的提高了数据读取的并发。快照读的一个简单示意图，快照数据就是当前数据之前的版本数据，可能有多个版本的快照数据，每个快照数据中包含了版本信息(如时间戳等)：

![snapshot_read](http://7xsk51.com2.z0.glb.clouddn.com/InnoDB%E9%94%81%E5%92%8C%E6%AD%BB%E9%94%81_snapshot_read.png)

为什么将插入/更新/删除操作，都归为当前读？可以看看下面这个更新操作，在数据库中的执行流程：

![update-lock](http://7xsk51.com2.z0.glb.clouddn.com/InnoDB%E9%94%81%E5%92%8C%E6%AD%BB%E9%94%81_update-lock.png)

从图中，可以看到，一个Update操作的具体流程。当Update SQL被发给MySQL后，MySQL Server会根据where条件，读取第一条满足条件的记录，然后InnoDB引擎会将第一条记录返回，并加锁(current read)。待MySQL Server收到这条加锁的记录之后，会再发起一个Update请求，更新这条记录。一条记录操作完成，再读取下一条记录，直至没有满足条件的记录为止。因此，Update操作内部，就包含了一个当前读。同理，Delete操作也一样。Insert操作会稍微有些不同，简单来说，就是Insert操作可能会触发Unique Key的冲突检查，也会进行一个当前读。

注：根据上图的交互，针对一条当前读的SQL语句，InnoDB与MySQL Server的交互，是一条一条进行的，因此，加锁也是一条一条进行的。先对一条满足条件的记录加锁，返回给MySQL Server，做一些DML操作；然后在读取下一条加锁，直至读取完毕。

### 3.1 不同隔离界别下的 snapshot read

在`Read Committed`级别下，快照读总是读取被锁定行的最新的快照数据。而在`Repeatable Read`和`Serializable`级别，快照读读取的是事物开始时候的行数据版本。

下面是一个简单的例子，一个很简单的表，插入一条数据：

```sql
tqdb@localhost.[tqdb] 14:03:16> CREATE TABLE parent (id int(10) NOT NULL,   PRIMARY KEY (id) ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.04 sec)

tqdb@localhost.[tqdb] 14:03:17> insert into parent (id) values(1);
Query OK, 1 row affected (0.01 sec)
```

我们起两个事务，一个读取(Session A)，一个更新(Session B)，用来验证不同事务隔离级别下快照读的差异：

```
1.Session A中首先开始事物，查询id=1的数据，这时候，无论在Read Committed还是Repeatable Read级别，结果都是1;
2.Session B然后开始事物，并执行update操作，没有commit;
3.这时候Session A再查询id=1的数据，显然Read Committed还是Repeatable Read级别，结果都是1;(在Read Uncommited灰度到未提交的脏数据).
4.Session B提交事物;
5.这时候Session A再查询id=1的数据，就发现差异:Read Committed级别下读取到被修改的数据，而Repeatable Read读取的还是老数据.因为Read Committed只读取最新的快照数据，而Repeatable Read是参考当前事物开始时间来读取快照数据.
```

首先是`Repeatable Read`的结果:

```sql
tqdb@localhost.[tqdb] 14:03:22> SELECT @@session.tx_isolation;
+------------------------+
| @@session.tx_isolation |
+------------------------+
| REPEATABLE-READ        |
+------------------------+
1 row in set (0.00 sec)

tqdb@localhost.[tqdb] 14:04:57> select * from parent where id = 1;
+----+
| id |
+----+
|  1 |
+----+
1 row in set (0.00 sec)
```

![snapshot_read_RR](http://7xsk51.com2.z0.glb.clouddn.com/InnoDB%E9%94%81%E5%92%8C%E6%AD%BB%E9%94%81_snapshot_read_RR.png)

下面是`Read Committed`的结果(Session B一旦提交，Session A未commit的情况下就能读到Session B提交的数据。)：

```sql
tqdb@localhost.[tqdb] 14:08:45> set tx_isolation='read-committed';
Query OK, 0 rows affected (0.00 sec)

tqdb@localhost.[tqdb] 14:08:47> SELECT @@session.tx_isolation;
+------------------------+
| @@session.tx_isolation |
+------------------------+
| READ-COMMITTED         |
+------------------------+
1 row in set (0.00 sec)

tqdb@localhost.[tqdb] 14:08:50> select * from parent where id = 1;
+----+
| id |
+----+
|  1 |
+----+
1 row in set (0.00 sec)
```

![snapshot_read_RC](http://7xsk51.com2.z0.glb.clouddn.com/InnoDB%E9%94%81%E5%92%8C%E6%AD%BB%E9%94%81_snapshot_read_RC.png)

## 4. InnoDB MVCC

InnoDB是一种多版本存储引擎，它必须保存各行老版本信息，这个信息存在一个称之为回滚段(rollback segment)的数据结构中。

在MySQL内部，InnoDB为每行数据额外增加三个字段：

> 1.  一个`6-byte`的名为DB_TRX_ID字段，用来表示最后一个插入(insert)或者更新(update)这行记录的事物的标记。注意，删除(delete)也被当成一种更新，只是标记这一行的一个额外的bit位来表征这个数据被删除。
> 2.  一个`7-byte`的名为DB_ROLL_PTR字段，称之为回滚指针(roll pointer)。这个指针指向写在rollback segment中的undo log记录。如果这一行被更新了，undo log就包含了能够将这一行完全恢复到修改之前的信息。
> 3.  一个`6-byte`的DB_ROW_ID字段，用来存行id(row ID)，行id是插入数据的时候自动严格递增生成的。如果InnoDB自动产生了一个聚簇索引(clustered index)，这个索引就包含行id。否在行id就不会在任何索引中出现。

InnoDB使用存储在rollback segment中的信息(undo log)去实现事物回滚时候的undo操作。另外，InnoDB也是使用这个信息构建快照读(snapshot read)时候的行数据。

rollback segment中的Undo logs分为插入(insert)和更新(update)的undo logs。

经常提交你的事物，包括那些只是consistent reads的事物。否则(长时间不提事物)会导致InnoDB不能及时废弃update undo logs中的数据，进而导致rollback segment中数据太大挤占你的表空间(tablespace)。

rollback segment中undo log记录的物理大小(physical size)通常小于对应的插入或者更新的行数。你可以使用这个信息去计算你的rollback segment需要的空间。

在InnoDB多版本方案中，当你删除一行记录，实际上行不会立即被物理删除。只有当这个删除对应的update undo log被废弃的时候这行记录才会真正被物理删除。此删除操作被称为清除(purge)是通过Purge后台进程实现的，这个过程非常的快，通常其顺序和SQL语句执行删除的顺序一致。Purge进程定期扫描InnoDB的undo，按照先读老undo，再读新undo的顺序，读取每条undo record。

参考： [https://dev.mysql.com/doc/refman/5.6/en/innodb-multi-versioning.html](https://dev.mysql.com/doc/refman/5.6/en/innodb-multi-versioning.html)

## 5. 隔离级别(Isolation Level)

### 5.1 InnoDB的4种隔离级别

MySQL InnoDB定义的4种隔离级别：

```
1. Read Uncommited
2. Read Committed (RC)
3. Repeatable Read (RR)
4. Serializable
```

Read Uncommited安全级别比较低，因此很少使用。Serializable隔离级别读写冲突，因此并发度急剧下降，在MySQL/InnoDB下不建议使用。Repeatable Read是InnoDB默认的事物级别。Oracle和MS SQL的默认级别是Read Committed。

### 5.2 脏读，不可重复读，幻读

在事务并行下出现的几个问题：

```
1. 脏读
可能读取到其他会话中未提交事务修改的数据，在Read Uncommited级别下可能出现。
2. 不可重复读
同一个事务中前后两次读取的内容不一致，在Read Uncommited和Read Committed会出现。
3. 幻读
如果另一个事务同时提交了新数据(本事务查询时候感知不到这个变更)，本事务再更新时，就会惊奇的发现了这些新数据(比如触发违反了uniq key等)，就好像之前读到的数据是鬼影一样的幻觉。这种情况就是上述说的，快照读和当前读一起存在的情况，会出现幻读的场景。必须使用当前读，才能避免幻读。比如：select ...lock in share mode和select ...for update.
```

各个事物界别下可能出现的问题：

| 隔离级别            | 脏读(Dirty Read) | 不可重复读(NonRepeatable Read) | 幻读(Phantom Read) |
| --------------- | -------------- | ------------------------- | ---------------- |
| Read Uncommited | 可能             | 可能                        | 可能               |
| Read Committed  | 不可能            | 可能                        | 可能               |
| Repeatable Read | 不可能            | 不可能                       | 可能               |
| Serializable    | 不可能            | 不可能                       | 不可能              |

幻读的一个示例，Session A在insert之前先select查看数据是否存在，结果告知可以插入，这时候Session B变更数据并提交。Session A再插入会因为主键冲突失败：

```sql
tqdb@localhost.[tqdb] 14:28:10> SELECT @@session.tx_isolation;
+------------------------+
| @@session.tx_isolation |
+------------------------+
| READ-COMMITTED         |
+------------------------+
1 row in set (0.00 sec)

tqdb@localhost.[tqdb] 14:28:13> select * from parent where id = 1;
+----+
| id |
+----+
|  1 |
+----+
1 row in set (0.00 sec)
```

![phantom_read](http://7xsk51.com2.z0.glb.clouddn.com/InnoDB%E9%94%81%E5%92%8C%E6%AD%BB%E9%94%81_phantom_read.png)

那么，InnoDB指出的可以避免幻读是怎么回事呢？

[http://dev.mysql.com/doc/refman/5.6/en/innodb-record-level-locks.html](http://dev.MySQL.com/doc/refman/5.6/en/innodb-record-level-locks.html)

> By default, `InnoDB` operates in [`REPEATABLE READ`](http://dev.mysql.com/doc/refman/5.6/en/set-transaction.html#isolevel_repeatable-read) transaction isolation level and with the [`innodb_locks_unsafe_for_binlog`](http://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_locks_unsafe_for_binlog) system variable disabled. In this case, `InnoDB` uses next-key locks for searches and index scans, which prevents phantom rows (see [Section 14.3.3, “Avoiding the Phantom Problem Using Next-Key Locking”](http://dev.mysql.com/doc/refman/5.6/en/innodb-next-key-locking.html)).


简单翻译就是，当隔离级别是可重复读，且禁用`innodb_locks_unsafe_for_binlog`的情况下，在搜索和扫描index的时候使用的next-key locks可以避免幻读。

关键点在于，是InnoDB默认对一个普通的查询也会加next-key locks，还是说需要应用自己来加锁呢？如果单看这一句，可能会以为InnoDB对普通的查询也加了锁，如果是，那和序列化（SERIALIZABLE）的区别又在哪里呢？

MySQL manual里还有一段：

[http://dev.MySQL.com/doc/refman/5.6/en/innodb-next-key-locking.html](http://dev.MySQL.com/doc/refman/5.6/en/innodb-next-key-locking.html)

> ### Avoiding the Phantom Problem Using Next-Key Locking
>
> To prevent phantoms, `InnoDB` uses an algorithm called next-key locking that combines index-row locking with gap locking.
>
> You can use next-key locking to implement a uniqueness check in your application: If you read your data in share mode and do not see a duplicate for a row you are going to insert, then you can safely insert your row and know that the next-key lock set on the successor of your row during the read prevents anyone meanwhile inserting a duplicate for your row. Thus, the next-key locking enables you to “lock” the nonexistence of something in your table.


根据这一段，我们可以理解为，InnoDB提供了next-key locks，但需要应用程序自己去加锁，才能防止幻读。manual里提供一个例子:

```sql
SELECT * FROM child WHERE id > 100 FOR UPDATE;
```

这样，InnoDB会给id大于100的行(假如child表里有一行id为102)，以及100-102，102+的gap都加上锁。可以使用`SHOW ENGINE INNODB STATUS \G`来查看是否给表加上了锁。

结论就是：MySQL InnoDB的REPEATABLE READ并不保证避免幻读，需要应用使用加锁读来保证。而这个加锁度使用到的机制就是next-key locks。

### 5.3 修改隔离级别

InnoDB默认是可重复读的(REPEATABLE READ)。可以在命令行用`set tx_isolation='read-committed';`选项，或在配置文件(/etc/my.cnf)里`transaction-isolation = READ-COMMITTED`，为所有连接设置默认隔离级别。

在my.cnf文件的[MySQLd]节里类似如下设置该选项：

```sql
transaction-isolation = {READ-UNCOMMITTED | READ-COMMITTED | REPEATABLE-READ | SERIALIZABLE}
```

用户可以用`SET TRANSACTION`语句改变单个会话或者所有新进连接的隔离级别：

```sql
SET [SESSION | GLOBAL] TRANSACTION ISOLATION LEVEL {READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE}
```

## 6. 死锁

> - 死锁发生的条件
>
> 互斥条件：一个资源每次只能被一个进程使用；
> 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放；
> 不剥夺条件：进程已获得的资源，在末使用完之前，不能强行剥夺；
> 循环等待条件：若干进程之间形成一种头尾相接的循环等待资源关系。

系统检测到死锁后，会自动回滚其中事务较小的一个(根据undo日志的大小来决定)。

对于DB而言，导致死锁意味着发生了循环等待，在InnoDB中由于行锁的引入，比较容易发生死锁。

下面总结一些发生死锁的情况（不全）：

1. 同一索引上，两个session相反的顺序加锁多行记录；
2. Primary key和Secondary index，通过primary key找到记录，更新Secondary index字段与通过Secondary index更新记录；
3. UPDATE/DELETE通过不同的二级索引更新多条记录，可能造成在Primary key上不同的加锁顺序。

> 一条DELETE语句与一条UPDATE语句产生了死锁。
>
> 经过分析找到原因：
>
> DELETE语句通过二级索引删除记录，加锁顺序：二级索引（WHERE使用到二级索引）–>主键索引 –> 所有其它二级索引。UPDATE语句的加锁顺序：二级索引（WHERE条件使用二级索引）–>主键索引 –>包含更新字段的其它二级索引。
>
> 由于DELETE操作更新了UPDATE语句WHERE条件使用到的索引，这导致DELETE与UPDATE加锁顺序相反，导致死锁。

#### 实验数据

```sql
tqdb@localhost.[tqdb] 15:15:20> CREATE TABLE t8 (
    ->   id int(11) NOT NULL AUTO_INCREMENT,
    ->   a int(11) DEFAULT NULL,
    ->   b int(11) DEFAULT NULL,
    ->   c int(11) DEFAULT NULL,
    ->   PRIMARY KEY (id),
    ->   KEY idx_a_b (a,b),
    ->   KEY idx_b (b)
    -> ) ENGINE=InnoDB
    -> ;
Query OK, 0 rows affected (0.01 sec)
tqdb@localhost.[tqdb] 15:15:35> --
tqdb@localhost.[tqdb] 15:15:35> insert into t8(a, b, c) values
    -> (2461, 3296, 9096)
    -> ,(5593,  676, 6600)
    -> ,( 972, 5062, 2391)
    -> ,(6773, 6688, 3123)
    -> ,(5550, 8383, 5266)
    -> ,(1181,   93, 6932)
    -> ,(4378, 1097, 2351)
    -> ;
Query OK, 16 rows affected (0.01 sec)
Records: 16  Duplicates: 0  Warnings: 0
```

##### update语句：

```sql
update t set a=a+1 where b=93;
```

步骤：

###### read阶段1：

> **row_search_for_mysql，对找到的二级索引记录加 LOCK_X（LOCK_ORDINARY）锁（index->name=idx_b）**
>

###### read阶段2：

> **row_search_for_mysql，对找到的主键索引记录加 LOCK_X（LOCK_REC_NOT_GAP）锁（index->name=PRIMARY）**
>

###### update阶段1：

> **row_upd_clust_step，更新主键索引记录**
>

###### update阶段2：

> **row_upd_sec_step，更新二级索引记录（node->index->name = idx_a_b）**
>

###### 接update阶段2：

> **二级索引记录加锁LOCK_X(LOCK_REC_NOT_GAP)（index->name = idx_a_b）**
>

###### select阶段结束：

> **锁住最后一条记录的下一条记录的间隙LOCK_X（LOCK_GAP），防止select阶段有数据插入（index->name=idx_b）**
>

### 总结

在InnoDB中，通过二级索引更新记录，首先会在WHERE条件使用到的二级索引上加Next-key类型的X锁，以防止查找记录期间的其它插入/删除记录，然后通过二级索引找到primary key并在primary key上加Record类型的X锁（之所以不是Next-key，是因为查询条件是二级索引，若WHERE条件使用到的是primary key，就会上Next-key类型的X锁），之后更新记录并检查更新字段是否是其它索引中的某列，如果存在这样的索引，通过update的旧值到二级索引中删除相应的entry，此时x锁类型为Record。

-- The End --
