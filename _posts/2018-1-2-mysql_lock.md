---
layout: post
title: "mysql锁总结"
categories: 数据库
tags:  [mysql]  
author: sunzhen
---


## 读写锁

### myisam

创建两个表cityi和countryi;
```mysql
CREATE TABLE `cityi` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` char(35) NOT NULL DEFAULT '',
  `population` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8

CREATE TABLE `countryi` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` char(52) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8
```
当有session1（简称s1）和session2（简称s2）同事连接到该数据库。当s1给cityi上读锁：

```mysql
lock tables cityi read;
```

在用LOCK TABLES给表显式加表锁时，必须同时取得所有涉及到表的锁，并且MySQL不支持锁升级。，在执行LOCK TABLES后，只能访问显式加锁的这些表，不能访问未加锁的表；如果加是读锁，那么只能执行查询操作，而不能执行更新操作。

此时s1再读其他表会报错,也不能插入,当s2对s1锁定的表插入或者更新数据时，会等待s1释放读锁。
```mysql
select * from countryi;
ERROR 1100 (HY000): Table 'countryi' was not locked with LOCK TABLES
mysql> insert into cityi(name,population) values("suzhou",200);
ERROR 1099 (HY000): Table 'cityi' was locked with a READ lock and can't be updated
```
如果想要处理并发插入，可以将concurrent_insert 设为2或者使用read local选项。但是更新依然会等待。

当s1给cityi加写锁时，s2对cityi的任何操作都要等待s1释放。

### innodb

innodb与myisam还是有很多不同的：

- innodb 支持事务
- innodb采用了行级锁 

首先，创建一个city表，包含id、城市名name，人口population。再创建一个国家表。

```mysql
CREATE TABLE `city` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` char(35) NOT NULL DEFAULT '',
  `population` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

CREATE TABLE `country` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` char(52) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

innodb有四种锁：

- 共享锁（S）-读锁-行锁
- 排他锁（X）-写锁-行锁
- 意向共享锁（IS）-表级 ：事务想要获得一张表中某几行的共享锁
- 意向排他锁（IX）-表级：事务想要获得一张表中某几行的排他锁

IS和IX由数据库自动申请。锁之间冲突关系如下(x小时欲加锁，y表示已有锁)：

| 已有\欲加锁  | X         | IX       | S              | IS       |
| ----------- |:---------:| --------:|:--------------:| --------:|
| X      | 冲突           | 冲突      | 冲突           | 冲突      |
| IX     | 冲突           | 兼容      | 冲突           | 兼容      |
| S      | 冲突           | 冲突      | 兼容           | 兼容      |
| IS     | 冲突           | 兼容      | 兼容           | 兼容      |

需要注意的是，InnoDB行锁是通过给索引上的索引项加锁来实现的，只有通过索引条件检索数据，InnoDB才使用行级锁。否则，InnoDB将使用表锁！实验如下：

```mysql
s1:
MySQL [test]> start transaction;
Query OK, 0 rows affected (0.00 sec)

MySQL [test]> select * from city where population=10000 for update;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  1 | 北京   |      10000 |
|  5 | 北京   |      10000 |
+----+--------+------------+
2 rows in set (0.00 sec)

之后s2：
MySQL [test]> start transaction;
Query OK, 0 rows affected (0.00 sec)

MySQL [test]> select * from city where population=10002 for update;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```
由于s1并没有使用索引，所以s1使用了表所，s2会等待。当使用索引时：

```mysql
S1:

MySQL [test]> start transaction;
Query OK, 0 rows affected (0.00 sec)

MySQL [test]> select * from city where id=1 for update;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  1 | 北京   |      10000 |
+----+--------+------------+
1 row in set (0.00 sec)

S2:
MySQL [test]> start transaction;
Query OK, 0 rows affected (0.01 sec)

