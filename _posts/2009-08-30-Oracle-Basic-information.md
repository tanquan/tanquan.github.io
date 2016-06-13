---
layout: post
title:  "Oracle基本信息检查"
date:   2009-08-30 +0800
categories: Oracle
tags: Oracle Oracle基本信息检查
author: dbtan
---

* content
{:toc}

## 检查Windows下的Oracle相关服务的状态

> 主要服务包括： OracleServiceORA10：Oracle实例服务
> OracleOraDb10g_home1TNSListenerFslmyoracle：	Oracle监听服务（OFS管理）
> OracleMSCSServices：Oracle Fail Safe for MSCS服务
> 在当前活动节点上，服务状态应该是正常“已启动”状态。





### 1. 检查Oracle初始化参数

```sql
select * from v$parameter;
```

### 2. 检查Oracle初始化参数

```sql
select * from v$parameter;
```

### 3. 检查Oracle的实例状态

```sql
select instance_name,version,status,database_status
from v$instance;
其中"STATUS"表示Oracle当前的实例状态，必须为"OPEN"；"DATABASE_STATUS"表示Oracle当前数据库的状态，必须为"ACTIVE"。
```

### 4. 检查后台线程的状态

```sql
Select name,Description
From V$BGPROCESS
Where Paddr<>'00';
```

### 5. 检查系统全局区SGA信息

```sql
select * from v$sga;
检查SGA各部份的分配情况，与实际内存比较是否合理。
```

### 6. 检查SGA各部分占用内存状况

```sql
select * from v$sgastat;
检查有无占用大量Shared pool的对象，及是否有内存浪费情况。
```

### 7. 检查系统SCN号

```sql
select dbms_flashback.get_system_change_number from dual;
select current_scn from v$database;
```

### 8. 检查数据库状态

```sql
select name,log_mode,open_mode from v$database;
```

### 9. 检查当前数据库的操作系统平台

```sql
select platform_name from v$database;
```

### 10. 检查数据库的大小，和空间使用情况

```sql
col tablespace format a20
select b.file_id　　文件ID,
　　b.tablespace_name　　表空间,
　　b.file_name　　　　　物理文件名,
　　b.bytes　　　　　　　总字节数,
　　(b.bytes-sum(nvl(a.bytes,0)))　　　已使用,
　　sum(nvl(a.bytes,0))　　　　　　　　剩余,
　　sum(nvl(a.bytes,0))/(b.bytes)*100　剩余百分比
　　from dba_free_space a,dba_data_files b
　　where a.file_id=b.file_id
　　group by b.tablespace_name,b.file_name,b.file_id,b.bytes
　　order by b.tablespace_name
/
　　dba_free_space --表空间剩余空间状况
　　dba_data_files --数据文件空间占用情况
```

### 11. 检查数据库的创建日期和归档方式

```sql
Select Created, Log_Mode, Log_Mode From V$Database;
```

### 12. 检查数据库是否处于归档模式，并启动了自动归档进程

```sql
archive log list;
```

### 13. 检查NLS信息（包括字符集）

```sql
select * from nls_database_parameters;
'NLS_LANGUAGE' || 'NLS_TERRITORY' || 'NLS_CHARACTERSET' 即字符集。
```

### 14. 检查表空间的名称、状态及大小

```sql
select t.tablespace_name, t.status, round(sum(bytes/(1024*1024)),0) ts_size
 from dba_tablespaces t, dba_data_files d
 where t.tablespace_name = d.tablespace_name
group by t.tablespace_name, t.status;
```

### 15. 检查每个表空间占用空间的大小

```sql
Select Tablespace_Name,Sum(bytes)/1024/1024
From Dba_Segments Group By Tablespace_Name;
```

### 16. 检查表空间物理文件的名称及大小

```sql
select tablespace_name, file_id, file_name,
round(bytes/(1024*1024),0) total_space
from dba_data_files
order by tablespace_name;
```

### 17. 查询表空间的剩余大小

```sql
select tablespace_name,sum(bytes)/(1024*1024) as free_space
from dba_free_space
group by tablespace_name;
```

