---
title: ELK_ElasticSearch5.5.1版本快速部署
date: 2017-10-29 19:30:30
tags:
- ELK
- ElasticSearch
categories: ELK
---

进入新公司，接触了一段时间ELK组件，挺有意思的，记录下学习过程。这篇主要介绍下ElasticSearch的部署过程，版本是5.5.1，ElasticSearch社区非常活跃，2017年从5.1迭代到6.X，大家一定多参考ES的官方文档，非常详细，中文参考书籍《ELK Stack 权威指南》第二版 饶琛琳 著，这本书对于新手很有帮助。

# 背景
本文主要描述如何快速部署ES三节点，多节点步骤部署和本文一样，这里我规划的节点属性是默认的属性，每个节点即是master节点，也是data节点，大规模集群的时候，这个属性建议分开，后面我写文章讲下我所理解的ES架构。安装步骤中涉及的IP，我统一用YOURHOSTIP替代了。有疑问直接EMAIL我。

# 安装步骤
## 安装JDK

```
1. java  -version 
如果发现有openjdk 或者其他非官方版的Java，卸载掉
rpm -qa|grep java
rpm -e --nodeps  rpmpackage
 
2.下载java rpm包
ES官方推荐the Oracle JDK version 1.8.0_131，Oracle官方已经GA到1.8.0_144，下载最新的JDK即可
 
3.安装
# rpm -ivh jdk-8u144-linux-x64.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:jdk1.8.0_144-2000:1.8.0_144-fcs  ################################# [100%]
Unpacking JAR files...
    tools.jar...
    plugin.jar...
    javaws.jar...
    deploy.jar...
    rt.jar...
    jsse.jar...
    charsets.jar...
    localedata.jar...
# java -version
java version "1.8.0_144"
Java(TM) SE Runtime Environment (build 1.8.0_144-b01)
Java HotSpot(TM) 64-Bit Server VM (build 25.144-b01, mixed mode)
成功显示即可

```
## Linux 参数调整

官方文档：https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html

```
1.关闭selinux状态
/usr/sbin/sestatus -v   显示disable
如果不是disable状态，临时关闭
setenforce 0
永久关闭 (需要重启机器)
修改/etc/selinux/config 文件
将SELINUX=enforcing改为SELINUX=disabled
 
2.关闭iptables
systemctl status firewalld.service  显示disable
如果不是disable状态，临时关闭
systemctl stop firewalld.service #停止firewall
systemctl disable firewalld.service #禁止firewall开机启动
 
3.文件描述符
cat /etc/security/limits.conf 若是发现soft nofile和hard nofile 小于65536，设置一下
ulimit -n 65536
echo -ne "
* soft nofile 65536
* hard nofile 65536
" >>/etc/security/limits.conf
 
4.SSD机器修改IO调度算法为noop或者deadline
首先找到对应的盘，替换掉下面的sda
cat /sys/block/sda/queue/scheduler
noop deadline [cfq]（默认是cfq）
echo "noop" >/sys/block/sda/queue/scheduler
cat /sys/block/sda/queue/scheduler
[noop] deadline cfq (变为noop)
 
5.系统线程限制
echo -ne "
* soft nproc 2048
* hard nproc 4096
" >>/etc/security/limits.conf
 
6.Virtual memory
修改系统限制一个进程可以拥有的VMA(虚拟内存区域)的数量
sysctl -w vm.max_map_count=262144
sysctl -a | grep "vm.max_map_count"
若无参数请增加
vim /etc/sysctl.conf
vm.max_map_count = 262144
 
7.关闭swap分区
swapoff -a
设置降低交换分区的使用优先级
vim /etc/sysctl.conf
vm.swappiness = 0
```


## 安装部署ES

