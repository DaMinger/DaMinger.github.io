---
title: replace操作导致主从auto_increment不一致
date: 2017-12-03 21:28:06
tags:
- MySQL
- replace
categories: MySQL
---
周二做了个主从切换，然后半夜被RD打电话叫起来，说插入主键报重复主键错误，我登录到机器上一看，表中的最大主键ID确实大于表的auto_increment值，临时处理了下，先恢复业务，那为什么会出现这种情况？第二天和RD沟通后，他们业务中有replace操作，我怀疑这个操作会导致主从auto_increment不一致。

试验：
构造表结构和数据

```
master:
mysql> CREATE TABLE `t` (
    ->   `id` int(11) NOT NULL AUTO_INCREMENT,
    ->   `age` int(11) DEFAULT NULL,
    ->   `msg` varchar(10) DEFAULT NULL,
    ->   PRIMARY KEY (`id`),
    ->   UNIQUE KEY `uniq_age` (`age`)
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.30 sec)

mysql> insert into t (age, msg) values (1,'aaa'),(2,'bbb'),(3,'ccc');
Query OK, 3 rows affected (0.01 sec)
Records: 3  Duplicates: 0  Warnings: 0

mysql> select * from t;
+----+------+------+
| id | age  | msg  |
+----+------+------+
|  1 |    1 | aaa  |
|  2 |    2 | bbb  |
|  3 |    3 | ccc  |
+----+------+------+
3 rows in set (0.00 sec)

mysql> show create table t;
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                                           |
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| t     | CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `age` int(11) DEFAULT NULL,
  `msg` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_age` (`age`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8 |
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

slave:
mysql> show create table t;
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                                           |
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| t     | CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `age` int(11) DEFAULT NULL,
  `msg` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_age` (`age`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8 |
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.03 sec)
```
进行replace操作

```
master:
mysql> replace into t (age, msg) values (1, '111');
Query OK, 2 rows affected (0.00 sec)

mysql> select * from t;
+----+------+------+
| id | age  | msg  |
+----+------+------+
|  2 |    2 | bbb  |
|  3 |    3 | ccc  |
|  4 |    1 | 111  |
+----+------+------+
3 rows in set (0.00 sec)

mysql> show create table t;
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                                           |
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| t     | CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `age` int(11) DEFAULT NULL,
  `msg` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_age` (`age`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8 |
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

```
我们看到replace操作，有2行被影响，auto_increment自动+1，网上查资料得知：对于已经存在的数据，replace操作相当于进行update操作，而update实际是delete+insert两个连续操作，insert会导致auto_incement+1,但是binlog只会产生一条update语句。

```
binlog记录：确实只有一条update
#171203 16:54:34 server id 114133006  end_log_pos 779912805     Table_map: `test`.`t` mapped to number 138
# at 779912805
#171203 16:54:34 server id 114133006  end_log_pos 779912863     Update_rows: table id 138 flags: STMT_END_F

BINLOG '
yrsjWhMOiM0GLAAAAGWGfC4AAIoAAAAAAAEABHRlc3QAAXQAAwMDDwIeAAY=
yrsjWh8OiM0GOgAAAJ+GfC4AAIoAAAAAAAEAAgAD///4AQAAAAEAAAADYWFh+AQAAAABAAAAAzEx
MQ==
'/*!*/;
### UPDATE `test`.`t`
### WHERE
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2=1 /* INT meta=0 nullable=1 is_null=0 */
###   @3='aaa' /* VARSTRING(30) meta=30 nullable=1 is_null=0 */
### SET
###   @1=4 /* INT meta=0 nullable=0 is_null=0 */
###   @2=1 /* INT meta=0 nullable=1 is_null=0 */
###   @3='111' /* VARSTRING(30) meta=30 nullable=1 is_null=0 */
# at 779912863
#171203 16:54:34 server id 114133006  end_log_pos 779912890     Xid = 9277304
COMMIT/*!*/;
```

```
slave:从库只会接受到update的binlog,auto_increment不会加+1
mysql> show create table t;
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                                           |
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| t     | CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `age` int(11) DEFAULT NULL,
  `msg` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_age` (`age`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8 |
+-------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```
至此主从auto_increment的不一致的原因已经定位到。

隐患点：
主从切换后，业务如有replace操作，可能造成新主库写入失败。

如何解决?

先了解下算法：MySQL对于auto_increment的算法是select max(id)+1 from t的方法来拿到auto_inrement的值。

淘宝的分享解释：

```
建表时可以指定 AUTO_INCREMENT值，不指定时默认为1，这个值表示当前自增列的起始值大小，
如果新插入的数据没有指定自增列的值，那么自增列的值即为这个起始值。
对于InnoDB表，这个值没有持久到文件中,而是存在内存中（dict_table_struct.autoinc）。
那么又问，既然这个值没有持久下来，为什么我们每次插入新的值后， show create table t1看到AUTO_INCREMENT值是跟随变化的。
其实show create table t1是直接从dict_table_struct.autoinc取得的（ha_innobase::update_create_info）。
```

解决办法

- 重启从库，所有表的auto_increment会修正。
- 插入新数据后，自增值自动追平，主库replace后如果有insert语句，这样会自动追平的。

延伸

- insert into .. on duplicate key update语句 碰到表中已经存在要replace的数据，也会导致主从auto_increment不一致
