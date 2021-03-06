---
layout: post
title:  "MySQL order by"
date:   2017-03-23 +0800
categories: MySQL SQL
tags: MySQL SQL
author: Jimmy Lee
---

* content
{:toc}

### 需求
一个表字段叫status,值是1,2,3,4,5,6,7,8,9

project manager想进行分类,然后某一类的排序规则是按照status 1,5,3,4 排序,相同状态的按时间排序

### 实现sql
```sql
select demo_id,status,create_time from demo_table
where userId='123456' and status in (1,5,3,4)
order by field(status,1,5,3,4), demo_id desc
limit 0,30;
```






```sql
+----+-------------+-----------------+------+---------------+------------+---------+-------+------+----------+----------------------------------------------------+
| id | select_type | table           | type | possible_keys | key        | key_len | ref   | rows | filtered | Extra                                              |
+----+-------------+-----------------+------+---------------+------------+---------+-------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | demo_table      | ref  | idx_userId    | idx_userId | 202     | const |   63 |   100.00 | Using index condition; Using where; Using filesort |
+----+-------------+-----------------+------+---------------+------------+---------+-------+------+----------+----------------------------------------------------+
1 row in set, 1 warning (0.00 sec)
```
这种查询用到了一个函数order by field,看执行计划用到了file sort,查询了一下并不是用到了磁盘排序,而是一个排序操作

在开发环境把数据造到1000w,查询依然可以走0.0几秒完成,暂时使用这种方式,虽说大量查询可能会有性能问题,先标记一下,慢慢观察,出现性能问题使用下面的方式改写


```sql
select * from (
select * from (select demo_id,status,create_time from demo_table where userId='123456789' and status=1 order by demo_id desc) t1
union all
select * from (select demo_id,status,create_time from demo_table where userId='123456789' and status=5 order by demo_id desc) t2
union all
select * from (select demo_id,status,create_time from demo_table where userId='123456789' and status=3 order by demo_id desc) t3
union all
select * from (select demo_id,status,create_time from demo_table where userId='123456789' and status=4 order by demo_id desc) t4
) t5
limit 0,30;
```
```sql
+----+--------------+-----------------+------+---------------+------------+---------+-------+------+----------+-----------------+
| id | select_type  | table           | type | possible_keys | key        | key_len | ref   | rows | filtered | Extra           |
+----+--------------+-----------------+------+---------------+------------+---------+-------+------+----------+-----------------+
|  1 | PRIMARY      | <derived2>      | ALL  | NULL          | NULL       | NULL    | NULL  |    8 |   100.00 | NULL            |
|  2 | DERIVED      | <derived3>      | ALL  | NULL          | NULL       | NULL    | NULL  |    2 |   100.00 | NULL            |
|  3 | DERIVED      | demo_table      | ref  | idx_userId    | idx_userId | 202     | const |    1 |   100.00 | Using where     |
|  4 | UNION        | <derived5>      | ALL  | NULL          | NULL       | NULL    | NULL  |    2 |   100.00 | NULL            |
|  5 | DERIVED      | demo_table      | ref  | idx_userId    | idx_userId | 202     | const |    1 |   100.00 | Using where     |
|  6 | UNION        | <derived7>      | ALL  | NULL          | NULL       | NULL    | NULL  |    2 |   100.00 | NULL            |
|  7 | DERIVED      | demo_table      | ref  | idx_userId    | idx_userId | 202     | const |    1 |   100.00 | Using where     |
|  8 | UNION        | <derived9>      | ALL  | NULL          | NULL       | NULL    | NULL  |    2 |   100.00 | NULL            |
|  9 | DERIVED      | demo_table      | ref  | idx_userId    | idx_userId | 202     | const |    1 |   100.00 | Using where     |
| NULL | UNION RESULT | <union2,4,6,8>  | ALL  | NULL          | NULL       | NULL    | NULL  | NULL |     NULL | Using temporary |
+----+--------------+-----------------+------+---------------+------------+---------+-------+------+----------+-----------------+
10 rows in set, 1 warning (0.00 sec)
```
因为要union all,所以用了临时表,同样的1000w数据,查询cost和上面基本一样

比上面麻烦的就是有多个分类组合的时候需要写多个sql

-- The End --

> 原文链接: [http://illegalaccess.com/2017/03/23/order-by/](http://illegalaccess.com/2017/03/23/order-by/ "http://illegalaccess.com/2017/03/23/order-by/")