### 18. 检查表空间的使用情况

```sql
SELECT A.TABLESPACE_NAME,A.BYTES TOTAL,B.BYTES USED, C.BYTES FREE,
(B.BYTES*100)/A.BYTES "% USED",(C.BYTES*100)/A.BYTES "% FREE"
FROM SYS.SM$TS_AVAIL A,SYS.SM$TS_USED B,SYS.SM$TS_FREE C
WHERE A.TABLESPACE_NAME=B.TABLESPACE_NAME
AND A.TABLESPACE_NAME=C.TABLESPACE_NAME;
```

### 19. 检查表空间碎块状况

```sql
col tablespace_name form a25
select tablespace_name, count(*) chunks,
max(bytes)/1024/1024 max_chunk,
sum(bytes)/1024/1024 total_space  
from dba_free_space group by tablespace_name;  
如果最大可用块(max_chunk)与总大小(total_space)相比太小，要考虑接合表空间碎片或重建某些数据库对象。 碎片接合的方法: alter tablespace 表空间名 coalesce;
```

### 20. 检查回滚段名称、状态及大小

```sql
select segment_name, tablespace_name, r.status,
(initial_extent/1024) InitialExtent,(next_extent/1024) NextExtent,
max_extents, v.curext CurExtent
From dba_rollback_segs r, v$rollstat v
Where r.segment_id = v.usn(+)
order by segment_name ;
```

### 21. 检查控制文件状态

```sql
select * from v$controlfile;
```

### 22. 检查日志文件状态

```sql
select * from v$logfile;
```

### 23. 检查日志组信息

```sql
select * from v$log;
```

### 24. 检查数据文件状态

```sql
select file_name,status
from dba_data_files;
```

### 25. 检查数据文件存放路径

```sql
col file_name format a50
select tablespace_name,file_id,bytes/1024/1024,file_name
from dba_data_files order by file_id;
```

### 26. 检查数据文件的自动增长控制

```sql
select file_name,autoextensible from dba_data_files;
```

### 27. 检查临时数据文件路径

```sql
select file_name
from Dba_temp_files;
```

### 28. 检查闪回恢复区的路径

```sql
select name from v$recovery_file_dest;
```

### 29. 检查数据库库对象

```sql
select owner, object_type, status, count(*) count#
from all_objects group by owner, object_type, status;
```

### 30. 检查数据库的版本　

```sql
Select version FROM Product_component_version
Where SUBSTR(PRODUCT,1,6)='Oracle';

VERSION
---------------
11.1.0.6.0
依次为：版本号11、新特性版本号1、维护版本号0、普通的补丁设置号码6、特殊的平台补丁设置号码0
```

### 31. 检查数据库的创建日期和归档方式

```sql
Select Created, Log_Mode, Log_Mode From V$Database;
```

### 32. 检查当前所有对象

```sql
select * from tab;
```

### 33. 检查当前连接用户

```sql
show user;
```

### 34. 检查已有用户：

```sql
select username from dba_users;
```
### 35. 检查所有表、索引、存储过程、触发器、包等对象的状态

```sql
select owner,object_name,object_type
from dba_objects where status!='VALID'
and owner!='SYS' and owner!='SYSTEM';
```

### 36. 检查当前用户的缺省表空间、临时表空间

```sql
select username,default_tablespace, temporary_tablespace from user_users;
```

### 37. 检查当前用户的角色

```sql
select * from user_role_privs;
```

### 38. 检查当前用户的系统权限和表级权限

```sql
select * from user_sys_privs;
select * from user_tab_privs;
```
### 39. 用户下所有的表

```sql
select * from user_tables;
```

### 40. 检查各个表的大小

```sql
检查当前用户每个表占用空间的大小：
Select Segment_Name,Sum(bytes)/1024/1024
From User_Extents Group By Segment_Name
注：段名即表名
按数据对象大小排序
Select Segment_Name,segment_type, Sum(bytes)/1024/1024 as MB
From User_Extents
Group By Segment_Name, segment_type
Order by MB;
```

### 41. 检查某表的创建时间

```sql
select object_name,created from user_objects where object_name=upper('&table_name');
```

