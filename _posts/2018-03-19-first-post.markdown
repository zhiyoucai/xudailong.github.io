---
layout: post
title:  "Next-Key Lock浅析"
date:   2018-3-19 18:36:35
categories: MySQL
tags: MySQL
excerpt: InnoDB在RR级别下完成幻读保护的经典算法
mathjax: true
---

# 什么是幻读

同一事务下，连续执行两次同样的SQL可能得到不同的结果，第二次的SQL可能返回了不存在的行

官方给出了一个例子：

child表只有id为90和102的记录，执行这样一条查询语句：

```sql
select * from child where id > 100 for update;
```

如果此时没有锁锁定90到100这个范围的话，另一个线程可能会成功插入一条id为101的数据，那么再次执行这个查询，就会出现一条id为101的记录，这就是幻读问题。

InnoDB采用Next-Key Lock解决幻读问题。

# 举例说明

新建一张表：

```sql
CREATE TABLE `test` (
  `id` int(11) DEFAULT NULL,
  KEY `id` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

insert into test values (1), (3), (5), (8), (11);
```

注意，这里id上是有索引的，因为该算法总是会去锁住索引记录。

现在，该索引可能被锁住的范围如下：

(-∞, 1], (1, 3], (3, 5], (5, 8], (8, 11], (11, +∞)

根据下面的方式开启事务执行SQL：

|order|Session A|Session B|
|---|---|---|
|1|begin;||
|2|select * from test where id = 8 for update;||
|3||begin;|
|4||insert into test select 1;|
|5||insert into test select 4;|
|6||insert into test select 5;|
|7||insert into test select 9;|
|8||insert into test select 12;|

Session A执行后会锁住的范围：

(5, 8], (8, 11]

除了锁住8所在的范围，还会锁住下一个范围，所谓Next-Key。

Session B执行完第五步，可能的锁范围会变成：

(-∞, 1], (1, 3], (3, 4], (4, 8], (8, 11], (11, +∞)

实际锁住的范围：

(4, 8], (8, 11]

这样，Session B执行到第六步会阻塞，跳过第六步不执行，第七步也会阻塞，但是并不阻塞第八步。

如果id是唯一索引，那么此时Session B的所有步骤中只有第四步和第六步会报唯一键约束错误，其他步骤全都成功。这是因为唯一索引上不采用Next-Key Lock，而是只锁住一行。