```
1.下载安装ES5.5.1
curl -L -O "https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.1.tar.gz"
tar -xvf elasticsearch-5.5.1.tar.gz
cp -r elasticsearch-5.5.1 /data/
mv elasticsearch-5.5.1 es01

2.创建es用户组及es用户
groupadd es
useradd es -g es
chown -R es:es es01

3.es启动前 jvm配置
vim /data/es01/config/jvm.options
-Xms16g
-Xmx16g
注：按需调整，配置物理机内存的50%，但不要超过31G。

4.es启动前 修改elasticsearch.yml
vim /data/es01/config/elasticsearch.yml

#集群名称 保持唯一
cluster.name: ESTEST
#节点名称
node.name: es_test01
#数据路径
path.data: /data/es01/esdata
#监听地址
network.host: 0.0.0.0
#防止脑裂 集群主节点数/2+1
discovery.zen.minimum_master_nodes: 2
#单播ES节点地址. 这里可以添加集群的任一一个或者多个节点
discovery.zen.ping.unicast.hosts: ["YOURHOSTIP"]

5.复制copy其他节点并编辑
# cp -r es01 es02
# cp -r es01 es03
# chown -R es:es es02/
# chown -R es:es es03/

只需要更改es02的elasticsearch.yml
#节点名称
#node.name: es_test02
#数据路径
path.data: /data/es02/esdata

只需要更改es03的elasticsearch.yml
#节点名称
#node.name: es_test03
#数据路径
path.data: /data/es03/esdata

6.用es用户以此启动
./es01/bin/elasticsearch -d
./es02/bin/elasticsearch -d
./es03/bin/elasticsearch -d
查看/data/es01/logs、/data/es02/logs、/data/es03/logs的ESTEST.log 查看启动是否成功[2017-08-15T21:40:03,881][INFO ][o.e.n.Node ] [ESTEST01] starting ...
[2017-08-15T21:40:04,036][INFO ][o.e.t.TransportService ] [ESTEST01] publish_address {YOURHOSTIP:9300}, bound_addresses {0.0.0.0:9300}
[2017-08-15T21:40:04,047][INFO ][o.e.b.BootstrapChecks ] [ESTEST01] bound or publishing to a non-loopback or non-link-local address, enforcing bootstrap checks
[2017-08-15T21:40:07,113][WARN ][o.e.d.z.ZenDiscovery ] [ESTEST01] not enough master nodes discovered during pinging (found [[Candidate{node={ESTEST01}{YpiDg9KHSuW0POdZpQ1Uqg}{ogEGQGNTRDKCiNubSVPbAQ}{YOURHOSTIP}{YOURHOSTIP:9300}, clusterStateVersion=-1}]], but needed [2]), pinging again
[2017-08-15T21:40:11,239][INFO ][o.e.c.s.ClusterService ] [ESTEST01] detected_master {ESTEST02}{Py8YVQGZQf-M1PApgWJ9eg}{RohcDroPS5mpCLAJ01gR9w}{YOURHOSTIP}{YOURHOSTIP:9301}, added {{ESTEST02}{Py8YVQGZQf-M1PApgWJ9eg}{RohcDroPS5mpCLAJ01gR9w}{YOURHOSTIP}{YOURHOSTIP:9301},}, reason: zen-disco-receive(from master [master {ESTEST02}{Py8YVQGZQf-M1PApgWJ9eg}{RohcDroPS5mpCLAJ01gR9w}{YOURHOSTIP}{YOURHOSTIP:9301} committed version [1]])
[2017-08-15T21:40:11,278][INFO ][o.e.h.n.Netty4HttpServerTransport] [ESTEST01] publish_address {YOURHOSTIP:9200}, bound_addresses {0.0.0.0:9200}
[2017-08-15T21:40:11,278][INFO ][o.e.n.Node ] [ESTEST01] started
观察到started就ok了

7.查看状态

查看集群状态
curl -XGET 'http://localhost:9200/_cat/health?v'
看到status 为Green就是健康的 
epoch      timestamp cluster status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1503041882 15:38:02  ESTEST  green           3         3      0   0    0    0        0             0                  -                100.0%

查看节点状态
curl -XGET 'http://localhost:9200/_cat/nodes?v'
ip           heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
YOURHOSTIP（你机器的IP）           37          83   7    0.31    0.63     0.80 mdi       -      es_test02
YOURHOSTIP（你机器的IP）          48          83   7    0.31    0.63     0.80 mdi       -      es_test03
YOURHOSTIP（你机器的IP）           37          83   6    0.31    0.63     0.80 mdi       *      es_test01
```