### 42. 检查名称包含log字符的表

```sql
select object_name,object_id from user_objects
 where instr(object_name,'LOG')>0;
```

### 43. 检查某表的大小

```sql
select sum(bytes)/(1024*1024) as "size(M)" from user_segments
 where segment_name=upper('&table_name');
```

### 44. 检查放在内存区里的表

```sql
select table_name,cache from user_tables where instr(cache,'Y')>0;
```
### 45. 检查索引个数和类别

```sql
select index_name,index_type,table_name from user_indexes order by table_name;
```
### 46. 检查索引中被索引的字段

```sql
select * from user_ind_columns where index_name=upper('&index_name');
```
### 47. 检查索引的大小

```sql
select sum(bytes)/(1024*1024) as "size(M)" from user_segments
 where segment_name=upper('&index_name');
```

### 48. 检查是否有失效的索引

```sql
select index_name, owner, table_name, tablespace_name
from dba_indexes
where owner not in ('SYS','SYSTEM') and status != 'VALID';
如果有记录返回，考虑重建这些索引。
```

### 49. 检查是否有无效的对象

```sql
select object_name,      object_type,      owner,      status  
from dba_objects  
where status !='VALID'    
and owner not in ('SYS','SYSTEM')
and object_type in  ('TRIGGER','VIEW','PROCEDURE','FUNCTION');  
如果存在无效的对象，手工重新编译一下。
```

### 50. 检查序列号

```sql
select * from user_sequences;
last_number是当前值
```

### 51. 检查序列号的使用

```sql
select sequence_owner, sequence_name, min_value,
max_value, increment_by, last_number,
cache_size, cycle_flag from dba_sequences;
检查是否存在即将达到max_value的sequence。
```

### 52. 检查视图的名称

```sql
select view_name from user_views;
```

### 53. 检查创建视图的select语句

```sql
set view_name,text_length from user_views;
set long 2000;                --说明：可以根据视图的text_length值设定set long 的大小
select text from user_views where view_name=upper('&view_name');
```

### 54. 检查同义词的名称

```sql
select * from user_synonyms;
```

### 55. 检查某表的约束条件

```sql
select constraint_name, constraint_type,search_condition, r_constraint_name
 from user_constraints where table_name = upper('&table_name');

select c.constraint_name,c.constraint_type,cc.column_name
 from user_constraints c,user_cons_columns cc
 where c.owner = upper('&table_owner') and c.table_name = upper('&table_name')
 and c.owner = cc.owner and c.constraint_name = cc.constraint_name
 order by cc.position;
```

### 56. 检查函数和过程的状态

```sql
select object_name,status from user_objects where object_type='FUNCTION';
select object_name,status from user_objects where object_type='PROCEDURE';
```

### 57. 检查函数和过程的源代码

```sql
select text from all_source where owner=user and name=upper('&plsql_name');
```
### 58. 检查当前数据库有几个用户连接

```sql
用系统管理员权限执行，
select username,sid,serial#, machine, status from v$session;
USERNAME：建立该会话的用户名；
SID：会话(session)的ID号；
SERIAL#：会话的序列号，和SID一起用来唯一标识一个会话；
PROGRAM：这个会话是用什么工具连接到数据库的；
MACHINE：这个会话是从哪台电脑连过来的
STATUS当前这个会话的状态，ACTIVE表示会话正在执行某些任务，INACTIVE表示当前会话没有执行任何操作；
如果要停某个连接用
SQL> alter system kill session 'sid,serial#';
如果这命令不行,找它UNIX的进程数
SQL> select pro.spid
from v$session ses,v$process pro
where ses.sid=21 and ses.paddr=pro.addr;
说明：21是某个连接的sid数
然后用 kill 命令杀此进程号。
```

### 59. 检查定时作业的完成情况

```sql
select job,log_user,last_date,failures
from dba_jobs;

select  job, this_date, this_sec, next_date, next_sec, failures, what
from dba_jobs where failures !=0 or failures is not null;
如果FAILURES列是一个大于0的数的话，说明JOB运行失败，要进一步的检查。
```

-- The End --