MySQL [test]> select * from city where id=2 for update;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  2 | 上海   |      10001 |
+----+--------+------------+
1 row in set (0.01 sec)
```
此时没有产生冲突。

#### 间隙锁（Next-Key）
当我们用范围条件时，并请求共享或排他锁时，InnoDB会给符合条件的已有数据记录的索引项加锁；对于键值在条件范围内但并不存在的记录，叫做“间隙（GAP)”，InnoDB也会对这个“间隙”加锁，这种锁机制就是间隙锁（Next-Key）。间隙锁是为了消灭幻读问题。

```mysql
S1:
MySQL [test]> select * from city for update;
+-----+---------+------------+
| id  | name    | population |
+-----+---------+------------+
|   1 | 北京    |      10000 |
|   2 | 上海    |      10001 |
|   3 | 杭州    |      10002 |
|   4 | 深圳    |      10003 |
|   5 | 北京    |      10022 |
|   6 | 上海    |      10001 |
|   7 | 杭州    |      10002 |
|  60 | haikou  |      10012 |
|  80 | chengdu |      10011 |
| 100 | hefei   |      10010 |
+-----+---------+------------+
10 rows in set (0.00 sec)

MySQL [test]> start transaction;
Query OK, 0 rows affected (0.00 sec)

MySQL [test]> select * from city where id>70 for update;

S2:
MySQL [test]> start transaction;
Query OK, 0 rows affected (0.00 sec)

MySQL [test]> insert into city(id,name,population) values(11,"nanjing",10017);
Query OK, 1 row affected (0.00 sec)

MySQL [test]> insert into city(id,name,population) values(90,"hasaki",10019);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
MySQL [test]> insert into city(id,name,population) values(61,"haikou",10012);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```
当id在7~60范围可以插入，当id>70时，s2没有办法插入。但是当id在60~70范围内s2也无法插入。

### 隐式锁和显示锁

在innodb中，除了显示加锁之外，在事务中，会根据隔离级别在需要的时候自动加锁。

## 事务隔离级别

不同隔离级别通过不同的读写锁和锁粒度来实现的。高并发下，可能会出现脏读和幻读问题。

- 脏读（dirty read）：两个事务，事务A读取到了事务B尚未提交的数据，这便是脏读。
- 不重复读（nonrepeated read）：两个事务，事务A与事务B，事务A在自己执行的过程中，执行了两次相同查询，第一次查询事务B未提交，第二次查询事务B已提交，从而造成两次查询结果不一样
- 幻读（phantom read）,如果事务B是一个会影响查询结果的insert操作，则好像新多出来的行像幻觉一样，因此被称为幻读。

sql规定了四种隔离级别：

- READ UNCOMMITTED (未提交读)：事务A中的修改，未提交前对事务B可见。
- READ COMMITTED (提交读)： A的修改在提交之后可以被B读取。
- REPEATABLE READ (可重复读) ：在InnoDB中，RR隔离级别保证对读取到的记录加锁 (记录锁)，同时保证对读取的范围加锁，新的满足查询条件的记录不能够插入 (间隙锁)
- SERIALIZABLE （可串行化）：强制事务串行执行，每一行数据上都加锁，所以可能会导致大量的锁争用

### 脏读
S1开启隔离级别为未提交读的事务
```mysql
MySQL [test]> start transaction;
Query OK, 0 rows affected (0.00 sec)

MySQL [test]> select * from city where id=2;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  2 | 上海   |       9821 |
+----+--------+------------+
1 row in set (0.00 sec)

```

S2修改上海的人口为10000：
```mysql
MySQL [test]> start transaction;
Query OK, 0 rows affected (0.00 sec)

MySQL [test]> update city set population=10000 where id=2;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MySQL [test]> select * from city where id=2;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  2 | 上海   |      10000 |
+----+--------+------------+
1 row in set (0.00 sec)
```

S1再次查看id为2的数据
```mysql
MySQL [test]> select * from city where id=2;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  2 | 上海   |      10000 |
+----+--------+------------+
1 row in set (0.00 sec)
```
S2没有提交，但是S1中可以看到S2的更改，产生脏读。

### 不可重复读

S1设置隔离级别为可重复读
```mysql
MySQL [test]> start transaction;
Query OK, 0 rows affected (0.00 sec)


MySQL [test]> select * from city where id=2;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  2 | 上海   |      10000 |
+----+--------+------------+
1 row in set (0.00 sec)
```

S2修改上海的人口为10000：
```mysql
MySQL [test]> start transaction;
Query OK, 0 rows affected (0.00 sec)

