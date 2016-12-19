---
layout: post
title:  "如何恢复 SET UNUSED 的 COLUMN"
date:   2016-12-19 +0800
categories: Oracle
tags: Oracle Unused
author: TanQuan
---

* content
{:toc}


通过[上篇文章](http://tanquan.me/2016/12/15/marking-columns-unused/)，我们已经基本了解 `标记列为未使用 (Marking Columns Unused)` 了。

我们知道，在 `SET UNUSED` 后 其实数据并未真的被删除，若这时又想恢复该列，有办法吗？

接下来，我们来实验一下。






### 如何修复被标记为 `UNUSED` 的字段

#### 0. 实验环境 Oracle 11.2.0.4

```sql
16:49:06 sys@TQ(tq-78)> select * from v$version;

BANNER
--------------------------------------------------------------------------------
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
PL/SQL Release 11.2.0.4.0 - Production
CORE    11.2.0.4.0      Production
TNS for Linux: Version 11.2.0.4.0 - Production
NLSRTL Version 11.2.0.4.0 - Production

Elapsed: 00:00:00.00
```

#### 1. 创建实验表 T20 ，插入测试数据，并将字段C 标记列为未使用

```sql
16:51:18 tq@TQ(tq-78)> -- 创建实验表 T20
16:51:18 tq@TQ(tq-78)> create table t20 (a number, b number, c varchar2(10), d number);

Table created.

Elapsed: 00:00:00.06
16:51:23 tq@TQ(tq-78)> -- 插入测试数据
16:51:24 tq@TQ(tq-78)> insert into t20 values (1, 2, '3', 4);

1 row created.

Elapsed: 00:00:00.02
16:53:12 tq@TQ(tq-78)> insert into t20 values (2, 3, 'a', 5);

1 row created.

Elapsed: 00:00:00.00
16:53:28 tq@TQ(tq-78)> insert into t20 values (3, 4, 'b', 6);

1 row created.

Elapsed: 00:00:00.01
16:53:39 tq@TQ(tq-78)> commit;

Commit complete.

Elapsed: 00:00:00.00
16:58:35 tq@TQ(tq-78)> -- 将t20表中的C字段标记列为未使用
16:58:39 tq@TQ(tq-78)> ALTER TABLE t20 SET UNUSED COLUMN C;

Table altered.

Elapsed: 00:00:00.22
16:58:53 tq@TQ(tq-78)> desc t20;
 Name                                      Null?    Type
 ----------------------------------------- -------- ----------------------------
 A                                                  NUMBER
 B                                                  NUMBER
 D                                                  NUMBER
```

#### 2. 以下进行恢复 以管理员的身份登陆

##### 2.1 查看表 t20 在数据库中分配的编号 object_id 为 152097

```sql
17:04:03 sys@TQ(tq-78)> -- 查看表 t20 在数据库中分配的编号 object_id 为 152097
17:04:03 sys@TQ(tq-78)> select object_id
17:04:03   2    from dba_objects
17:04:03   3   where object_name = 'T20'
17:04:03   4     and owner = 'TQ';

 OBJECT_ID
----------
    152097

Elapsed: 00:00:00.01
17:07:32 sys@TQ(tq-78)> col name for a30;
17:07:52 sys@TQ(tq-78)> --
17:08:02 sys@TQ(tq-78)> SELECT col#, segcol#, name, intcol#, type#
17:08:02   2    FROM sys.col$
17:08:02   3   WHERE obj# = 152097;

      COL#    SEGCOL# NAME                              INTCOL#      TYPE#
---------- ---------- ------------------------------ ---------- ----------
         1          1 A                                       1          2
         2          2 B                                       2          2
         0          3 SYS_C00003_16121916:58:53$              3          1
         3          4 D                                       4          2

Elapsed: 00:00:00.00
17:11:05 sys@TQ(tq-78)> -- 字段数变为3了
17:11:06 sys@TQ(tq-78)> SELECT COLS FROM sys.TAB$ WHERE OBJ# = 152097;

      COLS
----------
         3

Elapsed: 00:00:00.01
```

##### 2.2 更新相应的数据字典 sys.col$ 和 sys.tab$ 的值，并刷新 SHARED_POOL

```sql
17:20:51 sys@TQ(tq-78)> --
17:20:59 sys@TQ(tq-78)> UPDATE sys.COL$ SET COL# = INTCOL# WHERE OBJ# = 152097;

4 rows updated.

Elapsed: 00:00:00.01
17:21:01 sys@TQ(tq-78)> --
17:21:17 sys@TQ(tq-78)> UPDATE sys.TAB$ SET COLS = COLS + 1 WHERE OBJ# = 152097;

1 row updated.

Elapsed: 00:00:00.00
17:21:19 sys@TQ(tq-78)> --
17:21:29 sys@TQ(tq-78)> UPDATE sys.COL$
17:21:29   2     SET NAME = 'C'
17:21:29   3   WHERE OBJ# = 152097
17:21:29   4     AND COL# = 3;

1 row updated.

Elapsed: 00:00:00.00
17:21:30 sys@TQ(tq-78)> --
17:21:40 sys@TQ(tq-78)> UPDATE sys.COL$ SET PROPERTY = 0 WHERE OBJ# = 152097;

4 rows updated.

Elapsed: 00:00:00.00
17:21:41 sys@TQ(tq-78)> commit;

Commit complete.

Elapsed: 00:00:00.01

17:24:33 sys@TQ(tq-78)> -- 刷新 SHARED_POOL
17:26:26 sys@TQ(tq-78)> alter system flush SHARED_POOL;

System altered.

Elapsed: 00:00:00.11
```

##### 2.3 验证 t20 表的字段C 已经恢复完成

```sql
17:28:40 tq@TQ(tq-78)> desc t20;
 Name                                      Null?    Type
 ----------------------------------------- -------- ----------------------------
 A                                                  NUMBER
 B                                                  NUMBER
 C                                                  VARCHAR2(10)
 D                                                  NUMBER

17:29:07 tq@TQ(tq-78)> select * from t20;

          A          B C                   D
 ---------- ---------- ---------- ----------
          1          2 3                   4
          2          3 a                   5
          3          4 b                   6

 Elapsed: 00:00:00.00
```

-- The End --
