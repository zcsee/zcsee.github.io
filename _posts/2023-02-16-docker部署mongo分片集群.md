---
layout: post
title: 'docker部署mongo分片集群'
# subtitle: '优化了直接修改数据存放方案会有/run/k3s的挂载遗留问题'
date: 2023-02-16
categories: 数据库
author: Jason
# cover: 'assets/img/profile.png'
tags: mongo 分片集群
---

## 背景
> 通过docker部署一套mongo集群  

## mongo 分片集群结构

```shell
config
    config0
    config1
    config2
shard
    shard0
        shard00
        shard01
        shard02
    shard1
        shard10
        shard11
        shard12
mongos
```

## 具体目录及配置文件

```shell
config
    conf
    config0
        data
        log
    config1
        data
        log
    config2
        data
        log
shard
    conf0
        mongod.conf
    conf1
        mongod.conf
    shard0
        shard00
            data
            log
        shard01
            data
            log
        shard02
            data
            log
    shard1
        shard10
            data
            log
        shard11
            data
            log
        shard12
            data
            log
mongos
    conf
        mongod.conf
    data
    log
```

## 构建目录结构命令

```shell
mkdir -p /data1/mongo
cd /data1/mongo
mkdir -p config/conf
touch config/conf/mongod.conf
mkdir -p config/config{0,1,2}/{data,log}

mkdir -p shard/conf{0,1}
touch shard/conf{0,1}/mongod.conf
mkdir -p shard/shard0/shard0{0,1,2}/{data,log}
mkdir -p shard/shard1/shard0{0,1,2}/{data,log}
mkdir -p mongos/{conf,data,log}
touch mongos/conf/mongod.conf
```

## 生成 keyFile

MongoDB 使用 keyfile 认证，副本集中的每个 mongod 实例使用 keyfile 内容作为认证其他成员的共享密码。mongod 实例只有拥有正确的 keyfile 才可以加入副本集。
keyFile 的内容必须是 6 到 1024 个字符的长度，且副本集所有成员的 keyFile 内容必须相同。
有一点要注意是的：在 UNIX 系统中，keyFile 必须没有组权限或完全权限（也就是权限要设置成 X00 的形式）。Windows 系统中，keyFile 权限没有被检查。
可以使用任意方法生成 keyFile。例如，如下操作使用 openssl 生成复杂的随机的 1024 个字符串。然后使用 chmod 修改文件权限，只给文件拥有者提供读权限

```shell
# 400权限是要保证安全性，否则mongod启动会报错
openssl rand -base64 756 > mongodb.key
chmod 400 mongodb.key
```

## mongodb 配置文件

### 配置服务器的配置文件

```shell
cat  > config/conf/mongod.conf <<EOF
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# Where and how to store data.
storage:
  dbPath: /data/db
  journal:
    enabled: true
  directoryPerDB: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 0.25
      directoryForIndexes: true

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

security:
  keyFile: /etc/key.key
  clusterAuthMode: "keyFile"
  authorization: enabled

# network interfaces
net:
  port: 27019
  bindIp: 0.0.0.0

# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

#replication:
replication:
   oplogSizeMB: 256
   replSetName: cfg

sharding:
  clusterRole: configsvr

EOF
```

### 分片服务器 0 的配置文件

```shell
cat  > shard/conf0/mongod.conf <<EOF
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# Where and how to store data.
storage:
  #mongod实例存储其数据的目录
  dbPath: /data/db
  journal:
    #启用或禁用持久性日志已确保数据文件保持有效和可恢复
    enabled: true
  directoryPerDB: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 0.25
      directoryForIndexes: true

# where to write logging data.
systemLog:
  #日志输出的目标指定为文件
  destination: file
  #当mongos或mongod实例重新启动时，mongos或mongod会将新条目附加到现有日志文件的末尾
  logAppend: true
  #日志存放目录
  path: /var/log/mongodb/mongod.log

security:
  keyFile: /etc/key.key
  clusterAuthMode: "keyFile"
  authorization: enabled

# network interfaces
net:
  port: 27018
  bindIp: 0.0.0.0

# how the process runs
processManagement:
  #启用在后台运行mongos或mongod进程的守护进程模式
  #fork: true
  #指定用于保存mongos或mongod进程的进程ID的文件位置
  #pidFilePath: "/data/replica_sets/myrs_27018/log/mongod.pid"
  timeZoneInfo: /usr/share/zoneinfo

#replication:
replication:
   oplogSizeMB: 256
   #副本集的名称
   replSetName: rs0

sharding:
  clusterRole: shardsvr
EOF
```

