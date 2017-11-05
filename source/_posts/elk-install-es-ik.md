---
title: 如何安装ES的IK插件
date: 2017-11-05 14:24:08
tags:
- ELK
- IK插件
categories: ELK
---
本文主要介绍如何安装ES的IK插件，IK插件主要做分词用的，尤其对中文分词支持友好。

ES日志系统线上版本是5.5.1，IK地址：https://github.com/medcl/elasticsearch-analysis-ik

1.以ES的用户为每个节点安装插件

5.5.1以上版本官方已经支持命令安装，5.5.1以下版本需要mvn打包安装（如果觉得mvn打包麻烦，可以在plugins目录下创建ik目录，下载ZIP包，解压到该文件夹中,赋予正确权限也能行）

```
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.1/elasticsearch-analysis-ik-5.5.1.zip
```

2.先暂停集群的shard自动均衡

```
curl -XPUT http://YOURHOST:9200/_cluster/settings -d'
{
"transient" : {
"cluster.routing.allocation.enable" : "none"
}
}'
```
3.shutdown你要重启的节点

4.重启该节点，并确认该节点重新加入到了集群中

5.重启启动集群的shard均衡

```
curl -XPUT http://YOURHOST:9200/_cluster/settings -d'
{
"transient" : {
"cluster.routing.allocation.enable" : "all"
}
}'
```
6.重复2-5步，重启其它节点。

