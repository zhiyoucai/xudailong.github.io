---
layout: post
title:  "MySQL的varchar字段查询时的一点问题"
date:   2018-3-19 18:36:35
categories: MySQL
tags: MySQL 排错记录
excerpt: 为什么varchar查询时给一个int值会出现诡异的现象
mathjax: true
---

## 现象

执行这个SQL：

```sql
select * from test where nodeId = 0;
```
其中nodeId是个varchar类型的字段。

得到的结果如下：

![](https://ws1.sinaimg.cn/large/5fec9ab7ly1fpichnp4ycj20bo01j0sm.jpg)

明显是有问题的，这是MySQL的一个特点，就是在将varchar和数字进行比较的时候，会将varchar隐式的转换为数字，而不是将数字隐式的转换为字符。

## 原因

如果执行这个SQL就能看的更明白：

```sql
select cast(nodeId as signed) from test;
```
结果如下：

![](https://ws1.sinaimg.cn/large/5fec9ab7ly1fpici799itj205904mwec.jpg)

可以看到有两行转换后的结果是0，那么上面的SQL能查出两条记录来就不奇怪了。

因此在SQL编写时一定要严格按照类型写SQL，隐式转换可能会带来意想不到的问题。