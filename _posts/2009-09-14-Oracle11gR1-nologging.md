---
layout: post
title:  "nologging在Oracle11gR1中的实现"
date:   2009-09-14 +0800
categories: Oracle
tags: Oracle Oracle数据库管理
author: dbtan
---

* content
{:toc}

> ```sql
> -- 注：我的实验环境是RHEL5.3(64bit) + Oracle11gR1(11.1.0.6)(64bit)
> [root@test7: ~]#uname -a
> Linux test7 2.6.18-128.el5 #1 SMP Wed Dec 17 11:41:38 EST 2008 x86_64 x86_64 x86_64 GNU/Linux
>
> sys@CCDB> select * from v$version;
>
> BANNER
> ----------------------------------------------------------------------------
> Oracle Database 11g Enterprise Edition Release 11.1.0.6.0 - 64bit Production
> PL/SQL Release 11.1.0.6.0 - Production
> CORE    11.1.0.6.0      Production
> TNS for Linux: Version 11.1.0.6.0 - Production
> NLSRTL Version 11.1.0.6.0 - Production
> ```





正常的DML总要产生日志，但当我们要大量的加载数据的时候我们希望尽快的完成任务，我们可以使用
nologging的选项，该选项可以减少日志的产生，只产生少量的日志。当真的是这样吗？**不是的**，请看实验：

```sql
sys@CCDB> select FORCE_LOGGING from v$database;

FORCE_
------
YES

-- 我们先在数据库的运行模式为强制产生日志，实验一下。
```

```sql
sys@CCDB> conn tq/tq
已连接。
tq@CCDB> drop table t1 purge;
表已删除。
tq@CCDB> drop table t2 purge;
表已删除。
tq@CCDB> create table t1 as select * from scott.emp;
表已创建。
tq@CCDB> insert into t1 select * from t1;
已创建14行。
tq@CCDB> /
已创建28行。
--反复重复插入，直到2000行。构造一个“大表”。
tq@CCDB> commit;
提交完成。


建立空表t2：
tq@CCDB> create table t2 as select * from t1 where 0=1;
表已创建。


tq@CCDB> set autotrace traceonly statistics
tq@CCDB> insert into t2 select * from t1;   --正常的SQL语句，没有禁止日志的产生
已创建3584行。
统计信息
---------------------------------------------------
        301  recursive calls
        320  db block gets
        159  consistent gets
          0  physical reads
     190788  redo size			--正常产生日志
        758  bytes sent via SQL*Net to client
        846  bytes received via SQL*Net from client
          4  SQL*Net roundtrips to/from client
          2  sorts (memory)
          0  sorts (disk)
       3584  rows processed


tq@CCDB> rollback;
tq@CCDB> insert /*+ append */ into t2 select * from t1;  --直接路径插入
已创建3584行。
统计信息
-------------------------------------------------------
        304  recursive calls
        104  db block gets
        113  consistent gets
          0  physical reads
       8232  redo size    --有效果


tq@CCDB> rollback;
回退已完成。
tq@CCDB> alter table t2 nologging;    --将T2表改为nologging模式
表已更改。
tq@CCDB> insert into t2 select * from t1;
已创建3584行。
统计信息
---------------------------------------------------
        175  recursive calls
        163  db block gets
        126  consistent gets
          0  physical reads
     178876  redo size		--没有效果


tq@CCDB> rollback;
回退已完成。
tq@CCDB> insert /*+ append */ into t2 select * from t1;    --表t2为nologging模式，再直接路径插入
已创建3584行。
统计信息
-------------------------------------------------------
          4  recursive calls
         34  db block gets
         63  consistent gets
          0  physical reads
        488  redo size     --有效果，redo的减少最显著！
```

接下来，我们改数据库的运行模式为非强制产生日志，实验一下，看看有什么不同？

