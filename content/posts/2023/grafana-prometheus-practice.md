---
title: 【DevOps】使用 Grafana 和 Prometheus 做服务监控
date: 2023-06-27 15:45:00
tags: [DevOps,Grafana,Prometheus]
---

最近需要对自己的一些服务做监控，于是想要动手实践使用一下 Grafana 和 Prometheus。这篇文章就主要介绍搭建 Grafana 和 Prometheus 对 MySQL 做监控的过程。

# 工具介绍

Prometheus 是一个开源的系统监控和警报工具包，由 SoundCloud 启动。它的设计目标是对大规模的服务容器和微服务体系结构提供稳健的可靠性和实时的度量。Prometheus 提供了多种数据收集机制，包括通过 HTTP 拉取目标应用程序暴露的 metrics，通过服务发现或者静态配置去发现新的目标，或者接受推送的时间序列数据。Prometheus 还提供了一种灵活的查询语言（PromQL），以便在其数据上进行切片和切块，生成聚合，预测等操作。
Prometheus 服务器以一个独立的服务器形式运行，它从被配置的位置定期拉取度量数据，将数据存储在它的时间序列数据库中，并运行定义的警报规则以生成新的警报。它的核心组件是一个多维数据模型和一套灵活的查询语言。在数据收集方面，它主要采用了一种拉模式的设计，也支持推模式的数据收集。

Grafana 是一个开源的度量分析和可视化套件。它通常被用于时间序列数据的可视化，这种数据通常来源于如今在软件开发中广泛使用的实时监控系统，如 Prometheus。Grafana 主要用于可视化和探索度量数据。它可以通过一种查询编辑器（这种编辑器支持数据源特定的语言）来创建可视化的图表，并通过一种灵活的方式将这些图表组织在一个或多个"仪表板"上。此外，Grafana 还支持警报和通知功能，以及丰富的插件生态系统。
Grafana 是一个前后端分离的 Web 应用程序。其前端部分使用 AngularJS 和 Later.js 构建，后端部分则使用 Go 语言编写。在运行时，Grafana 服务器将请求时间序列数据，然后运行查询并将结果绘制成图表。


# 搭建过程

## docker 运行 MySQL
首先，我们用 docker 启动一个 MySQL 8.0 的容器，来充当我们需要被监控的服务。

创建 MySQL 8.0 配置文件 my.conf，该配置文件需要放在 conf 文件夹下，配置如下：
```
[mysqld]
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
docker 命令如下：
```shell
docker pull mysql/mysql-server

docker run -p 3306:3306 --name mysql-8.0 \
-v /Users/orechou/docker/mysql8/logs:/var/log/mysql \
-v /Users/orechou/docker/mysql8/data:/var/lib/mysql \
-v /Users/orechou/docker/mysql8/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql/mysql-server:latest
```

注意，你需要把命令中的本地挂载路径改成你自己机器上的路径。容器启动起来之后，远程连接 MySQL 的时候出现如下异常：HY000][1130] null, message from server: "Host '172.17.0.1' is not allowed to connect to this MySQL server". 则是需要配置mysql对外开放连接权限，操作如下：
```shell
docker exec -it mysql-8.0 mysql -uroot -proot

use mysql;
select host,user from user;
update user set host = '%' where user = 'root';
flush privileges;
select host,user from user;
```

自此，用于被监控的 MySQL 已经就绪了。

## docker 运行 node-exporter 和 mysqld-exporter

node-exporter 和 mysqld-exporter 都是 Prometheus 社区提供的一种官方的数据采集工具，用于采集和监控系统级别的 metrics，如 CPU 使用情况、内存使用、磁盘 IO、网络 IO、系统负载等。

运行 node-exporter 的 docker 命令如下：
```shell
docker run -d -p 9100:9100 --name node-exporter prom/node-exporter
```

创建 mysqld-exporter 配置文件 .my.cnf，配置如下：
```
[client]
user = root
password = root
host = 192.168.52.59
```

运行 mysqld-exporter 的 docker 命令如下：
```shell
docker run -d --name mysql-exporter \
  -p 9104:9104 \
  -v /Users/orechou/docker/mysqld-exporter/.my.cnf:/etc/mysqld-exporter/.my.cnf \
  prom/mysqld-exporter --config.my-cnf=/etc/mysqld-exporter/.my.cnf
