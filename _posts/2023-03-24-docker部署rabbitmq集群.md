---
layout: post
title: 'docker部署RabbitMQ集群'
# subtitle: '通过shell脚本，进行redis批量迁移'
date: 2023-03-24
categories: docker
author: Jason
# cover: 'assets/img/profile.png'
tags: rabbitmq cluster
---

# docker部署RabbitMQ集群

## 环境规划



机器数量：3台

机器信息：

| 主机IP      | 角色 | 备注       |
| ----------- | ---- | ---------- |
| 172.19.0.10 | mq1  | docker容器 |
| 172.19.0.20 | mq2  | docker容器 |
| 172.19.0.30 | mq3  | docker容器 |



## 部署前准备



```shell
# 创建docker网路空间
docker network create rabbitmq --subnet 172.19.0.0/.24

# 创建本地目录，用于存放配置文件和erlang.cookie
mkdir -p /data/dockerRabbitmq/mq{1,2,3}
vim /data/dockerRabbitmq/mq1/rabbitmq.conf
## 配置文件内容
loopback_users.guest = false
listeners.tcp.default = 5672
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@mq1
cluster_formation.classic_config.nodes.2 = rabbit@mq2
cluster_formation.classic_config.nodes.3 = rabbit@mq3


vim /data/dockerRabbitmq/mq1/.erlang.cookie
## erlang.cookie内容(任意内容，保证mq集群中的节点此文件数据一致即可)
UDCUIBNPHPETOIURAHRF

## 配置.erlang.cookie权限为owner只读
chmod 600 /data/dockerRabbitmq/mq1/.erlang.cookie

# 分发mq1的配置文件到mq2和mq3
cp -ar /data/dockerRabbitmq/mq1 /data/dockerRabbitmq/mq2
cp -ar /data/dockerRabbitmq/mq1 /data/dockerRabbitmq/mq3

```

## 运行docker容器，组成集群

需要指定的信息如下：

- docker网络
- 容器的IP
- 本地配置的挂载地址
- 默认账号信息
- 映射的端口
- 主机名

```shell
# docker运行

# 运行mq1
docker run -d \
--net rabbitmq \
--ip 172.19.0.10 \
-v /data/dockerRabbitmq/mq1/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf \
-v /data/dockerRabbitmq/mq1/.erlang.cookie:/var/lib/rabbitmq/.erlang.cookie \
-e RABBITMQ_DEFAULT_USER=admin \
-e RABBITMQ_DEFAULT_PASS=admin \
-p 5677:5672 \
-p 15677:15672 \
--name mq1 \
--hostname mq1 \
rabbitmq:3.11-management


# 运行mq2
docker run -d \
--net rabbitmq \
--ip 172.19.0.20 \
-v /data/dockerRabbitmq/mq2/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf \
-v /data/dockerRabbitmq/mq2/.erlang.cookie:/var/lib/rabbitmq/.erlang.cookie \
-e RABBITMQ_DEFAULT_USER=admin \
-e RABBITMQ_DEFAULT_PASS=admin \
-p 5678:5672 \
-p 15678:15672 \
--name mq2 \
--hostname mq2 \
rabbitmq:3.11-management

# 运行mq3
docker run -d \
--net rabbitmq \
--ip 172.19.0.30 \
-v /data/dockerRabbitmq/mq3/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf \
-v /data/dockerRabbitmq/mq3/.erlang.cookie:/var/lib/rabbitmq/.erlang.cookie \
-e RABBITMQ_DEFAULT_USER=admin \
-e RABBITMQ_DEFAULT_PASS=admin \
-p 5679:5672 \
-p 15679:15672 \
--name mq3 \
--hostname mq3 \
rabbitmq:3.11-management

# 查看运行状态
docker ps -a |grep mq
```

### 查看rabbitmq集群状态

```shell
# 通过浏览器打开控制台就能看到集群状态
url：{宿主机的IP}:15677
账号密码信息：
    账号：admin
    密码：admin
```

## 遇到的问题

- Node rabbit@mq1 thinks it's clustered with node rabbit@mq3, but rabbit@mq3 disagrees

  问题原因： 主机集群认为该节点仍在集群中， 而该节点实际上退出集群了。 导致数据文件日志不一致，而无法加入集群

  解决思路1：mq1作为新的节点，加入到mq2和mq3中

  解决方案：

  1. 在mq1上停止rabbitmq进程：rabbitmqctl stop_app
  2. 删除数据目录：rm -rf /var/lib/rabbitmq/mnesia
  3. 在mq3上将mq1移除：rabbitmqctl forget_cluster_node rabbit@mq1 
  4. 在mq3上重新加入mq1：rabbitmqctl join_cluster --disc rabbit@mq1
  5. 在mq1节点上启动mq进程：rabbitctl start_app
  6. 设置为镜像模式：rabbitctl set_policy ha-all "^ha\." '{"ha-mode":"all"}'
