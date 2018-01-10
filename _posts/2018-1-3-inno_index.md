---
layout: post
title: "Innodb页分裂"
categories: 数据库
tags:  [mysql]  
author: sunzhen
---

## 页分裂测试

首先，做个测试，创建两个表

```mysql
CREATE TABLE `t1` (
    id int primary key,
    c1 varchar(500),
    c2 varchar(500),
    c3 varchar(500),
    c4 varchar(500),
    c5 varchar(500)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8

CREATE TABLE `t2` (
    id int primary key,
    c1 varchar(500),
    c2 varchar(500),
    c3 varchar(500),
    c4 varchar(500),
    c5 varchar(500)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8
```

两个python脚本，插入100万条数据，顺序插入t1:
```python
#coding=utf-8
import MySQLdb
import time
import random

conn= MySQLdb.connect(
        host='127.0.0.1',
        port = 3306,
        user='root',
        passwd='xxxxx',
        db ='test',
        )
cur = conn.cursor()
strr = "a"*500
array1 = range(1,1000001)
start1 = time.time()
for i in array1:
        #插入一条数据
        cur.execute('insert into t1 values("%d","%s","%s","%s","%s","%s")' % (i,strr,strr,strr,strr,strr))
cur.close()
conn.commit()
conn.close()
end1 = time.time()
print "顺序插入程序执行时间：",end1-start1
```

乱序插入t2：

```python
#coding=utf-8
import MySQLdb
import time
import random

conn= MySQLdb.connect(
        host='127.0.0.1',
        port = 3306,
        user='root',
        passwd='xxxxx',
        db ='test',
        )
cur = conn.cursor()
strr = "a"*500
array2 = range(1,1000001)
random.shuffle(array2)
start2 = time.time()
for j in array2:
        #插入一条数据
        cur.execute('insert into t2 values("%d","%s","%s","%s","%s","%s")' % (j,strr,strr,strr,strr,strr))
cur.close()
conn.commit()
conn.close()
end2 = time.time()
print "非顺序插入程序执行时间：",end2-start2
```

结果如下：

| 测试项        |    t1            |        t2    |
| :------------ |:---------------:| -------------:|
| 时间(s)       | 533.831892967    | 988.436821938 |
| idb文件大小   | 2780823552       |   4575985664   |

可以看到t2不管是插入时间，还是磁盘占用都是大于t1的。


## innodb myisam数据分布

### myisam

MyISAM索引文件和数据文件是分离的，索引文件仅保存数据记录的地址。myisam按照数据的插入顺序存储在磁盘上。

![](https://raw.githubusercontent.com/woaielf/sunzhen1991.github.io/master/_posts/Pic/1801/170103-1.png)