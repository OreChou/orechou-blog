---
title: 【DevOps】使用 Docker 搭建 MySQL 主从复制架构
date: 2023-06-28 18:30:00
tags: [DevOps,MySQL]
---

这篇文章介绍一下如何使用 Docker 搭建 MySQL 主从复制架构，文中的架构中包含一个主节点 master，一个从节点 slave。

# 创建 docker 网络
在 Docker 中，容器之间如果需要可以相互访问，那么你需要把这些容器添加到同一个 Docker 网络中。创建一个叫 mysql-net 的 Docker 网络，命令如下：
```shell
docker network create mysql-net
```

# 搭建 master 节点
## 配置文件
配置 master 节点的配置文件 my.conf，注意修改 server-id，这里我们将 server-id 设置为 1。MySQL 复制过程中，主服务器和从服务器需要具有不同的 server id。配置文件内容如下：
```
[mysqld]
server-id=1
user=mysql
character-set-server=utf8
collation-server=utf8_unicode_ci
default_authentication_plugin=mysql_native_password
secure_file_priv=/var/lib/mysql
expire_logs_days=7
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
max_connections=1000

[client]
default-character-set=utf8

[mysql]
default-character-set=utf8
```

## 启动容器

在本地创建 master 节点的 logs、data、conf 挂载到 Docker 容器，启动容器的命令如下：
```
docker run --name mysql-master \
-v /Users/orechou/docker/mysql-cluster/master/logs:/var/log/mysql \
-v /Users/orechou/docker/mysql-cluster/master/data:/var/lib/mysql \
-v /Users/orechou/docker/mysql-cluster/master/conf:/etc/mysql \
--network=mysql-net \
-p 3306:3306 \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql/mysql-server:latest
```
注意我们将本机的 3306 端口映射到容器的 3306 端口，并将容器加入到 mysql-net 网络中。

## 配置 master

登录到 master 节点，使用 `docker exec -it mysql-master mysql -uroot -proot`  命令，然后进行相关配置。

### 创建一个账户
创建一个新的 MySQL 用户 'repl'，并授予它复制从服务器的权限。代码如下：
```sql
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'repl';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```
### 查看节点状态
查看主节点的状态，记录下 binlog 文件名和 position。代码如下：
```
mysql> SHOW MASTER STATUS;
+---------------+----------+--------------+------------------+-------------------+
| File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+---------------+----------+--------------+------------------+-------------------+
| binlog.000003 |     1741 |              |                  |                   |
+---------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

自此，master 节点就配置完成了。

# 搭建 slave 节点

## 配置文件
配置 slave 节点的配置文件 my.conf，注意修改 server-id，这里我们将 server-id 设置为 2。配置文件内容如下：
```
[mysqld]
server-id=2
user=mysql
character-set-server=utf8
collation-server=utf8_unicode_ci
default_authentication_plugin=mysql_native_password
secure_file_priv=/var/lib/mysql
expire_logs_days=7
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
max_connections=1000

[client]
default-character-set=utf8

[mysql]
default-character-set=utf8
```

## 启动容器
在本地创建 slave 节点的 logs、data、conf 挂载到 Docker 容器，启动容器的命令如下：
```
docker run --name mysql-slave \
-v /Users/orechou/docker/mysql-cluster/slave/logs:/var/log/mysql \
-v /Users/orechou/docker/mysql-cluster/slave/data:/var/lib/mysql \
-v /Users/orechou/docker/mysql-cluster/slave/conf:/etc/mysql \
--network=mysql-net \
-p 3307:3306 \
-e MYSQL_ROOT_PASSWORD=rolot \
-d mysql/mysql-server:latest
```
我们将本机的 3307 端口映射到容器的 3306 端口，并将容器加入到 mysql-net 网络中。

```
CHANGE MASTER TO MASTER_HOST='mysql-master', MASTER_USER='repl', MASTER_PASSWORD='repl', MASTER_LOG_FILE='binlog.000003', MASTER_LOG_POS=1741
; 
START SLAVE;
```

## 配置 slave

登录到 slave 节点，使用 `docker exec -it mysql-slave mysql -uroot -proot`  命令，然后进行相关配置。
### 设置 master 节点
先给 slave 节点设置 master 节点的 binlog 文件和 position，这是在面上通过 `SHOW MASTER STATUS` 所查询到的值，代码如下：
```sql
CHANGE MASTER TO MASTER_HOST='mysql-master', MASTER_USER='repl', MASTER_PASSWORD='repl', MASTER_LOG_FILE='binlog.000003', MASTER_LOG_POS=1741;
```
然后再启动 slave 节点，代码如下：
```
START SLAVE;
```

### 查看节点状态
使用 `SHOW SLAVE STATUS\G` 查看 slave 节点的状态，内容如下：
```
mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: mysql-master
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: binlog.000005
          Read_Master_Log_Pos: 650
               Relay_Log_File: 3413285257e7-relay-bin.000002
                Relay_Log_Pos: 816
        Relay_Master_Log_File: binlog.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 650
              Relay_Log_Space: 1033
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: ccc3db5d-1590-11ee-a214-0242ac110002
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 0
            Network_Namespace:
1 row in set, 1 warning (0.00 sec)
```

在结果中看到 `Slave_IO_Running: Yes` 和 `Slave_SQL_Running: Yes`，则说明从服务器已经正常启动，并且正在从主服务器同步数据。

# 验证效果
我们去 master 节点，创建一个 test 数据库，并创建一个 test 表。观察到了 slave 节点也同样生成了相关的数据库和表。说明整体的主从复制成功了。
![](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/mysql-master-slave-01.png)

以上就是在 Docker 中设置 MySQL 主从复制的大致步骤。需要注意的是，以上步骤为了简化操作，并没有考虑安全性问题，在实际生产环境中，你需要按照最佳实践来配置 MySQL，要考虑到使用更强的密码、限制用户权限、限制网络访问等。