### 分片服务器 1 的配置文件

```shell
cat > shard/conf1/mongod.conf << EOF
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# Where and how to store data.
storage:
  dbPath: /data/db
  journal:
    enabled: true
  directoryPerDB: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 0.25
      directoryForIndexes: true

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

security:
  keyFile: /etc/key.key
  clusterAuthMode: "keyFile"
  authorization: enabled

# network interfaces
net:
  port: 27018
  bindIp: 0.0.0.0

# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

#replication:
replication:
   oplogSizeMB: 256
   replSetName: rs1

sharding:
  clusterRole: shardsvr
EOF
```

### 路由服务器的配置文件

```shell
cat > mongos/conf/mongod.conf << EOF
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/


# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

security:
  keyFile: /etc/key.key
  clusterAuthMode: "keyFile"

# network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0

# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

sharding:
  configDB: cfg/mongo-config0:27019,mongo-config1:27019,mongo-config1:27019
EOF
```

### centos 环境检查

> 检查/etc/timezone 是否为文件，如为目录则删除后，新建一个内容为"Asia/Shanghai"的/etc/timezone 文件

```shell
test -d /etc/timezone && rmdir /etc/timezone && cat > /etc/timezone <<EOF
Asia/Shanghai
EOF
```

## docker 编排文件

> 创建自定义网络,使所有容器在同一网络里，互联互通

```shell
docker network create basenetwork --subnet=172.18.0.0/16
```

> 创建 docker-compose.yaml 编排文件

