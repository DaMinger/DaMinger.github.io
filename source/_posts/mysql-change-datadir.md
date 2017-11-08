---
title: MySQL更改数据目录
date: 2017-11-08 22:03:29
tags:
- MySQL
categories: MySQL
---
今天工作中收到数据库报警，报磁盘空间90%多了，我一看4T的盘呢，怎么会报磁盘空间报警呢？再仔细一看，之前的运维人员把MySQL部署在了根目录上，隐形的坑，临时清理了日志，联系业务，需要把数据库迁移到数据盘上。

步骤

路由掉从库流量，先从库后主库。
 
1.关闭监控

2.备份原有的MySQL配置文件

3.关闭mysql

mysqladmin -uXXXX -p'XXXX' -S XXXX.sock shutdown

4.更改配置文件，对datadir、slow_query_log_file、log-error的三个参数更改目录

/data/${mysqldir}=>/data1/${mysqldir} 

5.mv MySQL目录到数据盘/data1/

mv /data/${mysqldir} /data1/${mysqldir}

6.赋权

chown -R mysql:mysql /data1/${mysqldir}

7.启动

/usr/bin/mysqld_safe --defaults-file=/etc/XXXX.cnf >/dev/null & 

8.检查MySQL状态，打开报警

9.监控模板微调，磁盘报警需要改成监控的/data1磁盘情况

如果减少copy时间，注意清理slowlog和binlog
