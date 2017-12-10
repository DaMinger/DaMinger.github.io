---
title: MySQL字符集备注
date: 2017-12-10 13:18:20
tags:
- MySQL
categories: MySQL
---

字符集知识备注

```
查看数据库支持的字符集和校对规则
show charset;或者 show character set;
show collation;

查看数据库当前的字符集和校对规则
show variables like '%character%';
show variables like '%collation%';

查看表的校对规则 
show table status from db_name like '%table_name%' ;  

查看列的校对规则
show full columns from table_name;

更改表级别的字符集和字符校验规则
ALTER TABLE  tbl_name CONVERT TO CHARACTER SET charset_name COLLATE collation_name;

为具体的某个字段指定其字符集和校验规则
ALTER TABLE tbl_name MODIFY col_name {CHAR | VARCHAR | TEXT} (col_length)
    [CHARACTER SET charset_name] [COLLATE collation_name]
例子
ALTER TABLE customer_1 MODIFY name varchar(160) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin  NOT NULL DEFAULT '';
```

注意

```
字段属性的字符校对规则不一致，连表的时候有可能会报错，因为线上有些RD 会这样建表，
CREATE TABLE `customer_1` (
  `id` int(11) NOT NULL,
  `name` varchar(160) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE utf8mb4_general_ci

表的字符校验规则是utf8mb4_general_ci，某些字段校验规则是utf8mb4_bin

连表的时候有可能报错
[HY000][1267] Illegal mix of collations (utf8mb4_unicode_ci,IMPLICIT) and (utf8mb4_general_ci,IMPLICIT) for operation '='

另外如果对字符集校验规则不同的字段还建立了索引，连表的时候，这个索引可能走不到。
```
