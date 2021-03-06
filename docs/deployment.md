---
layout: global
displayTitle: Deployment
title: Deployment
description: Wormhole WH_VERSION_SHORT Deployment page
---

* This will become a table of contents (this text will be scraped).
{:toc}

## 前期准备

#### 环境准备
- JDK1.8
- Hadoop-client（HDFS，YARN）（支持版本 2.6+）
- Spark-client （支持版本 2.1.1，2.2.0）

#### 依赖服务

- Hadoop 集群（HDFS，YARN）（支持版本 2.6+）
- Zookeeper
- Kafka （支持版本 0.10.0.0）
- Elasticsearch（支持版本 5.x）（非必须，若无则无法查看 wormhole 处理数据的吞吐和延时）
- Grafana （支持版本 4.x）（非必须，若无则无法查看 wormhole 处理数据的吞吐和延时的图形化展示）
- MySQL

#### Jar包准备
mysql-connector-java-{your-db-version}.jar


## 部署配置

**下载 wormhole-0.4.0.tar.gz 包 (链接:https://pan.baidu.com/s/1pKAqp31 密码:4azr)，或者自编译**

```
wget https://github.com/edp963/wormhole/releases/download/0.4.0/wormhole-0.4.0.tar.gz
tar -xvf wormhole-0.4.0.tar.gz
或者自编译，生成的 tar 包在 wormhole/target
git clone -b 0.4 https://github.com/edp963/wormhole.git
cd wormhole
mvn install package -Pwormhole
```

**配置 WORMHOLE_HOME 环境变量**

**修改 application.conf  配置文件**

```
conf/application.conf 配置项介绍

wormholeServer {
  host = "127.0.0.1"
  port = 8989
  token.timeout = 7
  request.timeout = 120s
  admin.username = "admin"    #default admin user name
  admin.password = "admin"    #default admin user password
}

mysql = {
  driver = "slick.driver.MySQLDriver$"
  db = {
    driver = "com.mysql.jdbc.Driver"
    user = "root"
    url = "jdbc:mysql://localhost:3306/wormhole"
    password = "*******"
    numThreads = 4
  }
}

spark = {
  wormholeServer.user = "wormhole"   #WormholeServer linux user
  wormholeServer.ssh.port = 22       #ssh port, please set WormholeServer linux user can password-less login itself remote
  spark.home = "/usr/local/spark"
  yarn.queue.name = "default"        #WormholeServer submit spark streaming/job queue
  wormhole.hdfs.root.path = "hdfs://nn1/wormhole"   #WormholeServer hdfslog data default hdfs root path
  yarn.rm1.http.url = "localhost:8088"    #Yarn ActiveResourceManager address
  yarn.rm2.http.url = "localhost2:8088"   #Yarn StandbyResourceManager address
}

zookeeper.connection.url = "localhost:2181"  #WormholeServer stream and flow interaction channel

kafka = {
  brokers.url = "locahost:9092"         #WormholeServer feedback data store
  zookeeper.url = "localhost:2181"
  consumer = {
    feedback.topic = "wormhole_feedback"
    poll-interval = 20ms
    poll-timeout = 1s
    stop-timeout = 30s
    close-timeout = 20s
    commit-timeout = 70s
    wakeup-timeout = 60s
    max-wakeups = 10
    session.timeout.ms = 60000
    heartbeat.interval.ms = 50000
    max.poll.records = 500
    request.timeout.ms = 80000
    max.partition.fetch.bytes = 10485760
  }
}

#Wormhole feedback data store, if doesn't want to config, you will not see wormhole processing delay and throughput
#if not set, please comment it
#elasticSearch.http = {
#  url = "http://localhost:9200"
#  user = ""
#  password = ""
#}

#display wormhole processing delay and throughput data, get admin user token from grafana
#garfana should set to be anonymous login, so you can access the dashboard through wormhole directly
#if not set, please comment it
#grafana = {
#  url = "http://localhost:3000"
#  admin.token = "jihefouglokoj"
#}

#delete feedback history data on time
maintenance = {
  mysql.feedback.remain.maxDays = 7
  elasticSearch.feedback.remain.maxDays = 7
}

#Dbus integration, if not set, please comment it
#dbus.namespace.rest.api.url = ["http://localhost:8080/webservice/tables/riderSearch"]
```
**设置 wormhole server mysql 数据库编码为 uft8，并授权可远程访问**

**上传 mysql-connector-java-{version}.jar 至 $WORMHOLE_HOME/lib 目录**

**须使用 application.conf 中 spark.wormholeServer.user 项对应的用户启动服务，且须配置该用户可通过 ssh 远程免密登录到自己**

**若配置 Grafana，Grafana 须配置可使用 viewer 类型用户匿名登陆，并生成 admin 类型的 token，配置在 $WORMHOLE_HOME/conf/application.conf 中grafana.admin.token 项中**

**切换到 root 用户，为 WormholeServer 启动用户授权读写 HDFS 目录，若失败，请根据提示手动授权**

```
#将 hadoop 改为 Hadoop 集群对应的 super-usergroup
./deploy.sh --hdfs-super-usergroup=hadoop
```

## 启动停止

#### 启动

```
./start.sh

启动时会自动创建 table，kafka topic，elasticsearch index，grafana datasource，创建 kafka topic时，有时会因环境原因失败，须手动创建

topic name: wormhole_feedback partitions: 4
topic name：wormhole_heartbeat partitions: 1

# 创建或修改 topic 命令
./kafka-topics.sh --zookeeper localhost:2181 --create --topic wormhole_feedback --replication-factor 1 --partitions 4
./kafka-topics.sh --zookeeper localhost:2181 --create --topic wormhole_heartbeat --replication-factor 1 --partitions 1

./kafka-topics.sh --zookeeper localhost:2181 --alter --topic wormhole_feedback  --partitions 4
./kafka-topics.sh --zookeeper localhost:2181 --alter --topic wormhole_heartbeat  --partitions 1
```

#### 停止

```
./stop.sh
```

#### 重启

```
./restart.sh
```

**访问 http://ip:port 即可试用 Wormhole，可使用 admin 类型用户登录，默认用户名，密码见 application.conf 中配置**