---
title: 对MySQL关于order by+limit理解的一个误区 
date: 2017-12-11 23:48:44
tags:
- MySQL
- orderby
categories: MySQL
---
今天业务碰到一个线上问题,业务order by字段shopcode，用limit m,n做分页，希望按照shopcode排序获取所有的数据，但是遍历完一遍，发现有的数据没有获取到。
挺奇怪的，于是做了个实验，复现了情景。

```
mysql> show create table shop;
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                                                                          |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| shop  | CREATE TABLE `shop` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '',
  `shopcode` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> insert into shop(name,shopcode) values('a',10),('b',10),('c',10),('d',10),('e',10),('f',10),('g',10),('h',10),('i',10),('j',10);
Query OK, 10 rows affected (0.00 sec)
Records: 10  Duplicates: 0  Warnings: 0

mysql> select * from shop;
+----+------+----------+
| id | name | shopcode |
+----+------+----------+
|  1 | a    |       10 |
|  2 | b    |       10 |
|  3 | c    |       10 |
|  4 | d    |       10 |
|  5 | e    |       10 |
|  6 | f    |       10 |
|  7 | g    |       10 |
|  8 | h    |       10 |
|  9 | i    |       10 |
| 10 | j    |       10 |
+----+------+----------+
10 rows in set (0.00 sec)

mysql> select * from shop order by shopcode limit 0,2;
+----+------+----------+
| id | name | shopcode |
+----+------+----------+
| 10 | j    |       10 |
|  2 | b    |       10 |
+----+------+----------+
2 rows in set (0.09 sec)
mysql> select * from shop order by shopcode limit 2,2;
+----+------+----------+
| id | name | shopcode |
+----+------+----------+
|  3 | c    |       10 |
|  4 | d    |       10 |
+----+------+----------+
2 rows in set (0.00 sec)

mysql> select * from shop order by shopcode limit 4,2;
+----+------+----------+
| id | name | shopcode |
+----+------+----------+
|  5 | e    |       10 |
|  6 | f    |       10 |
+----+------+----------+
2 rows in set (0.00 sec)

mysql> select * from shop order by shopcode limit 6,2;
+----+------+----------+
| id | name | shopcode |
+----+------+----------+
|  7 | g    |       10 |
|  8 | h    |       10 |
+----+------+----------+
2 rows in set (0.00 sec)

mysql> select * from shop order by shopcode limit 8,2;
+----+------+----------+
| id | name | shopcode |
+----+------+----------+
|  2 | b    |       10 |
| 10 | j    |       10 |
+----+------+----------+
2 rows in set (0.00 sec)
```
我们发现 limit 0,2和limit 8,2的结果是一样的，看来原先对order by的理解有问题， 原来以为是先对shopcode整体排好序，再取对应分页的数据，查了下官方文档：

```
If you combine LIMIT row_count with ORDER BY, MySQL stops sorting as soon as it has found the first row_count rows of the sorted result, rather than sorting the entire result. 
If ordering is done by using an index, this is very fast. 
If a filesort must be done, all rows that match the query without the LIMIT clause are selected, and most or all of them are sorted, before the first row_count are found. 
After the initial rows have been found, MySQL does not sort any remainder of the result set.
```
上面官方文档里面有提到如果你将Limit row_count与order by混用，mysql会找到排序的row_count行后立马返回，而不是排序整个查询结果再返回。

为什么分页会不准呢？

```
If multiple rows have identical values in the ORDER BY columns, the server is free to return those rows in any order, and may do so differently depending on the overall execution plan.
In other words, the sort order of those rows is nondeterministic with respect to the nonordered columns.
```
如果order by的字段有多个行都有相同的值，mysql是会随机的顺序返回查询结果的，具体依赖对应的执行计划。

那这种情况应该怎么解决呢？

```
If it is important to ensure the same row order with and without LIMIT, include additional columns in the ORDER BY clause to make the order deterministic. 
For example, if id values are unique, you can make rows for a given category value appear in id order by sorting like this:
```
如果想在Limit存在或不存在的情况下，都保证排序结果相同，可以额外加一个排序条件。例如id字段是唯一的，可以考虑在排序字段中额外加个id排序去确保顺序稳定。

如果SQL改成 select * from shop order by shopcode,id limit M,N。遍历去取数据妥妥没问题了。

官方文档：https://dev.mysql.com/doc/refman/5.6/en/limit-optimization.html