```

## docker 运行 Prometheus

创建 Prometheus 的配置文件 prometheus.yml，配置如下：
```yml
global:
  scrape_interval:     60s
  evaluation_interval: 60s
 
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['192.168.52.59:9090']
        labels:
          instance: prometheus
# 不要使用localhost，因为使用localhost或者127.0.0.1都是去访问普罗米修斯容器的本身；你要使用宿主机ip+端口
  - job_name: node_exporter
    static_configs:
      - targets: ['192.168.52.59:9100']
        labels:
          instance: node_exporter
  # - job_name: pandora-demo
  #   metrics_path: '/actuator/prometheus' # 指标获取路径
  #   static_configs:
  #     - targets: ['192.168.52.70:18001']
  #       labels:
  #         instance: localhost
  - job_name: mysqld_exporter
    static_configs:
      - targets: ['192.168.52.59:9104']
        labels:
          instance: mysqld_exporter
```

运行 Prometheus 的 docker 命令如下：
```shell
docker run  -d -p 9090:9090 --name prometheus -v /Users/orechou/docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

然后我们访问地址：http://localhost:9090/targets ，截图如下：
![](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/prometheus-target.png)
会看到 Prometheus 正在监控的所有目标（targets）。这些目标代表 Prometheus 正在监控的数据源。在 Prometheus 的术语中，一个 "target" 通常指的是一个被监控的服务的实例。例如，一个运行在某个主机上的 MySQL 数据库、一个 Node Exporter 进程，或者一个 HTTP 服务器都可以是一个 target。包括最后一次从该 target 抓取数据的时间、抓取状态（是否成功）、抓取时长以及该 target 对应的 endpoint。

## 运行 Grafana

Grafana 我没有使用 docker 运行，因为 docker 的 Grafana 镜像版本没有 10 的最新版本。于是我使用 brew 进行的安装。安装、启动和停止的命令如下：
```shell
brew install grafana

# 启动
brew services start grafana
# 停止
brew services stop grafana
```

启动起来之后，Grafana 的地址是：http://localhost:3000/。

### 添加 Prometheus 数据源
第一步是添加 Prometheus 数据源，添加的截图如下：
![](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/grafana-add-datasource.png)

### 导入监控模板
第二步是导入监控模板，我们选择社区里面 https://grafana.com/grafana/dashboards/7362-mysql-overview/ 这个模板，其模板 ID 是 7362。操作截图如下：
![](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/grafana-add-dashboard-01.png)

![](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/grafana-add-dashboard-02.png)

完事之后，打开监控模板就能出现监控信息了。
![](https://orechou.oss-cn-shenzhen.aliyuncs.com/images/grafana-mysql-dashboard.png)

### 观察监控数据变化
到这里整个实践操作的过程基本已经结束了。我为了观察 MySQL 监控数据的变化，还写了一个 Python 脚本来测试数据库查询，从而观察监控面版的数据变化。相关的代码也贴在下面：

```python
import time
import mysql.connector
from multiprocessing import Pool

# 你的数据库配置
db_config = {
    'host': '127.0.0.1',
    'user': 'root',
    'password': 'root',
    'database': 'test_db'
}

# 查询语句
query = 'SELECT * FROM test LIMIT 10'

# 测试时间（秒）
test_duration = 60 * 2

# 测试并发数
concurrency = 16


def worker(_):
    cnx = mysql.connector.connect(**db_config)
    cursor = cnx.cursor()

    start_time = time.time()
    queries = 0

    while time.time() - start_time < test_duration:
        cursor.execute(query)
        cursor.fetchall()
        queries += 1

    cursor.close()
    cnx.close()

    return queries


if __name__ == "__main__":
    with Pool(concurrency) as p:
        result = p.map(worker, range(concurrency))
        total_queries = sum(result)
        print(f'Total queries executed: {total_queries}')
        print(f'QPS: {total_queries / test_duration}')
```

实践一遍后，对整个流程有了一定的了解。后续监控自己的服务，实操起来也会更流畅。
