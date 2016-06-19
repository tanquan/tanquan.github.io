---
layout: post
title:  "使用sqlloader导入数据"
date:   2009-09-26 +0800
categories: Oracle
tags: Oracle Oracle数据库管理
author: dbtan
---

* content
{:toc}

今天为开发数据库导入数据，使用sqlloader。记录一下遇到的问题及笔记。

### 1、表结构如下：

```sql
oss@WINKSDB> desc member_actions;
 Name              Null?    Type
 ----------------- -------- ------------
 PHONE             NOT NULL VARCHAR2(50)
 ACTION_TIME       NOT NULL DATE
 URL                        VARCHAR2(80)
 APP                        VARCHAR2(20)
 ACTIVE_LEVEL               VARCHAR2(10)
```





### 2、要导入的文件（infile），是由log分析得来：

```sql
[oracle@devdb: ~]$ll -th deactive_actionlist.2009-09-23
-rw-r--r-- 1 oracle oinstall 49M Sep 24 03:11 deactive_actionlist.2009-09-23
[oracle@devdb: ~]$head !$
head deactive_actionlist.2009-09-23
2009-09-23 00:00:04,314 135****8333 user/registeruser comm deactive
2009-09-23 00:00:17,053 135****3585 user/registeruser comm deactive
2009-09-23 00:02:05,704 138****3712 user/registeruser comm deactive
2009-09-23 00:02:41,097 136****1107 user/registeruser comm deactive
2009-09-23 00:02:43,330 135****6592 user/registeruser comm deactive
2009-09-23 00:04:18,818 187****7004 user/registeruser comm deactive
2009-09-23 00:05:52,878 151****6277 user/registeruser comm deactive
2009-09-23 00:09:14,438 134****9551 user/registeruser comm deactive
2009-09-23 00:09:37,988 159****9392 user/registeruser comm deactive
2009-09-23 00:09:46,555 158****8127 user/registeruser comm deactive
```

### 3、sqlloader的controlfile如下：

```sql
load data		                            --控制文件标识
 infile 'deactive_actionlist.2009-09-23'	    --要导入数据的文件名
 truncate into table MEMBER_ACTIONS	            --truncate table MEMBER_ACTIONS_IMP后，再导入数据
 fields terminated by " " optionally enclosed by '"'	  --字符终止于“空格”，并附上"双引号
(ACTION_TIME POSITION(1:19) date "YYYY-MM-DD HH24:MI:SS",     --infile中的日期／时间精确到千分秒，而ACTION_TIME列 为DATE型
field2 FILLER,						  --所以多了一段3位的千分秒，要把它漏过去
phone,
url,
app,
active_level
)							  --定义列对应顺序
```

> **说明**：
> 
> - INSERT：为缺省方式，在数据装载开始时要求表为空。
>
> - APPEND：在表中追加新记录。
>
> - REPLACE：使用一种传统DELETE语句；因此，如果要加载的表中已经包含许多记录，这个操作可能执行得很慢。
>
> - TRUNCATE：则不同，它使用TRUNCATE SQL命令，通常会更快地执行，因为它不必物理地删除每一行。

### 4、现在来导入并看一下产生的log：

```sql
[oracle@devdb: ~]$sqlldr oss/oss control=deactive.ctl direct=true ---导入性能果然很快！只用了5.75秒。
[oracle@devdb: ~]$cat deactive.log
SQL*Loader: Release 11.1.0.7.0 - Production on Fri Sep 25 11:02:59 2009
Copyright (c) 1982, 2007, Oracle. All rights reserved.
Control File: deactive.ctl
Data File: deactive_actionlist.2009-09-23
Bad File: deactive_actionlist.bad
Discard File: none specified
(Allow all discards)
Number to load: ALL
Number to skip: 0
Errors allowed: 50
Continuation: none specified
Path used: Direct
Table MEMBER_ACTIONS, loaded from every logical record.
Insert option in effect for this table: TRUNCATE
Column Name Position Len Term Encl Datatype
------------------------------ ---------- ----- ---- ---- ---------------------
ACTION_TIME 1:19 19 WHT O(") DATE YYYY-MM-DD HH24:MI:SS
FIELD2 NEXT * WHT O(") CHARACTER
(FILLER FIELD)
PHONE NEXT * WHT O(") CHARACTER
URL NEXT * WHT O(") CHARACTER
APP NEXT * WHT O(") CHARACTER
ACTIVE_LEVEL NEXT * WHT O(") CHARACTER
Table MEMBER_ACTIONS:
821263 Rows successfully loaded.
0 Rows not loaded due to data errors.
0 Rows not loaded because all WHEN clauses were failed.
0 Rows not loaded because all fields were null.
Date conversion cache disabled due to overflow (default size: 1000)
Bind array size not used in direct path.
Column array rows : 5000
Stream buffer bytes: 256000
Read buffer bytes: 1048576
Total logical records skipped: 0
Total logical records read: 821263
Total logical records rejected: 0
Total logical records discarded: 0
Total stream buffers loaded by SQL*Loader main thread: 195
Total stream buffers loaded by SQL*Loader load thread: 0
Run began on Fri Sep 25 11:02:59 2009
Run ended on Fri Sep 25 11:03:05 2009
Elapsed time was: 00:00:05.75
CPU time was: 00:00:03.26
```

### 5、最后，看看导入的数据有没有问题：

```sql
oss@WINKSDB> select count(*) from member_actions;

  COUNT(*)
----------
    821263

oss@WINKSDB> alter session set nls_date_format='yyyy-dd-mm hh24:mi:ss';

Session altered.

oss@WINKSDB> col phone for a15
oss@WINKSDB> col url for a30
oss@WINKSDB> col app for a15
oss@WINKSDB> select * from member_actions where rownum <= 10;

PHONE           ACTION_TIME         URL                            APP             ACTIVE_LEVEL
--------------- ------------------- ------------------------------ --------------- ---------------
158****0739     2009-23-09 23:21:16 specialwinks                   comm            deactive
158****8114     2009-23-09 23:21:16 winks/show                     comm            deactive
158****6605     2009-23-09 23:21:16 winks/show                     comm            deactive
134****5440     2009-23-09 23:21:16 message                        comm            deactive
132****9996     2009-23-09 23:21:16 winks/show                     comm            deactive
151****9347     2009-23-09 23:21:16 specialwinks                   comm            deactive
139****8854     2009-23-09 23:21:16 specialwinks                   comm            deactive
158****8584     2009-23-09 23:21:16 winks/show                     comm            deactive
138****6827     2009-23-09 23:21:16 specialwinks                   comm            deactive
151****1746     2009-23-09 23:21:16 config                         comm            deactive

10 rows selected.
```

导入的数据没有问题！

> 该服务器的CPU为1颗4核的Intel(R) Xeon(R) CPU E5410 @ 2.33GHz     内存：4G

-- The End --