```shell
cat > docker-compose.yaml << EOF
version:"3.9"
# optional since v1.27.0
services:
  # mongodb开始************************
  # https://www.cnblogs.com/ricklz/p/13237419.html
  #
  # sharding--->
  # docker-compose部署单机版本分片mongo  https://cloud.tencent.com/developer/article/1717360
  # docker部暑mongodb_4.4.8 sharding集群（arm64和amd64）,mongodb_consistent_backup备份与恢复  https://blog.csdn.net/u010533742/article/details/119754119
  # 分布式NoSQL数据库MongoDB初体验-v5.0.5 https://www.cnblogs.com/itxiaoshen/p/15728782.html
  # https://docs.mongodb.com/manual/

###### 配置服务器，存储数据库的元数据（路由、分片）。副本配置。
  mongo-config0:
    image: mongo:latest
    privileged: true
    hostname: mongo-config0
    container_name: mongo-config0
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - ./mongo/config/config0/data:/data/db # 挂载数据目录
      - ./mongo/config/config0/log:/var/log/mongodb
      - ./mongo/mongodb.key:/etc/key.key
      - ./mongo/config/conf:/etc/mongod
    command: -f /etc/mongod/mongod.conf
    #定义IP网络
    networks:
      basenetwork:
        ipv4_address: 172.18.0.30

  mongo-config1:
    image: mongo:latest
    privileged: true
    hostname: mongo-config1
    container_name: mongo-config1
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - ./mongo/config/config1/data:/data/db # 挂载数据目录
      - ./mongo/config/config1/log:/var/log/mongodb
      - ./mongo/mongodb.key:/etc/key.key
      - ./mongo/config/conf:/etc/mongod
    command: -f /etc/mongod/mongod.conf
    #定义IP网络
    networks:
      basenetwork:
        ipv4_address: 172.18.0.31

  mongo-config2:
    image: mongo:latest
    privileged: true
    hostname: mongo-config2
    container_name: mongo-config2
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - ./mongo/config/config2/data:/data/db # 挂载数据目录
      - ./mongo/config/config2/log:/var/log/mongodb
      - ./mongo/mongodb.key:/etc/key.key
      - ./mongo/config/conf:/etc/mongod
    command: -f /etc/mongod/mongod.conf
    #定义IP网络
    networks:
      basenetwork:
        ipv4_address: 172.18.0.32


  mongo-shard0-0:
    image: mongo:latest
    privileged: true
    hostname: mongo-shard0-0
    container_name: mongo-shard0-0
    # depends_on:
    #   - "mongo-shard0-1"
    #   - "mongo-shard0-2"
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - ./mongo/shard/shard00/data:/data/db # 挂载数据目录
      - ./mongo/shard/shard00/log:/var/log/mongodb
      - ./mongo/mongodb.key:/etc/key.key
      - ./mongo/shard/conf0:/etc/mongod
      # - ./mongo/shard/scripts:/scripts
      # - ./mogon/shard/initdb:/docker-entrypoint-initdb.d # 需要初始化的数据库脚本存放文件夹，容器启动后会自动执行。
    command: -f /etc/mongod/mongod.conf
    # entrypoint: [ "/scripts/setup.sh" ]
    #定义IP网络
    networks:
      basenetwork:
        ipv4_address: 172.18.0.33

  mongo-shard0-1:
    image: mongo:latest
    privileged: true
    hostname: mongo-shard0-1
    container_name: mongo-shard0-1
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - ./mongo/shard/shard01/data:/data/db # 挂载数据目录
      - ./mongo/shard/shard01/log:/var/log/mongodb
      - ./mongo/mongodb.key:/etc/key.key
      - ./mongo/shard/conf0:/etc/mongod
    command: -f /etc/mongod/mongod.conf
    #定义IP网络
    networks:
      basenetwork:
        ipv4_address: 172.18.0.34

  mongo-shard0-2:
    image: mongo:latest
    privileged: true
    hostname: mongo-shard0-2
    container_name: mongo-shard0-2
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - ./mongo/shard/shard02/data:/data/db # 挂载数据目录
      - ./mongo/shard/shard02/log:/var/log/mongodb
      - ./mongo/mongodb.key:/etc/key.key
      - ./mongo/shard/conf0:/etc/mongod
    command: -f /etc/mongod/mongod.conf
    #定义IP网络
    networks:
      basenetwork:
        ipv4_address: 172.18.0.35

  mongo-shard1-0:
    image: mongo:latest
    privileged: true
    hostname: mongo-shard1-0
    container_name: mongo-shard1-0
    # depends_on:
    #   - "mongo-shard1-1"
    #   - "mongo-shard1-2"
    # environment:
    #   - MONGO1=mongo-shard1-0
    #   - MONGO2=mongo-shard0-1
    #   - MONGO3=mongo-shard0-2
    #   - RS=rs1
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - ./mongo/shard/shard10/data:/data/db # 挂载数据目录
      - ./mongo/shard/shard10/log:/var/log/mongodb
      - ./mongo/mongodb.key:/etc/key.key
      - ./mongo/shard/conf1:/etc/mongod
      # - ./mogon/shard/initdb:/docker-entrypoint-initdb.d # 需要初始化的数据库脚本存放文件夹，容器启动后会自动执行。
      # - ./mongo/shard/scripts:/scripts
    command: -f /etc/mongod/mongod.conf
    # entrypoint: [ "/scripts/setup.sh" ]
    #定义IP网络
    networks:
      basenetwork:
        ipv4_address: 172.18.0.36

  mongo-shard1-1:
    image: mongo:latest
    privileged: true
    hostname: mongo-shard1-1
    container_name: mongo-shard1-1
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - ./mongo/shard/shard11/data:/data/db # 挂载数据目录
      - ./mongo/shard/shard11/log:/var/log/mongodb
      - ./mongo/mongodb.key:/etc/key.key
      - ./mongo/shard/conf1:/etc/mongod
    command: -f /etc/mongod/mongod.conf
    #定义IP网络
    networks:
      basenetwork:
        ipv4_address: 172.18.0.37

  mongo-shard1-2:
    image: mongo:latest
    privileged: true
    hostname: mongo-shard1-2
    container_name: mongo-shard1-2
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - ./mongo/shard/shard12/data:/data/db # 挂载数据目录
      - ./mongo/shard/shard12/log:/var/log/mongodb
      - ./mongo/mongodb.key:/etc/key.key
      - ./mongo/shard/conf1:/etc/mongod
    command: -f /etc/mongod/mongod.conf
    #定义IP网络
    networks:
      basenetwork:
        ipv4_address: 172.18.0.38


  mongos:
    image: mongo:latest
    privileged: true
    hostname: mongos
    container_name: mongos
    volumes:
      - /etc/timezone:/etc/timezone:ro
      - ./mongo/mongos/log:/var/log/mongodb
      - ./mongo/mongodb.key:/etc/key.key
      - ./mongo/mongos/conf:/etc/mongod
    command: mongos -f /etc/mongod/mongod.conf
    ports:
      - "27017:27017"
    #定义IP网络
    networks:
      basenetwork:
        ipv4_address: 172.18.0.39


#定义网络。 docker network create basenetwork --subnet=172.18.0.0/16
networks:
  basenetwork:
    external: true
EOF
```

