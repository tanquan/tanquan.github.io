---
layout: post
title:  "标记列为未使用 (Marking Columns Unused)"
date:   2016-12-15 19:05:18 +0800
categories: Oracle
tags: Oracle Unused
author: TanQuan
---

* content
{:toc}

### Marking Columns Unused 基本操作步骤

#### 1. 标记列为未使用

```sql
ALTER TABLE <table_name> SET UNUSED (<column_name>);
```

#### 2. 然后在数据库空闲时，再删除列

```sql
ALTER TABLE <table_name> DROP UNUSED COLUMNS CHECKPOINT <n>;
```

> -- `CHECKPOINT <n>`
>
> In the ALTER TABLE statement that follows, the optional clause CHECKPOINT is specified. This clause causes a checkpoint to be applied after processing the specified number of rows, in this case 250. Checkpointing cuts down on the amount of undo logs accumulated during the drop column operation to avoid a potential exhaustion of undo space.
>
> ALTER TABLE hr.admin_emp DROP UNUSED COLUMNS CHECKPOINT 250;

参见官方文档：[http://docs.oracle.com/cd/E11882_01/server.112/e25494/tables.htm#ADMIN11662](http://docs.oracle.com/cd/E11882_01/server.112/e25494/tables.htm#ADMIN11662)

![Marking Columns Unused](https://o8foyu42q.qnssl.com/tq_notes/%E6%A0%87%E8%AE%B0%E5%88%97%E4%B8%BA%E6%9C%AA%E4%BD%BF%E7%94%A8/Marking_Columns_Unused.png)

### 将列设为UNUSED时的系统行为

#### 0. 实验环境 Oracle 11.2.0.4

```sql
17:03:36 sys@TQ(tq-78)> select * from v$version;

BANNER
--------------------------------------------------------------------------------
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
PL/SQL Release 11.2.0.4.0 - Production
CORE    11.2.0.4.0      Production
TNS for Linux: Version 11.2.0.4.0 - Production
NLSRTL Version 11.2.0.4.0 - Production

Elapsed: 00:00:00.00
```

#### 1. 创建测试表 `t18`

```sql
17:04:18 tq@TQ(tq-78)> -- 创建表 t18
17:04:19 tq@TQ(tq-78)> create table t18 (a number, b number, c number, d number);

Table created.

Elapsed: 00:00:00.06
```

#### 2. 查看表 `t18` 在数据库中分配的编号 `object_id` 为 152037

```sql
17:05:02 tq@TQ(tq-78)> --
17:05:03 tq@TQ(tq-78)> SELECT object_id
17:05:03   2    FROM dba_objects
17:05:03   3   WHERE object_name = 'T18'
17:05:03   4     AND owner = 'TQ';

 OBJECT_ID
----------
    152037

Elapsed: 00:00:00.01
```

#### 3. 查看表 `t18` 在 `col$` 数据字典中 `col#`, `segcol#`, `intcol#` 的值

```sql
17:05:48 tq@TQ(tq-78)> --
17:05:49 tq@TQ(tq-78)> col name for a30;
17:05:49 tq@TQ(tq-78)>
17:05:49 tq@TQ(tq-78)> SELECT col#, segcol#, name, intcol#, type#
17:05:49   2    FROM sys.col$
17:05:49   3   WHERE obj# IN (SELECT object_id
17:05:49   4                    FROM dba_objects
17:05:49   5                   WHERE object_name = 'T18'
17:05:49   6                     AND owner = 'TQ');

      COL#    SEGCOL# NAME                              INTCOL#      TYPE#
---------- ---------- ------------------------------ ---------- ----------
         1          1 A                                       1          2
         2          2 B                                       2          2
         3          3 C                                       3          2
         4          4 D                                       4          2

Elapsed: 00:00:00.01
```

```sql
17:06:38 tq@TQ(tq-78)> --
17:06:39 tq@TQ(tq-78)> col column_name for a20;
17:06:39 tq@TQ(tq-78)> col data_type for a20;
17:06:39 tq@TQ(tq-78)>
17:06:39 tq@TQ(tq-78)> select column_name,
17:06:39   2         data_type,
17:06:39   3         column_id,
17:06:39   4         hidden_column,
17:06:39   5         segment_column_id  seg_cid,
17:06:39   6         internal_column_id internal_cid
17:06:39   7    from dba_tab_cols
17:06:39   8   where owner = 'TQ'
17:06:39   9     and table_name = 'T18';

COLUMN_NAME          DATA_TYPE             COLUMN_ID HID    SEG_CID INTERNAL_CID
-------------------- -------------------- ---------- --- ---------- ------------
A                    NUMBER                        1 NO           1            1
B                    NUMBER                        2 NO           2            2
C                    NUMBER                        3 NO           3            3
D                    NUMBER                        4 NO           4            4

Elapsed: 00:00:00.00
```

> **说明**：`col$` 数据字典中 `col#`, `segcol#`, `intcol#` 的意义：
>
> `COL#` 可以表示该列是否在用（ 0 为 UNUSED ），
>
> `SEGCOL#` 表示各列在数据块上存储时的顺序，
>
> `INTCOL#` 表示创建表时各列的定义顺序。



#### 4. 标记列 `b,d` 未使用，并查看表 `t18` 在 `col$` 数据字典中 `col#`, `segcol#`, `intcol#` 的值

```sql
17:08:31 tq@TQ(tq-78)> -- 标记字段b,d 未使用
17:08:32 tq@TQ(tq-78)> alter table t18 set unused (b, d);

Table altered.

Elapsed: 00:00:00.17
17:08:35 tq@TQ(tq-78)> desc t18;
 Name                                      Null?    Type
 ----------------------------------------- -------- ----------------------------
 A                                                  NUMBER
 C                                                  NUMBER
```

```sql
17:12:23 tq@TQ(tq-78)> --
17:12:24 tq@TQ(tq-78)> col name for a30;
17:12:24 tq@TQ(tq-78)>
17:12:24 tq@TQ(tq-78)> SELECT col#, segcol#, name, intcol#, type#
17:12:24   2    FROM sys.col$
17:12:24   3   WHERE obj# IN (SELECT object_id
17:12:24   4                    FROM dba_objects
17:12:24   5                   WHERE object_name = 'T18'
17:12:24   6                     AND owner = 'TQ');

      COL#    SEGCOL# NAME                              INTCOL#      TYPE#
---------- ---------- ------------------------------ ---------- ----------
         1          1 A                                       1          2
         0          2 SYS_C00002_16121517:08:35$              2          2
         2          3 C                                       3          2
         0          4 SYS_C00004_16121517:08:35$              4          2

Elapsed: 00:00:00.00
17:12:24 tq@TQ(tq-78)>
17:12:24 tq@TQ(tq-78)> --
17:12:24 tq@TQ(tq-78)> col column_name for a30;
17:12:24 tq@TQ(tq-78)> col data_type for a10;
17:12:24 tq@TQ(tq-78)>
17:12:24 tq@TQ(tq-78)> select column_name,
17:12:24   2         data_type,
17:12:24   3         column_id,
17:12:24   4         hidden_column,
17:12:24   5         segment_column_id  seg_cid,
17:12:24   6         internal_column_id internal_cid
17:12:24   7    from dba_tab_cols
17:12:24   8   where owner = 'TQ'
17:12:24   9     and table_name = 'T18';

COLUMN_NAME                    DATA_TYPE   COLUMN_ID HID    SEG_CID INTERNAL_CID
------------------------------ ---------- ---------- --- ---------- ------------
A                              NUMBER              1 NO           1            1
SYS_C00002_16121517:08:35$     NUMBER                YES          2            2
C                              NUMBER              2 NO           3            3
SYS_C00004_16121517:08:35$     NUMBER                YES          4            4

Elapsed: 00:00:00.00
```

这里可以看到，在将列设为 `UNUSED` 之后，`COL#` 变为 `0`，其余的列的 `COL#` 重新排序。而此时该列在数据段上并没有被删除掉，因此其 `SEGCOL#` 列仍然保持原来的值。

这里原来的`B,D`列，其名字为系统自动生成的一列，命名形式为 `SYS_CNNNNN_YYMMDDHH24:MI:SS$` ，`NNNNN` 为原来的`COLUMN_ID`，前面补`0`补足成`5`位数。`hidden`已经变为`YES`，`COLUMN_ID`为`NULL`。其他两列`A,C`的`COLUMN_ID`顺序作了调整。这三列的`SEGMENT_COLUMN_ID`和`INTERNAL_COLUMN_ID`没有变化。

```sql
17:20:47 tq@TQ(tq-78)> -- 在DBA_TAB_COLUMNS视图中，B,D列已经没有显示出来。
17:20:48 tq@TQ(tq-78)> select column_name, data_type, column_id
17:20:48   2    from dba_tab_columns
17:20:48   3   where owner = 'TQ'
17:20:48   4     and table_name = 'T18';

COLUMN_NAME                    DATA_TYPE   COLUMN_ID
------------------------------ ---------- ----------
A                              NUMBER              1
C                              NUMBER              2

Elapsed: 00:00:00.00
```

#### 5. 插入2条测试数据，并 DUMP 数据块

```sql
17:25:50 tq@TQ(tq-78)> insert into t18 values (1, 1);

1 row created.

Elapsed: 00:00:00.02
17:25:52 tq@TQ(tq-78)> insert into t18 values (2, 2);

1 row created.

Elapsed: 00:00:00.00
17:25:59 tq@TQ(tq-78)> commit;

Commit complete.

Elapsed: 00:00:00.00
17:26:01 tq@TQ(tq-78)> select dbms_rowid.rowid_relative_fno(rowid) file#,dbms_rowid.rowid_block_number(rowid) block#,a,c from t18;

     FILE#     BLOCK#          A          C
---------- ---------- ---------- ----------
        35     133443          1          1
        35     133443          2          2

Elapsed: 00:00:00.04
17:26:17 tq@TQ(tq-78)> alter system checkpoint;

System altered.

Elapsed: 00:00:00.08
17:29:26 tq@TQ(tq-78)> alter system dump datafile 35 block 133443;

System altered.

Elapsed: 00:00:00.09
17:29:47 tq@TQ(tq-78)> select value from v$diag_info where name like 'De%';

VALUE
------------------------------------------------------------
/app/oracle/diag/rdbms/tq/tq/trace/tq_ora_25965.trc

Elapsed: 00:00:00.08
```

```
block_row_dump:
tab 0, row 0, @0x3f8e
tl: 10 fb: --H-FL-- lb: 0x1  cc: 3
col  0: [ 2]  c1 02
col  1: *NULL*
col  2: [ 2]  c1 02
tab 0, row 1, @0x3f84
tl: 10 fb: --H-FL-- lb: 0x1  cc: 3
col  0: [ 2]  c1 03
col  1: *NULL*
col  2: [ 2]  c1 03
end_of_block_dump
```

#### 6. 增加字段e，并插入数据，再次 DUMP 数据块

```sql
17:46:38 tq@TQ(tq-78)> alter table t18 add (e number);

Table altered.

Elapsed: 00:00:00.03
17:46:40 tq@TQ(tq-78)> desc t18;
 Name                                      Null?    Type
 ----------------------------------------- -------- ----------------------------
 A                                                  NUMBER
 C                                                  NUMBER
 E                                                  NUMBER

17:46:45 tq@TQ(tq-78)> --
17:47:05 tq@TQ(tq-78)> col name for a30;
17:47:05 tq@TQ(tq-78)>
17:47:05 tq@TQ(tq-78)> SELECT col#, segcol#, name, intcol#, type#
17:47:05   2    FROM sys.col$
17:47:05   3   WHERE obj# IN (SELECT object_id
17:47:05   4                    FROM dba_objects
17:47:05   5                   WHERE object_name = 'T18'
17:47:05   6                     AND owner = 'TQ');

      COL#    SEGCOL# NAME                              INTCOL#      TYPE#
---------- ---------- ------------------------------ ---------- ----------
         1          1 A                                       1          2
         0          2 SYS_C00002_16121517:08:35$              2          2
         2          3 C                                       3          2
         0          4 SYS_C00004_16121517:08:35$              4          2
         3          5 E                                       5          2

Elapsed: 00:00:00.00
17:47:05 tq@TQ(tq-78)>
17:47:05 tq@TQ(tq-78)> --
17:47:05 tq@TQ(tq-78)> col column_name for a30;
17:47:05 tq@TQ(tq-78)> col data_type for a10;
17:47:05 tq@TQ(tq-78)>
17:47:05 tq@TQ(tq-78)> select column_name,
17:47:05   2         data_type,
17:47:05   3         column_id,
17:47:05   4         hidden_column,
17:47:05   5         segment_column_id  seg_cid,
17:47:05   6         internal_column_id internal_cid
17:47:05   7    from dba_tab_cols
17:47:05   8   where owner = 'TQ'
17:47:05   9     and table_name = 'T18';

COLUMN_NAME                    DATA_TYPE   COLUMN_ID HID    SEG_CID INTERNAL_CID
------------------------------ ---------- ---------- --- ---------- ------------
A                              NUMBER              1 NO           1            1
SYS_C00002_16121517:08:35$     NUMBER                YES          2            2
C                              NUMBER              2 NO           3            3
SYS_C00004_16121517:08:35$     NUMBER                YES          4            4
E                              NUMBER              3 NO           5            5

Elapsed: 00:00:00.00
17:48:24 tq@TQ(tq-78)> update t18 set e = 1 where a = 1;

1 row updated.

Elapsed: 00:00:00.01
17:48:26 tq@TQ(tq-78)> update t18 set e = 2 where a = 2;

1 row updated.

Elapsed: 00:00:00.00
17:48:33 tq@TQ(tq-78)> commit;

Commit complete.

Elapsed: 00:00:00.00
17:51:32 tq@TQ(tq-78)> select dbms_rowid.rowid_relative_fno(rowid) file#,dbms_rowid.rowid_block_number(rowid) block#,a,c,e from t18;

     FILE#     BLOCK#          A          C          E
---------- ---------- ---------- ---------- ----------
        35     133443          1          1          1
        35     133443          2          2          2

Elapsed: 00:00:00.01
17:51:43 tq@TQ(tq-78)> alter system checkpoint;

System altered.

Elapsed: 00:00:00.02
17:51:58 tq@TQ(tq-78)> alter system dump datafile 35 block 133443;

System altered.

Elapsed: 00:00:00.01
17:52:19 tq@TQ(tq-78)> alter system checkpoint;

System altered.

Elapsed: 00:00:00.02
17:52:57 tq@TQ(tq-78)> select value from v$diag_info where name like 'De%';

VALUE
------------------------------------------------------------
/app/oracle/diag/rdbms/tq/tq/trace/tq_ora_25965.trc

Elapsed: 00:00:00.03
```

```
block_row_dump:
tab 0, row 0, @0x3f76
tl: 14 fb: --H-FL-- lb: 0x2  cc: 5
col  0: [ 2]  c1 02
col  1: *NULL*
col  2: [ 2]  c1 02
col  3: *NULL*
col  4: [ 2]  c1 02
tab 0, row 1, @0x3f68
tl: 14 fb: --H-FL-- lb: 0x2  cc: 5
col  0: [ 2]  c1 03
col  1: *NULL*
col  2: [ 2]  c1 03
col  3: *NULL*
col  4: [ 2]  c1 03
end_of_block_dump
```

#### 7. 删除字段e，会同时删除标记为未使用的字段

```sql
17:56:02 tq@TQ(tq-78)> alter table t18 drop (e);

Table altered.

Elapsed: 00:00:00.20
17:56:03 tq@TQ(tq-78)> desc t18;
 Name                                      Null?    Type
 ----------------------------------------- -------- ----------------------------
 A                                                  NUMBER
 C                                                  NUMBER

17:56:10 tq@TQ(tq-78)> --
17:56:26 tq@TQ(tq-78)> col name for a30;
17:56:26 tq@TQ(tq-78)>
17:56:26 tq@TQ(tq-78)> SELECT col#, segcol#, name, intcol#, type#
17:56:26   2    FROM sys.col$
17:56:26   3   WHERE obj# IN (SELECT object_id
17:56:26   4                    FROM dba_objects
17:56:26   5                   WHERE object_name = 'T18'
17:56:26   6                     AND owner = 'TQ');

      COL#    SEGCOL# NAME                              INTCOL#      TYPE#
---------- ---------- ------------------------------ ---------- ----------
         1          1 A                                       1          2
         2          2 C                                       2          2

Elapsed: 00:00:00.00
17:56:26 tq@TQ(tq-78)>
17:56:26 tq@TQ(tq-78)> --
17:56:26 tq@TQ(tq-78)> col column_name for a30;
17:56:26 tq@TQ(tq-78)> col data_type for a10;
17:56:26 tq@TQ(tq-78)>
17:56:26 tq@TQ(tq-78)> select column_name,
17:56:26   2         data_type,
17:56:26   3         column_id,
17:56:26   4         hidden_column,
17:56:26   5         segment_column_id  seg_cid,
17:56:26   6         internal_column_id internal_cid
17:56:26   7    from dba_tab_cols
17:56:26   8   where owner = 'TQ'
17:56:26   9     and table_name = 'T18';

COLUMN_NAME                    DATA_TYPE   COLUMN_ID HID    SEG_CID INTERNAL_CID
------------------------------ ---------- ---------- --- ---------- ------------
A                              NUMBER              1 NO           1            1
C                              NUMBER              2 NO           2            2

Elapsed: 00:00:00.01
17:56:27 tq@TQ(tq-78)> select dbms_rowid.rowid_relative_fno(rowid) file#,dbms_rowid.rowid_block_number(rowid) block#,a,c from t18;

     FILE#     BLOCK#          A          C
---------- ---------- ---------- ----------
        35     133443          1          1
        35     133443          2          2

Elapsed: 00:00:00.01
17:56:47 tq@TQ(tq-78)> alter system checkpoint;

System altered.

Elapsed: 00:00:00.02
17:56:52 tq@TQ(tq-78)> alter system dump datafile 35 block 133443;

System altered.

Elapsed: 00:00:00.01
17:57:00 tq@TQ(tq-78)> alter system checkpoint;

System altered.

Elapsed: 00:00:00.02
17:57:05 tq@TQ(tq-78)> select value from v$diag_info where name like 'De%';

VALUE
------------------------------------------------------------
/app/oracle/diag/rdbms/tq/tq/trace/tq_ora_25965.trc

Elapsed: 00:00:00.00
```

```
block_row_dump:
tab 0, row 0, @0x3f76
tl: 9 fb: --H-FL-- lb: 0x1  cc: 2
col  0: [ 2]  c1 02
col  1: [ 2]  c1 02
tab 0, row 1, @0x3f68
tl: 9 fb: --H-FL-- lb: 0x1  cc: 2
col  0: [ 2]  c1 03
col  1: [ 2]  c1 03
end_of_block_dump
```

可以看出`B,D`列和`E`列已经被删除。从这个实验就可以看出，在删除`E`时会将`UNUSED列`一并删除。

DUMP出数据块可以发现，块中每一行只有2列(即：A,C)。

**总结**：

因此 `SET UNUSED` 只是标记该列未使用，只是修改了数据字典，速度较快。

`SET UNUSED (<column_name>)` 比 `DROP (column_name)` 可以节省时间，因为只需修改数据字典不用动到数据块。

而`DROP (<column_name>)`，不仅修改数据字典，而且修改实际的块数据。如果表比较大，会耗费比较长的时间，会阻止其他dml的，所以一般都是在系统空闲下来的时候才做drop的。


-- The End --
