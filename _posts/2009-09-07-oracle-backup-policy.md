---
layout: post
title:  "备份策略的选择"
date:   2009-09-07 +0800
categories: Oracle
tags: Oracle Oracle备份与恢复
author: dbtan
---

* content
{:toc}

### 1、数据量

```
如果数据量小，每天全备份
如果数据量大，要有rman的备份策略
```





### 2、归档否

```
非归档数据库：只有冷备份
归档：既可以冷备份，也可以热备份
```

### 3、是否是容易加载，而比恢复更快

```
如果数据是每天统一灌入，以后就是查询
可以每天冷备份，失败后先将数据库恢复到备份点，再重新灌入
```

### 4、是否有存储软件

```
如果有其它存储软件
最好归档，rman，增量
```

### 5、是否为裸设备

```
如果是裸设备
rman或逻辑备份
还有ocopy
ocopy from_file [to_file [a | size_1 [size_n]]]
ocopy -b from_file to_drive
ocopy -r from_drive to_dir
```

-- The End --
