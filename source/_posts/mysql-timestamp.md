---
title: MySQL time_stamp字段基本用法
date: 2017-11-18 18:33:35
tags:
- time_stamp
- MySQL
categories: MySQL
---

MySQL版本mysql5.6.36,测试了解time_stamp的基本用法

1.timestamp类型允许一个表里可以有多列字段拥有自动插入时间和自动更新时间,更新其他列的时候，带有ON UPDATE CURRENT_TIMESTAMP的字段都会更新时间

```
mysql> CREATE TABLE time_test (
    ->    id int(11),
    ->    time_1 timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    ->    time_2 timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
    -> ) ENGINE=InnoDB  DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.31 sec)

mysql> show create table time_test;
+-----------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table     | Create Table                                                                                                                                                                                                                                                          |
+-----------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| time_test | CREATE TABLE `time_test` (
  `id` int(11) DEFAULT NULL,
  `time_1` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `time_2` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+-----------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)


mysql> insert time_test(id) values (1);
Query OK, 1 row affected (0.00 sec)

mysql> select * from time_test;
+------+---------------------+---------------------+
| id   | time_1              | time_2              |
+------+---------------------+---------------------+
|    1 | 2017-11-14 17:44:49 | 2017-11-14 17:44:49 |
+------+---------------------+---------------------+
1 row in set (0.01 sec)

mysql> update time_test set id =2 where id=1;
Query OK, 1 row affected (0.05 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from time_test;
+------+---------------------+---------------------+
| id   | time_1              | time_2              |
+------+---------------------+---------------------+
|    2 | 2017-11-14 17:45:20 | 2017-11-14 17:45:20 |
+------+---------------------+---------------------+
1 row in set (0.00 sec)

```


2.允许一列字段有自动插入时间属性，另一列字段只有自动更新时间属性，但是会默认赋予DEFAULT '0000-00-00 00:00:00'，更新其他列的时候，带有自动更新时间属性的字段会被更新

```
mysql> CREATE TABLE time_test_3 (
    ->     id int(11),
    ->     time_1 timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    ->     time_2 timestamp NOT NULL ON UPDATE CURRENT_TIMESTAMP
    ->  ) ENGINE=InnoDB  DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.08 sec)

mysql> show create table time_test_3;
+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table       | Create Table                                                                                                                                                                                                                                    |
+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| time_test_3 | CREATE TABLE `time_test_3` (
  `id` int(11) DEFAULT NULL,
  `time_1` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `time_2` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00' ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.01 sec)


mysql> insert time_test_3(id) values(1),(2);
Query OK, 2 rows affected (0.01 sec)
Records: 2  Duplicates: 0  Warnings: 0

mysql> select * from  time_test_3;
+------+---------------------+---------------------+
| id   | time_1              | time_2              |
+------+---------------------+---------------------+
|    1 | 2017-11-14 17:53:43 | 0000-00-00 00:00:00 |
|    2 | 2017-11-14 17:53:43 | 0000-00-00 00:00:00 |
+------+---------------------+---------------------+
2 rows in set (0.00 sec)

mysql> update time_test_3 set id=3 where id =2;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from  time_test_3;
+------+---------------------+---------------------+
| id   | time_1              | time_2              |
+------+---------------------+---------------------+
|    1 | 2017-11-14 17:53:43 | 0000-00-00 00:00:00 |
|    3 | 2017-11-14 17:53:43 | 2017-11-14 17:54:44 |
+------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

3.为多个字段指定timestamp字段，但不显示指明属性，会为第一个timestamp字段设置属性DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP，剩余timestamp字段设置 DEFAULT '0000-00-00 00:00:00'属性

```

mysql> CREATE TABLE `time_test2`(id int,time_1 timestamp,time_2 timestamp);
Query OK, 0 rows affected (0.07 sec)

mysql> show create table time_test2;
+------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table      | Create Table                                                                                                                                                                                                                                                                 |
+------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| time_test2 | CREATE TABLE `time_test2` (
  `id` int(11) DEFAULT NULL,
  `time_1` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `time_2` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci |
+------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.01 sec)

```
