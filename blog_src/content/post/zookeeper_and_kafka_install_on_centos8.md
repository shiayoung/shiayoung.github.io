---
title: "Zookeeper 与 Kafka 集群搭建与配置"
date: "2020-05-11T20:00:00"
lastmod: "2020-08-05T20:00:00"
draft: true
tags: ["kafka", "基础组件", "zookeeper"]
categories: ["基础组件"]
---

> 本文涵盖 centos 8 环境下的 java jdk 安装、zookeeper 安装及配置、kafka 安装及配置

## CentOS
###  安装 JDK (openjdk)
    yum install java-1.8.0-openjdk-devel

   
### 配置zookeeper环境变量，首先打开profile文件
先 [下载](https://zookeeper.apache.org/releases.html) zookeeper 文档包
``` sh
wget https://mirror.bit.edu.cn/apache/zookeeper/zookeeper-3.5.8/apache-zookeeper-3.5.8-bin.tar.gz
```

- 将 zookeeper 解压到 /usr/local 文件夹下
``` sh
tar zxf apache-zookeeper-3.5.8-bin.tar.gz -C /usr/local        
```

- 简化命名
``` sh
mv apache-zookeeper-3.5.8-bin zookeeper-3.5.8
```

- 配置环境变量（为方便运行，不用进入程序根目录运行相关脚本）
    
将下面内容 append 到 /etc/profile 文档最后

``` sh
export ZK_HOME=/usr/local/zookeeper-3.5.8
export ZK_PATH=$ZK_HOME/bin
export KAFKA_HOME=/usr/local/kafka_2.13-2.6.0
export KAFKA_PATH=$KAFKA_HOME/bin
export PATH=$PATH:$ZK_HOME:$KAFKA_HOME:$ZK_PATH:$KAFKA_PATH
```

配置完成后运行 `source /etc/profile` 使用配置生效 

- 修改 zoo.cfg 配置文件（如果不存在，则从 zoo_example.cfg 复制） 
``` sh
#修改数据文件夹路径
dataDir=/usr/local/zookeeper-3.5.8/data
#在文件末尾添加
server.1=192.168.140.81:2888:3888
server.2=192.168.140.82:2888:3888
server.3=192.168.140.83:2888:3888
```
将 server.{num} 中的 num 写入 data 文件夹下的 myid 文件

### zookeeper 相关命令
``` sh
zkServer.sh start # 以 background 方式启动 zookeeper 服务

zkServer.sh status # 查看 zookeeper 服务状态

zkServer.sh start-foreground # 启动 zookeeper 服务
```

如启动成功，运行 zkServer.sh status 会有如下结果：
```
leader 节点输出：
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper-3.5.8/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost.
Mode: leader
-----------------------------------------------------------------
follower 节点输出：
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper-3.5.8/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost.
Mode: follower
```

### kafka 的 server.properties 相关 key 的配置内容如下
```
listeners
listeners就是主要用来定义Kafka Broker的Listener的配置项。是kafka真正bind的地址。

advertised.listeners
advertised.listeners参数的作用就是将 Broker 的 Listener 信息发布到Zookeeper中，是暴露给外部的 listeners，如果没有设置，会用 listeners。

inter.broker.listener.name
inter.broker.listener.name：专门用于Kafka集群中Broker之间的通信

listener.security.protocol.map
配置监听者的安全协议的，比如PLAINTEXT、SSL、SASL_PLAINTEXT、SASL_SSL


broker.id=1
listeners=EXTERN://0.0.0.0:9092,INTERN://0.0.0.0:9093
advertised.listeners=EXTERN://192.168.140.81:9092,INTERN://192.168.140.81:9093
inter.broker.listener.name=INTERN
listener.security.protocol.map=PLAINTEXT:PLAINTEXT,SSL:SSL,INTERN:PLAINTEXT,EXTERN:PLAINTEXT
-----------------------------------------------------------------
broker.id=2
listeners=EXTERN://0.0.0.0:9092,INTERN://0.0.0.0:9093
advertised.listeners=EXTERN://192.168.140.82:9092,INTERN://192.168.140.82:9093
inter.broker.listener.name=INTERN
listener.security.protocol.map=PLAINTEXT:PLAINTEXT,SSL:SSL,INTERN:PLAINTEXT,EXTERN:PLAINTEXT
-----------------------------------------------------------------
broker.id=3
listeners=EXTERN://0.0.0.0:9092,INTERN://0.0.0.0:9093
advertised.listeners=EXTERN://192.168.140.83:9092,INTERN://192.168.140.83:9093
inter.broker.listener.name=INTERN
listener.security.protocol.map=PLAINTEXT:PLAINTEXT,SSL:SSL,INTERN:PLAINTEXT,EXTERN:PLAINTEXT
```

### 启动 kafka
``` sh
bin/kafka-server-start.sh -daemon config/server.properties
OR
kafka-server-start.sh -daemon "$KAFKA_HOME/config/server.properties"

kafka-topics.sh --create --zookeeper 192.168.140.81:2181 --replication-factor 3 --partitions 1 --topic test-topic

kafka-topics.sh --describe --zookeeper 192.168.140.81:2181 --topic test-topic

kafka-console-producer.sh --bootstrap-server 192.168.140.81:9092 --topic test-topic

kafka-console-consumer.sh --bootstrap-server 192.168.140.81:9092 --topic test-topic # --from-beginning [--group tester]

1.x 新版本加入 consumer-group
kafka-console-consumer.sh --bootstrap-server 192.168.140.81:9092 --topic test-topic --group tester

0.x 低版本加入 consumer-group
kafka-console-consumer.sh --bootstrap-server 192.168.140.81:9092 --topic test-topic --consumer-property group.id=tester


kafka-consumer-groups --bootstrap-server localhost:9092 --describe --all-group

kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group my-group

## 查看指定 consumer group 中的成员
kafka-consumer-groups.sh --bootstrap-server localhost:9093 --describe --group tester --members

## 查看所有 consumer groups 中的成员
kafka-consumer-groups.sh --bootstrap-server localhost:9093 --describe --all-groups --members
```

## zookeeper 相关操作
首先启动 `zkCli.sh`，进入 zkCli 的运行环境

### 查看注册的 kafka broker 实例数
    ls /broker/ids

### 查看 broker 为 1 的 kafka 实例信息
    get /brokers/ids/1

### 查看 topic 元信息（可以获取 broker leader 信息）
    ls /brokers/topics/test-topic/partitions/0/state

## 参考
[Kafka集群搭建与配置](https://www.jianshu.com/p/bdd9608df6b3)