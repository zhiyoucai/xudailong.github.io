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

如果此时没有锁锁定90到102这个范围的话，另一个线程可能会成功插入一条id为101的数据，那么再次执行这个查询，就会出现一条id为101的记录，这就是幻读问题。

InnoDB采用Next-Key Lock解决幻读问题。

# 举例说明

**以下所有的操作都是在Read Repeatable级别下进行测试，测试版本是MySQL官方5.7.21版本**

新建一张表：

```sql
CREATE TABLE `test` (
  `id` int(11) primary key auto_increment,
  `xid` int,
  KEY `xid` (`xid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

insert into test(xid) values (1), (3), (5), (8), (11);
```

注意，这里xid上是有索引的，因为该算法总是会去锁住索引记录。

现在，该索引可能被锁住的范围如下：

(-∞, 1], (1, 3], (3, 5], (5, 8], (8, 11], (11, +∞)

根据下面的方式开启事务执行SQL：

|order|Session A|Session B|
|---|---|---|
|1|begin;||
|2|select * from test where id = 8 for update;||
|3||begin;|
|4||insert into test(xid) values (1);|
|5||insert into test(xid) values (4);|
|6||insert into test(xid) values (5);|
|7||insert into test(xid) values (9);|
|8||insert into test(xid) values (11);|
|9||insert into test(xid) values (12);|

Session A执行后会锁住的范围：

(5, 8], (8, 11]

除了锁住8所在的范围，还会锁住下一个范围，所谓Next-Key。

这样，Session B执行到第六步会阻塞，跳过第六步不执行，第七步也会阻塞，但是并不阻塞第八步，第九步也不阻塞。

上面的结果似乎并不符合预期，因为11这个值看起来就是在(8, 11]区间里，而5这个值并不在(5, 8]这个区间里。

那么先引进一张图：

![](http://wx1.sinaimg.cn/large/5fec9ab7ly1fpsanraom2j20lp0gvq35.jpg)

辅助索引中黄色部分是被record lock锁住的行，除此之外还有两个Gap Lock，锁住了我上面说的范围。

先说为什么锁住了5的插入，观察主键索引，主键索引是自增的，因此在id=4这条记录之前，是不允许插入一条xid=5的记录的。

上面说的并不是很能让人信服,什么时候看了源码再说吧，不过记住这个结论能够推断出正确的锁定范围的。

不锁定xid=11的写入还是可以用id是自增的解释，B+树是有序的，并不会阻塞后续的插入。

**因此在判断Next-Key Lock的锁定范围的时候，首先要判断出区间，然后分析，判断区间边缘值是否会被锁定**

如果id是唯一索引，那么此时Session B的所有步骤中只有第四步和第六步会报唯一键约束错误，其他步骤全都成功。这是因为唯一索引上不采用Next-Key Lock，而是只锁一行。

如果session A执行这个SQL：

```sql
select * from test where xid = 1 for update;
```

这个时候的锁定范围是(-∞, 1], (1, 3]，但是不锁定3

如果Session B执行下面的SQL都会阻塞：

```sql
insert into test(xid) values (-100);
insert into test(xid) values (0);
insert into test(xid) values (1);
insert into test(xid) values (2);
```

这些都在上面说的区间内，也印证了我的看法，但是这条SQL开始就不会被阻塞了：

```sql
insert into test(xid) values (3);
```

下面来测试一下到正无穷大的是否满足我的理解：

session A执行这个SQL：

```sql
select * from test where xid = 11 for update;
```

推断锁定的范围是：(8, 11], (11, +∞)，锁定8和上述区间内的数据插入

现在session B执行这些SQL应该都会被阻塞：

```sql
insert into test(xid) values (8);
insert into test(xid) values (9);
insert into test(xid) values (10);
insert into test(xid) values (11);
insert into test(xid) values (99999);
```

但是session B执行这个SQL就不会阻塞：

```sql
insert into test(xid) values (7);
```

经过实际测试结果符合我的预期。

再研究一个主键不是自增的例子，这个例子来自何登成的博客，下面有链接，我稍微改动了一下。

首先建表：

```sql
create table test 
(
  uname varchar(10) primary key,
  xid int,
  key (xid)
);
```

插入数据如下：

```sql
insert into test(uname, xid) values ('a', 15),('c',10), ('e',6),('g', 10),('m',11),('z',2);
```

下面Session A试图锁住xid=10：

```sql
begin;
select * from test where xid=10 for update;
```

下面是主键索引和辅助索引的关系示意图：

![](http://wx3.sinaimg.cn/large/5fec9ab7ly1fpsbt4tkv1j20pv0g5wes.jpg)

锁住的范围推断应该是((6,e),(10,c)],((10,c),(10,g)]

因为不是自增主键了，所以情况会稍微麻烦一点，但是有一点原则要把握好就是B+树的连续性不能被破坏，那么可以判断下面的SQL一定被阻塞：

```sql
-- 因为(10,d)的组合会破坏B+树的连续性，可以自己画图
insert into test(uname, xid) values ('d', 10);

-- 因为(10,f)组合也会破坏B+树连续性
insert into test(uname, xid) values ('f', 10);

-- 同理也会阻塞
insert into test(uname, xid) values ('d', 7);
insert into test(uname, xid) values ('d', 8);
insert into test(uname, xid) values ('d', 9);
```

现在来分析一下本文开头时MySQL官方给的例子，需要注意的是如果没有显式的指定主键InnoDB会选择一个rowid作为主键，这个rowid是单调递增的。

```sql
select * from child where id > 100 for update;
```

这个SQL在表中只有90和102两个数据的情况下Next-Key Lock锁定的范围是：

(90, 102], (102, +∞)

因此在这个区间的保护下，官方例子中插入一个id=101的记录是会被阻塞的，这也就完成了幻读保护，同样插入90也是不可以的，插入比102大的数据也不可以。

而实际的测试也印证了我的推断。

# 参考资料

[1. MySQL 加锁处理分析（何登成）](http://hedengcheng.com/?p=771#_Toc374698316)

[2. Innodb锁机制：Next-Key Lock 浅谈](https://www.cnblogs.com/zhoujinyi/p/3435982.html)

[3. MySQL Document](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-next-key-locks)