> 通过 docker-compose 创建容器  
> docker-compose -f docker-compose.yaml up -d

### 初始化配置服务器、分片服务器

#### 初始化配置服务器副本机器

```shell
####初始化配置服务器副本集群
# 1、进入到任一config容器中
docker exec -it mongo-config0 bash
# 2、登录mongo
mongosh --port 27019
# 3、初始config化副本集（https://www.cnblogs.com/ricklz/p/13237419.html）
rs.initiate({
     _id: "cfg",
     members: [
         { _id : 0, host : "mongo-config0:27019" },
         { _id : 1, host : "mongo-config1:27019" },
         { _id : 2, host : "mongo-config2:27019" }
     ]
});
```

#### 初始化两个分片服务器副本集群

```shell
####初始化两个分片服务器副本集群
# 1、进入到分片服务器集群0的任一容器中
docker exec -it mongo-shard0-0 bash
# 2、登录mongo
mongosh --port 27018
# 3、初始分片副本集（https://www.cnblogs.com/ricklz/p/13237419.html）
rs.initiate({
     _id: "rs0",
     members: [
        { _id : 0, host : "mongo-shard0-0:27018" },
        { _id : 1, host : "mongo-shard0-1:27018" },
        { _id : 2, host : "mongo-shard0-2:27018" }
     ]
});

# 1、进入到分片服务器集群1的任一容器中
docker exec -it mongo-shard1-0 bash
# 2、登录mongo
mongosh --port 27018
# 3、初始分片副本集（https://www.cnblogs.com/ricklz/p/13237419.html）
rs.initiate({
     _id: "rs1",
     members: [
         { _id : 0, host : "mongo-shard1-0:27018" },
         { _id : 1, host : "mongo-shard1-1:27018" },
         { _id : 2, host : "mongo-shard1-2:27018" }
    ]
});
```

### 添加用户、分片等操作

```shell
# 1、进入mongos容器并连接mongos。端口号与mongos配置文件中设定一致。
docker exec -i mongos mongosh --port 27017 #可以执行docker exec -it mongos /bin/bash进入容器，再执行mongo连接mongo。因为使用了默认端口27017，所以登录mongo的时候可以不设置端口号。

# 2、创建用户，
use admin
db.createUser(
  {
    user: "root",
    pwd: "123456",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" , {role: 'root', db: 'admin'}]
  }
);
# 验证用户。不验证的话无法执行mongo命令。
db.auth('root','123456')


# 3、将分片加入到集群。
sh.addShard("rs0/mongo-shard0-0:27018,mongo-shard0-0:27018,mongo-shard0-2:27018");
sh.addShard("rs1/mongo-shard1-0:27018,mongo-shard1-1:27018,mongo-shard1-2:27018");
# 查看分片状态。
sh.status();

# 4、创建库并设置分片
# 切换数据库（会自动创建库）
use springboot
# 创建该库的操作用户
db.createUser(
   {
     user: "admin",
     pwd: "123456",
     roles: [ { role: "dbOwner", db: "springboot" } ]
   }
);
db.auth('admin','123456')
# 给新创建的用户赋权分片
db.createUser(
{
	user:'admin',
	pwd:'123456',
	roles: [{role:'enableSharding',db:'springboot'}]
}
)
# 指定要分片的数据库
sh.enableSharding("springboot")
# 指定集合的分片规则
# 这里表示指定springboot库下的user集合的_id字段（也就是主键，每个集合都有这个字段）按hash散列进行分片，{ id : 1 }表示按字段id进度范围分片，这里id必须是整型
# 要分片存储的集合都需要指定分片规则，分片规则一经创建不可修改，只能删除集合再重新设置
sh.shardCollection("springboot.user", { _id : "hashed" } )
# 查询user的集合状态
db.user.stats()

# 5、测试分片效果。
use springboot
# 尝试写入数据，并观察数据分块
# 插入5百万个简单的文档，耐心等待插入结束
for(var i=1;i<=1000;i++){
    db.user.insert({
        name:i,
        age:Math.round(Math.random() * 100),
        score1:Math.round(Math.random() * 100),
        score2:Math.round(Math.random() * 100),
        score3:Math.round(Math.random() * 100),
        score4:Math.round(Math.random() * 100),
        score5:Math.round(Math.random() * 100)
    });
}
#查询user的集合状态
db.user.stats()
```