MySQL [test]> update city set population=10000 where id=2;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MySQL [test]> select * from city where id=2;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  2 | 上海   |      10000 |
+----+--------+------------+
1 row in set (0.00 sec)
```

S1查看id为2的数据
```mysql
MySQL [test]> select * from city where id=2;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  2 | 上海   |       9821 |
+----+--------+------------+
1 row in set (0.00 sec)
```
S2提交事务：
```mysql
MySQL [test]> commit;
Query OK, 0 rows affected (0.00 sec)
```
S1查看id为2的数据
```mysql
MySQL [test]> select * from city where id=2;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  2 | 上海   |      10000 |
+----+--------+------------+
1 row in set (0.00 sec)
```
S2提交前后，S1中可以看到数据不同，既不可重复读。


### 幻读
S1开启隔离级别为可重复读的事务
```mysql
MySQL [test]> start transaction;
Query OK, 0 rows affected (0.00 sec)

MySQL [test]> select * from city;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  1 | 北京   |      10000 |
|  2 | 上海   |      10033 |
|  3 | 杭州   |      10002 |
|  4 | 深圳   |      10003 |
+----+--------+------------+
4 rows in set (0.00 sec)
```

S2修改上海的人口为10000，并且提交：
```mysql
MySQL [test]> update city set population=10000 where id=2;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MySQL [test]> select * from city;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  1 | 北京   |      10000 |
|  2 | 上海   |      10000 |
|  3 | 杭州   |      10002 |
|  4 | 深圳   |      10003 |
+----+--------+------------+
4 rows in set (0.00 sec)

MySQL [test]> commit;
```

S1查看city数据
```mysql
MySQL [test]> select * from city;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  1 | 北京   |      10000 |
|  2 | 上海   |      10033 |
|  3 | 杭州   |      10002 |
|  4 | 深圳   |      10003 |
+----+--------+------------+
4 rows in set (0.00 sec)
```
此时，s1读取的数据并没有改变，可见解决了不可重复读的问题。

S2，执行insert，提交事务：
```mysql
MySQL [test]> insert into city(name,population) values("广州",12345);
Query OK, 1 row affected (0.01 sec)

MySQL [test]> select * from city;
+-----+--------+------------+
| id  | name   | population |
+-----+--------+------------+
|   1 | 北京   |      10000 |
|   2 | 上海   |      10000 |
|   3 | 杭州   |      10002 |
|   4 | 深圳   |      10003 |
| 102 | 广州   |      12345 |
+-----+--------+------------+
5 rows in set (0.00 sec)

MySQL [test]> commit;
Query OK, 0 rows affected (0.00 sec)
```

S1查看city数据，s1也插入数据。
```mysql
MySQL [test]> select * from city;
+----+--------+------------+
| id | name   | population |
+----+--------+------------+
|  1 | 北京   |      10000 |
|  2 | 上海   |      10033 |
|  3 | 杭州   |      10002 |
|  4 | 深圳   |      10003 |
+----+--------+------------+
4 rows in set (0.00 sec)

MySQL [test]> insert into city(id,name,population) values(102,"guangzhou",56789); 
ERROR 1062 (23000): Duplicate entry '102' for key 'PRIMARY'
```
S2并没有看到id为102的数据，但是当插入id为102的数据时却会报错，这就是幻读问题。其实，关于幻读另一个定义是：

>在一个事务的两次查询中数据笔数不一致，例如有一个事务查询了几行数据，而另一个事务却在此时插入了几行数据，先前的事务在接下来的查询中，就会发现有几行数据是它先前所没有的。

但是由于innodb的mvcc的存在，这种情况其实是不成立的。

## innodb多版本并发控制

MVCC是行级锁的一个变种，通过保存数据在摸个时间的的快照来避免加锁操作。Innodb的MVVC通过在每行记录后面保存两个隐藏的列来实现（行版本号/行删除版本号）。行版本号保存行的创建时间，行删除表示保存行的过期时间，时间其实是事务的版本号。

当执行select时，通过两个条件半段记录是否作为返回结果：

- 行版本小于等于当前事务版本
- 行删除版本要么未定义，要么大于当前事务的版本号

插入时，新插入行的行版本号设置为当前事务版本号。

删除时，删除行的行删除版本号设置为当前事务版本。

更新时，插入一条新纪录，新纪录的行版本号设置为当前版本号，原纪录的行删除版本号设置为当前版本号。

注：MVCC只在REPEATABLE READhe READ COMITTED两个级别下工作。