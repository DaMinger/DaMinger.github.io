---
title: Inception_备份测试
date: 2017-10-25 20:47:23
tags:
- Inception
- 表变更平台
categories: MySQLTools
---

Inception的亮点是支持语句的备份，方便回滚，这篇文章主要测试它的备份功能

# 备份参数

```
inception_remote_backup_host //远程备份库的host
inception_remote_backup_port //远程备份库的port
inception_remote_system_user //远程备份库的一个用户
inception_remote_system_password //上面用户的密码
```
这些参数可以直接在命令行中指定，也可以用--defaults-file指定一个my.cnf文件，一经指定，再不能更改，除非将服务器关闭，修改之后再次启动。

# 注意项

- Inception是默认开备份功能的
- 执行时被影响的表如果没有主键的话，就不会做备份了，这样更简单并且备份时间及数据都会少一点，不然回滚语句的WHERE条件就会将所有列写进去，这样会影响性能且没有太大意义，所以在WHERE条件中，只需要主键即可
- Inception备份用户必须要具备下面的权限才能成功备份的，包括：CREATE、INSERT。CREATE权限用于创建表或者库的，INSERT权限用于插入备份数据的，其它的权限不需要
- 备份机器的库名组成是由线上机器的IP地址的点换成下划线，再加上端口号，再加上库名三部分，这三部分也是通过下划线连接起来的。
- 备份库中的表与线上表的列不同，它是用来存储所有对这个表修改的回滚语句的
    - rollback_statement text
    
     这个列存储的是针对当前这个表的某一行被修改后，生成的这行修改的回滚语句。因为binlog是ROW模式的，所以不管是什么语句，产生的回滚语句都是针对一行的，同时有可能一条语句的修改影响了多行，那么这里就会有多个回滚语句，但对应的是同一个 SQL 语句。对应关于通过下面的列来关联起来。
    
    - opid_time varchar(50)
    
      这个列存储的是的被执行的 SQL 语句在执行时的一个序列号，这个序列号由三部分组成：
        + timestamp(int 值，是语句被执行的时间点)
        + 线上服务器执行时所产生的 thread_id
        + 当前这条语句在所有被执行的语句块中的一个序号
         产生结果类似下面的样子：1413347135_136_3，针对同一个语句影响多行的情况，那么所产生的多行数据中，这个列的值都是相同的，这样就可以找到一条语句对应的所有被影响数据的回滚语句
         
# 测试

## 表带有自增主键

### 表结构和数据

!["inception_backup"](/images/inception_bakup_01.png)

### insert

插入一行数据

insert into test(col2,col3,col4) values (1,1,1); 下图这个sequence 非常重要

!["inception_backup"](/images/inception_bakup_02.png)

查看回滚语句

!["inception_backup"](/images/inception_bakup_03.png)

插入多行数据

insert into test(col2,col3,col4) values (1,1,1),(2,2,2),(3,3,3),(4,4,4),(5,5,5)

查看回滚语句

!["inception_backup"](/images/inception_bakup_04.png)

### delete

删除多条记录

delete from test where col2=2 and col3=1 and col4=6;

查看回滚语句

!["inception_backup"](/images/inception_bakup_05.png)

### update

更新多条记录

update test set col4=8 where col4=6

查看回滚语句

!["inception_backup"](/images/inception_bakup_06.png)


### alter
也支持alter的回滚，但是对于备份来说意义不大，测试两种

alter table test add col5 int not null default 10 comment "test";

回滚语句

!["inception_backup"](/images/inception_bakup_07.png)

alter table test drop col5;

回滚语句

!["inception_backup"](/images/inception_bakup_08.png)

## 表带有主键，但不自增
### 表结构

!["inception_backup"](/images/inception_bakup_09.png)

### insert
插入一行数据

insert into test5(col2,col3,col4) values(1,1,1),注意下面warning，没有指定主键，但是还是备份和执行成功了。默认id为0

!["inception_backup"](/images/inception_bakup_10.png)

回滚语句

!["inception_backup"](/images/inception_bakup_11.png)

插入多行数据

insert into test5(id,col2,col3,col4) values(1,1,2,3),(2,1,2,3),(3,1,2,3)

回滚语句

!["inception_backup"](/images/inception_bakup_12.png)

### delete

delete from test5 where id>=7

回滚语句

!["inception_backup"](/images/inception_bakup_13.png)

### update

update test5 set col2=2 where id not in (10)

回滚语句

!["inception_backup"](/images/inception_bakup_14.png)
 
### alter

和上面一致回滚结果一致

## 表没有主键
### 表结构

!["inception_backup"](/images/inception_bakup_15.png)

### insert
insert test6(id,col2,col3,col4) values(1,1,1,1),(1,2,1,1),(1,1,1,4),(1,1,3,1);
尽管提示备份成功了，但实际没有备份

!["inception_backup"](/images/inception_bakup_16.png)

没有备份

!["inception_backup"](/images/inception_bakup_17.png)

### update

update test6 set col3=6 where id=1

没有备份

!["inception_backup"](/images/inception_bakup_18.png)

### delete
delete from test6  where id=1

没有备份

!["inception_backup"](/images/inception_bakup_19.png)

## 表带有联合主键
### 创建联合主键表

!["inception_backup"](/images/inception_bakup_20.png)

!["inception_backup"](/images/inception_bakup_21.png)

### insert
insert into test (col2,col3,col4) values (1,1,1),(2,2,2),(3,3,3),(2,3,4);

回滚语句

!["inception_backup"](/images/inception_bakup_22.png)

### update
update test set col4=4 where id in (2,3,4);

回滚语句

!["inception_backup"](/images/inception_bakup_23.png)

### delete
delete from test  where id in (2,3,4);

回滚语句

!["inception_backup"](/images/inception_bakup_24.png)