```sql
sys@CCDB> alter database no FORCE LOGGING;
数据库已更改。
sys@CCDB> select FORCE_LOGGING from v$database;

FORCE_
------
NO
--数据库的运行模式为非强制产生日志

--重新构造一下实验环境：
sys@CCDB> conn tq/tq
已连接。
tq@CCDB> drop table t1 purge;
表已删除。
tq@CCDB> drop table t2 purge;
表已删除。
tq@CCDB> create table t1 as select * from scott.emp;
表已创建。
tq@CCDB> insert into t1 select * from t1;
已创建14行。
tq@CCDB> /
已创建28行。
--反复重复插入，直到2000行。构造一个“大表”。
tq@CCDB> commit;
提交完成。
--建立空表t2：
tq@CCDB> create table t2 as select * from t1 where 0=1;
表已创建。


sys@CCDB> conn tq/tq
已连接。
tq@CCDB> set autotrace traceonly statistics
tq@CCDB> insert into t2 select * from t1;    --正常的SQL语句，没有禁止日志的产生
已创建3584行。
统计信息
---------------------------------------------------
         88  recursive calls
        260  db block gets
         98  consistent gets
          0  physical reads
     186864  redo size		--没有效果


tq@CCDB> rollback;
tq@CCDB> insert /*+ append */ into t2 select * from t1;  --直接路径插入
已创建3584行。
统计信息
-------------------------------------------------------
        139  recursive calls
        107  db block gets
         96  consistent gets
          0  physical reads
       8652  redo size    --有效果


tq@CCDB> rollback;
回退已完成。
tq@CCDB> alter table t2 nologging;    --将T2表改为nologging模式
表已更改。
tq@CCDB> insert into t2 select * from t1;    
已创建3584行。
统计信息
---------------------------------------------------
          0  recursive calls
        163  db block gets
         74  consistent gets
          0  physical reads
     179024  redo size		--没有效果


tq@CCDB> rollback;
回退已完成。
tq@CCDB> insert /*+ append */ into t2 select * from t1;    --表t2为nologging模式，再直接路径插入
已创建3584行。
统计信息
-------------------------------------------------------
          4  recursive calls
         34  db block gets
         63  consistent gets
          0  physical reads
        488  redo size    --有效果，redo的减少最显著！
```

#### 总结：

- 单独使用nologging参数，对redo size没有多少影响，只有和（`/*+ append */`）直接路径插入配合时，才能产生效果。

- 单独使用append提示，对redo的产生影响很大，（`/*+ append */`）直接路径插入是绕过freelists，直接去寻找新块，能减少对freelists的争用。

- 本实验是在非归档模式下进行的，数据库的运行模式是否为强制产生日志，对redo size产生的多少“几乎没有影响”。

- 如果在归档模式下，对于常规表的insert append产生和insert同样的redo。此时的insert append实际上并不会有性能提高。

#### 关于NOLOGGING操作，需要注意以下几点：

- 事实上，还是会生成一定数量的redo。这些redo的作用是保护数据字典。这是不可避免的。与以前（不使用NOLOGGING）相比，尽管生成的redo量要少多了，但是确实会有一些redo。

- NOLOGGING不能避免所有后续操作生成redo。所有后续的“正常”操作（如INSERT、UPDATE和DELETE）还是会生成日志。其他特殊的操作（如使用SQL*Loader的直接路径加载，或使用`INSERT /*+ APPEND */`语法的直接路径插入）不生成日志（除非你ALTER这个表，再次启用完全的日志模式）。不过，一般来说，应用对这个表执行的操作都会生成日志。

- 在一个ARCHIVELOG模式的数据库上执行NOLOGGING操作后，必须尽快为受影响的数据文件建立一个新的基准备份，从而避免由于介质失败而丢失对这些对象的后续修改。实际上，我们并不会丢失后来做出的修改，因为这些修改确实在重做日志中；我们真正丢失的只是要应用这些修改的数据（即最初的数据）。

-- The End --
