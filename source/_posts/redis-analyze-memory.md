---
title: Redis_分析集群中key的内存情况
date: 2017-11-15 22:11:32
tags:
- Redis
categories: Redis
---

Redis集群内存暴增，需要下集群中哪类key占用内存比较多，简述下我的思路和操作步骤

1.去找个slave,bgsave一下，生成rdb文件

2.使用rdb工具分析rdb文件生成csv文件

    rdb -c memory redis6379.rdb --bytes 0 -f memory.csv

    查看下csv文件
    cat memory.csv   
    database,type,key,size_in_bytes,encoding,num_elements,len_largest_element
    0,string,ops.073a7ae3d0fec21cd13c7a34d497cb70,304,string,210,210
    0,string,test12,48,string,8,8
    0,string,ops.5c8b7a27fd1597ef3202e9e4906cb630,304,string,194,194
    0,hash,bach-price-dev.2ssku_inventory_price_100016002,5651,ziplist,323,9
    0,set,monitor_device_list,83588,hashtable,920,37
    0,string,ops.e58ccdb5730ae3dbdbcb7103747b228b,304,string,194,194
    0,string,test4,48,string,8,8
    0,string,ops.8bf1f83967a11fffb7ed55ac1f42fa10,304,string,213,213
    0,string,test17,48,string,8,8
    
    把第一行数据表头去了，方便导入MySQL分析

3.创建MySQL统计表

    创建统计表
    create table memory_count(
    id int primary key auto_increment,
    db int not null default 0 ,
    key_type varchar(20) not null,
    key_name varchar(200) not null,
    key_size_in_bytes bigint not null,
    key_encoding varchar(20) not null,
    key_num_elements bigint not null,
    key_len_largest_element  bigint not null);

```
show create table memory_count;
memory_count | CREATE TABLE `memory_count` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `db` int(11) NOT NULL DEFAULT '0',
  `key_type` varchar(20) COLLATE utf8mb4_unicode_ci NOT NULL,
  `key_name` varchar(200) COLLATE utf8mb4_unicode_ci NOT NULL,
  `key_size_in_bytes` bigint(20) NOT NULL,
  `key_encoding` varchar(20) COLLATE utf8mb4_unicode_ci NOT NULL,
  `key_num_elements` bigint(20) NOT NULL,
  `key_len_largest_element` bigint(20) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci

```

4.导入到MySQL

```
把文件移动到这个变量路径下
SHOW VARIABLES LIKE "secure_file_priv"; 

load语句
LOAD DATA INFILE '/var/lib/mysql-files/memory.csv' 
INTO TABLE  memory_count
FIELDS TERMINATED BY ',' 
OPTIONALLY ENCLOSED BY '"' 
ESCAPED BY '"' 
LINES TERMINATED BY '\n'
(db,key_type,key_name,key_size_in_bytes,key_encoding,key_num_elements,key_len_largest_element)

到了MySQL里面，可以随便定制SQL统计语句了
select left(key_name, 20),sum(key_size_in_bytes) from memory_count group by left(key_name, 20) order by sum(key_size_in_bytes) desc;
```
