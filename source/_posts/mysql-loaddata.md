---
title: MySQL的Load data的权限误区
date: 2017-11-01 21:41:32
tags:
- MySQL
- Load data
categories: MySQL
---

今天刚到公司，手机接到主从延时的报警，登录线上机器，发现是一条load data语句导致的。

```
load data LOW_PRIORITY local infile '/data/tmp/job-temp/app_ordering_system_maiev_check_reference_data_20171031' replace into table order_check fields terminated by '\t' ( order_date
,batch_code
,transport_num
,shop_code
,shop_name
,mid_classify_code
,mid_classify_name
,sku_main_code
,sku_code
,sku_name
,sku_type
,whether_new
,whether_popular
,sale_level
,booking_spec
,yesterday_stock
,refer_abandon_amount
,sale_amount_day
,order_amount_day
,expiration
,whether_yummy
,purchasing_unit_price
,shop_start_date
,shop_state
,shop_business_state
,picking_type
,booking_qty
,shop_type
)

```

延迟原因很清晰明了，但是我看了下这个库不是数据组的业务，按理说不应该有load data权限，我找到账号，show grants一下，确实只有增删改查权限，找对应RD也确认了下，确实刚才使用这个账号执行了这个语句，线上数据库版本是MySQL Community 5.6.36，我在测试环境上也能够复现这个情景。原先我以为load data权限怎么也得有file权限，看来存在误区。

上网搜了搜，看到这段解释解决了我的疑问

```
- load data infile into table table_name ; 
  执行命令的用户有权限访问的文件，然后load到DB中,并且需要file权限。
  
- load data local infile  into table table_name ;
  只要是客户端用户有权限访问的文件，都可以上传到服务端，然后load到DB中，这样是特别危险的。
  注意：此时的load data不需要file权限！

- select * from t into outfile 'filename';
  先看运行mysqld的用户有没有对存放文件的目录有写权限，再看执行命令的用户有没有file权限.
```

解决办法，针对load data local infile,扫描线上MySQL参数local_infile的值，设置成OFF即可。

反思
    
- 对看有疑惑的问题，一定要先记录下来，想办法去解决，不断走出自己的知识误区。
