---
title: Redis不可写问题追踪
date: 2017-12-24 01:20:14
tags:
- Redis
- casestudy
categories: Redis
---
线上某个Redis集群业务反馈写入不进去，读取正常，奇怪，该集群最近没有做过任何变更，内存使用率也没有满，简单连上发现确实只能读，不可写，于是查看redis log，发现一段疑问信息

```
Can't save in background: fork: Cannot allocate memory
```
调查发现

    Redis做RDB持久化需要fork出一个子进程，向系统申请内存失败，
    由于stop-writes-on-bgsave-error配置的是yes，
    即表示RDB持久化失败会拒绝业务写入操作，从而使得业务写入失败。
    
将默认配置stop-writes-on-bgsave-error 设置为no,业务恢复正常。（这样只是当redis写硬盘快照出错时，可以让用户继续做更新操作，但是写硬盘仍然是失败的。）

彻底解决方式

```
编辑/etc/sysctl.conf添加
vm.overcommit_memory=1
执行sysctrl -p 使其生效

vm.overcommit_memory 这个参数又是干什么的呢？

Linux对大部分申请内存的请求都回复"yes"，以便能跑更多更大的程序。
因为申请内存后，并不会马上使用内存，将这些不会使用的空闲内存分配给其它程序使用，以提高内存利用率，这种技术叫做Overcommit。
一般情况下，当所有程序都不会用到自己申请的所有内存时，系统不会出问题，
但是如果程序随着运行，需要的内存越来越大，在自己申请的大小范围内，不断占用更多内存，直到超出物理内存，
当linux发现内存不足时，会发生OOM killer(OOM=out-of-memory)。
它会选择杀死一些进程(用户态进程，不是内核线程，哪些占用内存越多，运行时间越短的进程越有可能被杀掉)，以便释放内存。
```
在什么条件下，linux会发现内存不足呢？

```
在Linux下有个CommitLimit 用于限制系统应用使用的内存资源:
grep -i commit /proc/meminfo
其中：
CommitLimit是一个内存分配上限，
Committed_AS是已经分配的内存大小。
虚拟内存算法：
CommitLimit = 物理内存 * overcommit_ratio(/proc/sys/vm/overcmmit_ratio，默认50，即50%) + swap大小
它是由内核参数overcommit_ratio的控制的,当我们的应用申请内存的时候，当出现以下情况：
应用程序要申请的内存 + 系统已经分配的内存（也就是Committed_AS）> CommitLimit
这时候就是内存不足，到了这里，操作系统要怎么办，就要祭出我们的主角“overcommit_memory”参数了（/proc/sys/vm/overcommit_memory);

vm.overcommit_memory = 1 允许overcommit
直接放行，系统在为应用进程分配虚拟地址空间时，完全不进行限制，
这种情况下，避免了fork可能产生的失败，但由于malloc是先分配虚拟地址空间，而后通过异常陷入内核分配真正的物理内存，
在内存不足的情况下，这相当于完全屏蔽了应用进程对系统内存状态的感知，即malloc总是能成功，
一旦内存不足，会引起系统OOM杀进程，应用程序对于这种后果是无法预测的。
```

为什么RDB持久化之前没有问题，今天突然才出现这个问题？

由于redis服务内存使用最近突然增长，因此之前没有出现RDB失败的情况，通过分析RDB文件，某类KEY最近激增导致。

改进点

1. 这是旧的Redis集群，改成aof方式
2. 将所有redis配置stop-writes-on-bgsave-error调整为no
3. 检查线上redis服务器vm.overcommit_memory配置
4. 增加redis不可写监控
