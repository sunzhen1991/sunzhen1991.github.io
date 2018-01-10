---
layout: post
title: "Innodb插入顺序的影响"
categories: 数据库
tags:  [mysql]  
author: sunzhen
---

## 测试

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

不管myisam还是innodb的索引实际上都是使用b+树实现的，只是具体实现方式不同。

### myisam

MyISAM索引文件和数据文件是分离的，索引文件仅保存数据记录的地址。myisam按照数据的插入顺序存储在磁盘上。自己不会画，从网上找来的几个图说明myisam的存储结构。

myisam的主键索引结构如下：

![](https://raw.githubusercontent.com/sunzhen1991/sunzhen1991.github.io/master/_posts/Pic/1801/myisam_primary.png)

叶子节点中保存的实际上是指向存放数据的物理块的指针。也不难看出为什么MYISAM引擎的索引文件（.MYI）和数据文件(.MYD)是相互独立的。

myisam的二级索引和主键索引没什么区别：

![](https://raw.githubusercontent.com/sunzhen1991/sunzhen1991.github.io/master/_posts/Pic/1801/myisam_second.png)


### innodb

聚簇索引是一种索引的存储方式，而不是索引类型。在聚簇索引中，表的数据和主键（Primary Key）共同组成了一个索引结构，叶节点其实就是实际的表记录，我们常规理解上的索引信息全部在分支节点上面。除了聚簇索引之外的其他所有索引在Innodb中被称为二级索引。二级索引就和普通的B-Tree索引差不多了，但是和myisam不同的是，innodb的二级索引的叶节点保存的不是数据地址而是主键值，所以如果通过二级索引找数据的话，其实是先通过二级索引找到主键值，然后通过主键索引找到数据。

聚簇索引有以下优点：
- 可以把相关数据保存在一起。例如实现电子邮件时，可以根据用户ID来聚集数据，这样只需要从磁盘读取少数的数据页就能获取某个用户的全部邮件。如果没有使用聚族索引，则每封邮件都可能导致一次磁盘I/O；
- 聚族索引将索引和数据保存在同一个B-Tree中，因此从聚族索引中获取数据通常比在非聚族索引中查找更快
- 使用覆盖索引扫描的查询可以直接使用节点中的主键值。

聚簇索引有以下缺点：
- 数据在内存中的话，聚簇就没优势了
- 依赖插入顺序（详见上面的测试）
- 更新索引代价很高
- 插入新行时可能导致页分裂
- 二级索引访问需要查找两次

innodb的主键索引结构如下：

![](https://raw.githubusercontent.com/sunzhen1991/sunzhen1991.github.io/master/_posts/Pic/1801/innodb_primary.png)

聚簇索引中的每个叶子节点包含主键值、事务ID（用于mvcc）、回滚指针(rollback pointer用于事务和MVCC）和余下的列。

innodb的二级索引：

![](https://raw.githubusercontent.com/sunzhen1991/sunzhen1991.github.io/master/_posts/Pic/1801/innodb_second.png)


## 顺序插入和乱序插入

### 页分裂
关于页分裂，可以查看这文章

<http://php.net/manual/en/migration70.new-features.php>

### 对比

InnoDB会按主键索引顺序组织文件，如果按主键顺序插入，可以直接在最尾部加入。并且只填充页面的15/16，这样可以预留部分空间以供行修改，这样组织的数据是非常紧凑的。如果不是顺序插入的话：

- 原来页可能已经写到磁盘，需要重新读取目标页到内存中，造成大量io操作。
- 需要为新行寻找合适的位置，造成频繁页分裂，会移动大量数据，一次插入可能需要修改三个页
- 频繁页分裂会导致数据碎片，可能需要使用optimize table来优化

