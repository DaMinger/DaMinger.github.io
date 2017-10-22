---
title: Inception_OSC测试
date: 2017-10-22 19:17:06
tags:
- Inception
- 表变更平台
categories: MySQLTools
---
这篇文章主要介绍Inception对OSC工具的支持，如果对OSC工具不熟悉，可以自行先Google下

我这里配置的OSC配置文件

```
######inception的osc设置######
#指定pt-online-schema-change脚本的位置
inception_osc_bin_dir=/usr/bin/pt-online-schema-change
#对应OSC参数--recursion-method
inception_osc_recursion_method=processlist
#对应OSC参数--[no]drop-old-table
inception_osc_drop_old_table=1
#对应OSC参数--[no]check-replication-filters
inception_osc_check_replication_filters=1
#对应OSC参数--check-interval
inception_osc_check_interval=5
#对应OSC参数--max-lag
inception_osc_max_lag=1
#对应OSC参数--chunk-time
inception_osc_chunk_time=0.05
#对应OSC参数--max-load中的Threads_connected部分
inception_osc_max_thread_connected=1300
#对应OSC参数--max-load中的Threads_running部分
inception_osc_max_thread_running=64
#这个参数实际上是一个OSC的开关，如果设置为0，则全部ALTER语句都走OSC，如果设置为非0，则当这个表占用空间大小大于这个值时才使用OSC方式。单位为M，
inception_osc_min_table_size=16
#一个全局的OSC开关，默认是打开的
inception_osc_on=1
#无法支持--set-vars --tries
```

测试过程
生成一张表，数据量100W

```
sysbench  /usr/share/sysbench/oltp_read_write.lua --mysql-host=xxx --mysql-port=xxx --mysql-db=test --mysql-user=admin --mysql-password=xxx --tables=1 --table-size=1000000  --report-interval=10 --threads=10 --time=300 --db-driver=mysql prepare
```

你在python脚本中，改下参数--enable-check，先进行审核，得到改表语句的一个hash值，方便后面查看改表进度

!["inception_osc"](/images/inception_osc_01.png)

查看改表进度
inception get osc_percent '*0521D6C1E73C8F3B82809CF9D5D43D1D68095674';或者使用inception get osc processlist;

!["inception_osc"](/images/inception_osc_02.png)

终止改表
inception stop alter '*0521D6C1E73C8F3B82809CF9D5D43D1D68095674';

!["inception_osc"](/images/inception_osc_03.png)

再次改表，如果发现有触发器没有删干净，会报错退出

!["inception_osc"](/images/inception_osc_04.png)

特殊说明

- 如果你kill掉了python脚本，OSC工具在后台依旧在执行，需要手动kill掉，清理触发器和临时表。
