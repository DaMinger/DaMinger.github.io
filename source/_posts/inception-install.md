---
title: Inception_安装部署
date: 2017-10-19 00:18:07
tags:
- Inception
- 表变更平台
categories: MySQLTools
---
# 背景

公司现阶段还没有自助表变更平台，工作中一些DDL和DML还得手动搞，一是重复的事情反复做，时间消耗多，二是人工执行存在风险隐患，于是开始调研一下去哪儿的inception工具，如果可行，对它进行二次封装进行web化开发。

线上服务器Linux版本：Centos7

# 下载源码和安装相关依赖包

从Github上下载源码

    git clone https://github.com/mysql-inception/inception.git

安装必要的编译包

    yum install -y openssl-devel gcc-c++  ncurses-devel cmake

注意bison包需要编译安装，下载bison：http://ftp.gnu.org/gnu/bison/ 版本最好是2.6之前的，最新的可能会有问题
    
    wget http://ftp.gnu.org/gnu/bison/bison-2.6.tar.gz
    tar -zxvf bison-2.6.tar.gz
    cd bison-2.6
    ./configure
    make
    make install
    然后做个软连接
    cd /usr/bin
    ln -s /data/bison-2.6/src/bison
    查看bison版本号
    bison -V

# 编译安装inception

进入inception目录

    sh inception_build.sh debug [linux]
上述命令中debug为编译安装的目录，[linux]为当前操作系统平台
那么需要注意的是，每次如果出错之后，需要把编译目录删除掉，重新执行，不然会执行出错。
 
编辑配置文件inc.inf文件,这个配置文件在后面的文章详细讲解配置下

    [inception]
    general_log=1
    general_log_file=inception.log
    port=6669
    socket=/tmp/inc.socket
    character-set-client-handshake=0
    character-set-server=utf8
    inception_remote_system_password=YOURBACKPASSWORD
    inception_remote_system_user=admin
    inception_remote_backup_port=33006
    inception_remote_backup_host=YOURBACKUSER
    inception_support_charset=utf8mb4
    inception_enable_nullable=0
    inception_check_primary_key=1
    inception_check_column_comment=1
    inception_check_table_comment=1
    inception_osc_min_table_size=1
    inception_osc_bin_dir=/data/temp
    inception_osc_chunk_time=0.1
    inception_enable_blob_type=1
    inception_check_column_default_value=1

启动

    ./inception/debug/mysql/bin/Inception --defaults-file=inc.inf
测试

    mysql -uroot -h127.0.0.1 -P6669
    inception get variables;
正常输出就OK

# 简单使用

编写一个test.py,在test库里面创建一张表

    #!/usr/bin/python
    #-*- coding: utf-8 -*-
    import MySQLdb
    sql='/*--user=YOURUSER;--password=YOURPASSWORD;--host=YOURHOST;--port=33006;--execute=1;*/\
    inception_magic_start;\
    use test;\
    CREATE TABLE lqm_test(\
    id int unsigned not null auto_increment comment "aaaa",\
    username varchar(10) not null default 0 comment "xxx",\
    primary key(id))\
    engine = innodb default charset utf8mb4 comment "dddd";\
    inception_magic_commit;'
    try:
        conn=MySQLdb.connect(host='127.0.0.1',user='root',passwd='',db='',port=6669)
        cursor=conn.cursor()
        cursor.execute(sql)
        results = cursor.fetchall()
        column_name_max_size=max(len(i[0]) for i in cursor.description)
        row_num=0
        for result in results:
            row_num=row_num+1
            print '*'.ljust(27,'*'),row_num,'.row', '*'.ljust(27,'*')
            row = map(lambda x, y: (x,y), (i[0] for i in cursor.description), result)
            for each_column in row:
                if each_column[0] != 'errormessage':
                    print each_column[0].rjust(column_name_max_size),":",each_column[1]
                else:
                    print each_column[0].rjust(column_name_max_size),':',each_column[1].replace('\n','\n'.ljust(column_name_max_size+4))
        cursor.close()
        conn.close()
    except MySQLdb.Error,e:
        print "Mysql Error %d: %s" % (e.args[0], e.args[1])

python test.py,观察log

    # less inception.log
    171011 18:11:46     2 Query     set autocommit=0
                    2 Query     /*--user=XXX;--password=XXX;--host=XXX;--port=33006;--execute=1;*/inception_magic_start;use test;CREATE TABLE lqm_test(id int unsigned not null auto_increment comment "aaaa",username varchar(10) not null default 0 comment "xxx",primary key(id))engine = innodb default charset utf8mb4 comment "dddd";inception_magic_commit
检查表是否创建成功即可
