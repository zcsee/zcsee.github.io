---
layout: post
title: 'docker部署hadoop集群'
date: 2023-03-13
categories: 中间件
author: Jason
tags: hadoop 集群 docker
---



## 创建docker网络，指定网段

docker network create --subnet=172.128.0.0/24 hadoop

### 运行master

docker run -d --name DMaster315 -h DMaster315 \
 -p 50070:50070 \
 --privileged=true \
 --network hadoop \
 --ip 172.128.0.10 \
registry.cn-beijing.aliyuncs.com/jing-studio/centos7-hadoop /usr/sbin/init

## 运行slave1

docker run -d --name DSlave01-315 -h DSlave01-315 \
--privileged=true \
--network hadoop \
--ip 172.128.0.20 \
registry.cn-beijing.aliyuncs.com/jing-studio/centos7-hadoop /usr/sbin/init

## 运行slave2

docker run -d --name DSlave02-315 -h DSlave02-315 \
--privileged=true \
--network hadoop \
--ip 172.128.0.30 \
registry.cn-beijing.aliyuncs.com/jing-studio/centos7-hadoop /usr/sbin/init

## 传送到DSlave01-315

scp slaves root@DSlave01-315:/usr/local/hadoop/etc/hadoop/
scp mapred-site.xml root@DSlave01-315:/usr/local/hadoop/etc/hadoop
scp yarn-site.xml root@DSlave01-315:/usr/local/hadoop/etc/hadoop/  
scp hdfs-site.xml root@DSlave01-315:/usr/local/hadoop/etc/hadoop
scp core-site.xml root@DSlave01-315:/usr/local/hadoop/etc/hadoop
scp -r /home/data/hadoopdata root@DSlave01-315:/home/

## 传送到DSlave02-315

scp slaves root@DSlave02-315:/usr/local/hadoop/etc/hadoop/
scp mapred-site.xml root@DSlave02-315:/usr/local/hadoop/etc/hadoop
scp yarn-site.xml root@DSlave02-315:/usr/local/hadoop/etc/hadoop/  
scp hdfs-site.xml root@DSlave02-315:/usr/local/hadoop/etc/hadoop
scp core-site.xml root@DSlave02-315:/usr/local/hadoop/etc/hadoop
scp -r /home/data/hadoopdata root@DSlave02-315:/home/

## 设置主机名

vi /etc/sysconfig/network

## DMaster315

NETWORKING=yes
HOSTNAME=DMaster315

## DSlave01-315

NETWORKING=yes
HOSTNAME=DSlave01-315

## DSlave02-315

NETWORKING=yes
HOSTNAME=DSlave02-315

## 整理hosts，并配置到每台机器上

172.128.0.10 DMaster315
172.128.0.20 DSlave01-315
172.128.0.30 DSlave02-315

## docker部署，重启会导致hosts文件被初始化，需要将初始化hosts和服务停止部分设置到开机启动过程中

vi ~/.bashrc

echo "172.128.0.10 DMaster315" >> /etc/hosts
echo "172.128.0.20 DSlave01-315" >> /etc/hosts
echo "172.128.0.20 DSlave02-315" >> /etc/hosts
systemctl stop iptables.service

## 创建用于存放的目录

mkdir -p /home/data
mkdir -p /home/data/hadoopdata

## =======================================================

## 调整配置文件，注意点是`<property>`标签要在configuration中

### 切换到配置文件存放的目录

cd /usr/local/hadoop/etc/hadoop

### 配置hadoop的核心参数，具体文件为core-site.xml

```xml
<configuration><property>
           <name>fs.defaultFS</name>
           <value>hdfs://DMaster315:9000</value>
          </property>
      <property>
    <name>hadoop.tmp.dir</name>
                <value>/home/data/hadoopdata</value>
      </property>
</configuration>
```

### 配置hdfs的相关参数，具体文件为hdfs-site.xml

```xml
<configuration>
    <property><!--配置存储namenode数据的目录-->
        <name>dfs.namenode.name.dir</name>
        <value>/home/data/hadoopdata/name</value>
    </property>
　<property><!--配置存储datanode数据的目录-->
        <name>dfs.datanode.data.dir</name>
        <value>/home/data/hadoopdata/data</value>
    </property>
　　<property><!--配置副本数量-->
          <name>dfs.replication</name>
          <value>1</value>
        </property>　　
　　<property><!--配置第二名称节点，放到DMaster315 -->
　　　　 <name>dfs.secondary.http.address</name>
　　　　 <value>DMaster315:50090</value>
　　</property>
</configuration>
```

### 配置map-reduce，具体文件为mapred-site.xml

#### 先从模板复制一份，再进行修改

```shell
cp mapred-site.xml.template mapred-site.xml && vi mapred-site.xml
```

```xml
<configuration>
   <property>
     <name>mapreduce.Framework.name</name>
     <value>yarn</value>
 </property>
</configuration>
```

### 配置用于资源调整的yarn，具体文件为yarn-site.xml

```xml
<configuration>
    <property> <!--配置yarn主节点-->
         <name>yarn.resourcemanager.hostname</name>
         <value>DMaster315</value>
    </property>
    <property><!--配置执行的计算框架-->
         <name>yarn.nodemanager.aux-services</name>
         <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

### 配置slaves，具体文件为slaves

DMaster315
DSlave01-315
DSlave02-315

## 测试互联互通性,确保服务器之间都能互通

ping DMaster315
ping DSlave01-315
ping DSlave02-315

## 文件传送

## 将master上配置过的文件查询到slave1和slave2上

```shell
cd /usr/local/hadoop/etc/hadoop
```

## 传送文件到DSlave01-315

```shell
scp slaves root@DSlave01-315:/usr/local/hadoop/etc/hadoop/
scp mapred-site.xml root@DSlave01-315:/usr/local/hadoop/etc/hadoop
scp yarn-site.xml root@DSlave01-315:/usr/local/hadoop/etc/hadoop/  
scp hdfs-site.xml root@DSlave01-315:/usr/local/hadoop/etc/hadoop
scp core-site.xml root@DSlave01-315:/usr/local/hadoop/etc/hadoop
scp -r /home/data/hadoopdata root@DSlave01-315:/home/
```

## 传送文件到DSlave02-315

```shell
scp slaves root@DSlave02-315:/usr/local/hadoop/etc/hadoop/
scp mapred-site.xml root@DSlave02-315:/usr/local/hadoop/etc/hadoop
scp yarn-site.xml root@DSlave02-315:/usr/local/hadoop/etc/hadoop/  
scp hdfs-site.xml root@DSlave02-315:/usr/local/hadoop/etc/hadoop
scp core-site.xml root@DSlave02-315:/usr/local/hadoop/etc/hadoop
scp -r /home/data/hadoopdata root@DSlave02-315:/home/
```

## =======================================================================

## 格式化节点

### 在三台机器分别执行

hadoop namenode -format

### Master节点执行成功后，会有如下提示，其中172.128.0.20为该节点的IP

SHUTDOWN_MSG: Shutting down NameNode at DMaster315/172.128.0.10

### Slave节点执行成功后，会有如下提示，其中172.128.0.20为该节点的IP

SHUTDOWN_MSG: Shutting down NameNode at DSlave01-315/172.128.0.20

## 启动hadoop集群

/usr/local/hadoop/sbin/start-all.sh

## 查看运行状态

hadoop dfsadmin -report

## Web UI查看集群状态，通过浏览器打开,虚拟机的IP为192.168.216.3

192.168.216.3:50070
