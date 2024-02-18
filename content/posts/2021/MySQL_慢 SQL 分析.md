---
title: 【MySQL】慢 SQL 分析
date: 2021-08-10 12:30:00
tags: MySQL
---

## 慢查询日志分析
### 设置慢查询
可以通过修改命令设置：
+ 设置开启：SET GLOBAL slow_query_log = 1; #默认未开启，开启会影响性能，mysql重启会失效
+ 查看是否开启：SHOW VARIABLES LIKE '%slow_query_log%';
+ 设置阈值：SET GLOBAL long_query_time=3;
+ 查看阈值：SHOW 【GLOBAL】 VARIABLES LIKE 'long_query_time%'; #重连或新开一个会话才能看到修改值

也可以通过修改配置文件设置，配置文件 my.conf 会一直生效，在[mysqld]下配置：
```
    [mysqld]
    slow_query_log = 1;　　#开启
    slow_query_log_file=/var/lib/mysql/atguigu-slow.log　　　#慢日志地址，缺省文件名host_name-slow.log
    long_query_time=3;　　  #运行时间超过该值的SQL会被记录，默认值>10
    log_output=FILE　　　　　　　　　　　
```

### 获取慢 SQL 信息
查看慢查询日志记录数：SHOW GLOBAL STATUS LIKE '%Slow_queries%';

模拟语句：SELECT SLEEP(4);

查看慢查询日志：
```
    $ cat /usr/local/var/mysql/OreChoudeMacBook-Pro-slow.log
    /usr/local/Cellar/mysql/8.0.25_1/bin/mysqld, Version: 8.0.25 (Homebrew). started with:
    Tcp port: 3306  Unix socket: /tmp/mysql.sock
    Time                 Id Command    Argument
    # Time: 2021-08-10T06:29:53.513752Z
    # User@Host: root[root] @ localhost [127.0.0.1]  Id:    11
    # Query_time: 4.003605  Lock_time: 0.000000 Rows_sent: 1  Rows_examined: 1
    use test;
    SET timestamp=1628576989;
    select sleep(4);
```

### 使用 mysqldumpslow 分析
使用样例：
+ mysqldumpslow -s r -t 10 /usr/local/var/mysql/OreChoudeMacBook-Pro-slow.log     #得到返回记录集最多的10个SQL
+ mysqldumpslow -s c -t 10 /usr/local/var/mysql/OreChoudeMacBook-Pro-slow.log     #得到访问次数最多的10个SQL
+ mysqldumpslow -s t -t 10 -g "LEFT JOIN" /usr/local/var/mysql/OreChoudeMacBook-Pro-slow.log   #得到按照时间排序的前10条里面含有左连接的查询语句
+ mysqldumpslow -s r -t 10 /usr/local/var/mysql/OreChoudeMacBook-Pro-slow.log | more      #结合| more使用，防止爆屏情况

## Explain 分析


## Show Profile 分析
Show Profile 能够获取比 Explain 更为详细的信息，能够分析当前会话中语句执行时的资源消耗，获取 SQL 在整个生命周期的时间。

### 开启 Profile
```
    开启：set profiling = on;
    查看：SHOW VARIABLES LIKE 'profiling%';
```
开启后 MySQL 后台会保存最近 15 次的结果。

### 查看 Profile
使用 SHOW PROFILES 可以查看最近的 15 次结果。

### 查看具体的 Profile
通过 Query_ID 可以得到具体 SQL 从连接 - 服务 - 引擎 - 存储四层结构完整生命周期的耗时。

使用命令：SHOW PROFILE CPU, BLOCK IO FOR Query_ID

可用参数type:
+ ALL  　　#显示所有的开销信息
+ BLOCK IO　　#显示块IO相关开销
+ CONTEXT SWITCHES　　#上下文切换相关开销
+ CPU     #显示CPU相关开销信息
+ IPC     #显示发送和接收相关开销信息
+ MEMORY　#显示内存相关开销信息
+ PAGE FAULTS　　#显示页面错误相关开销信息
+ SOURCE　　#显示和Source_function，Source_file，Source_line相关的开销信息
+ SWAPS　　#显示交换次数相关开销的信息

如果出现以下几个状态则 SQL 需要重点分析：
+ converting HEAP to MyISAM  　　#查询结果太大，内存不够用了，在往磁盘上搬
+ Creating tmp table         　　#创建了临时表，回先把数据拷贝到临时表，用完后再删除临时表
+ Copying to tmp table on disk 　#把内存中临时表复制到磁盘，危险！！！
+ locked