## 其他 mongodb 常用命令

```shell
###################################  注意 #################################################
# 在分片的主节点也可以同样步骤创建用户并验证，然后可以执行其他命令。
# 之后，可以在分片副本上用上一步创建的用户并验证，但是show dbs 会出现错误uncaught exception: Error: listDatabases failed:
## 当前从节点只是一个备份，不是奴隶节点，无法读取数据，写当然更不行。因为默认情况下，从节点是没有读写权限的，可以增加（删除）读的权限，rs.secondaryOk()/rs.secondaryOk(false)。
# 仲裁节点不会放任何业务数据，可以登陆查看.仲裁节点不能进行数据的查看。


#### https://www.cnblogs.com/cy0628/p/15175428.html
# 查看副本集的配置：返回包含当前副本集配置的文档。rs.conf()/rs.config()是该方法的别名。
# 查看副本集的状态：返回包含状态信息的文档，次输出结果从副本集的其他成员发送的心跳包中获得的数据反映副本集的当前状态。rs.status()
# 添加副本从节点：在主节点添加从节点，将其他成员加入到副本集。rs.add(host, arbiterOnly)
# 查看全局所有账户：db.system.users.find().pretty()
# 查看当前库下的账户：use test; show users;



#数据库 命令
db.serverStatus().connections; //连接数查看
show collections  //查看表信息
db.test_shard.find().count() //查看table1数据长度
db.test_shard.remove({}) //删除数据表
db.stats()   //查看所有的分片服务器状态
db.adminCommand( { listShards: 1 } ) //分片列表
db.test_shard.find({ age: 36 }).explain()   //精确查询
db.test_shard.find({ age: { $gte : 36 ,$lt : 73 } }).explain() //范围查询

#分片 操作命令
sh.enableSharding('testdb')                //开启数据库testdb分片
sh.shardCollection('testdb.users',{uid:1})    //按testdb.users的uid字段分片
sh.shardCollection("testdb.test_shard",{"age": 1})     //按ranged分片
sh.shardCollection("testdb.test_shard2",{"age": "hashed"}) //按hash分片
sh.status()   //查看分片节点
sh.addShard() //向集群中添加一个 shard
sh.getBalancerState()   //查看平衡器
sh.disableBalancing()   //禁用平衡器
sh.enableBalancing()    //启用平衡器
db.runCommand( { removeShard: "mongodb0" } ) //删除分片mongodb0，迁移数据查看命令
db.runCommand( { movePrimary: "test", to: "mongodb1" })   //将数据库test未分片mongodb0的数据，迁移到mongodb1主分片。
db.adminCommand("flushRouterConfig") //处理分片后，刷新路由数据。

use config
db.databases.find()  //查看所有数据库使用分片
db.settings.save({_id:"chunksize",value:1}) //将 chunk 的大小调整为 1MB
db.serverStatus().sharding


#副本集 操作命令
rs.status()   //查看成员的运行状态等信息
rs.config()    //查看配置信息
rs.slaveOk()  //允许在SECONDARY节点上进行查询操作，默认从节点不具有查询功能
rs.isMaster()  //查询该节点是否是主节点
rs.add({})   //添加新的节点到该副本集中
rs.remove()   //从副本集中删除节点
rs.stepDown //降级节点
db.printSlaveReplicationInfo()  //查看同步情况
rs.addArb("172.20.0.16:27038") //添加仲裁节点

#强制加入仲裁节点：
config=rs.conf()
config.members=[config.members[0],config.members[1],{_id:5,host:"127.0.0.1:27023",priority:5,arbiterOnly:"true"}]
rs.reconfig(config,{force:true})

#强制主节点：
cfg = rs.conf()
cfg.members[0].priority = 0.5
cfg.members[1].priority = 0.5
cfg.members[2].priority = 1
rs.reconfig(cfg)

#备份/恢复
mongodump -h 127.0.0.1:27017 -d test -o /data/backup/
mongorestore -h 127.0.0.1:27017 -d test --dir /data/db/test
```
