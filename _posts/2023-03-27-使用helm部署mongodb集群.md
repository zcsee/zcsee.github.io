---
layout: post
title: 'helm安装mongodb集群'
# subtitle: '通过shell脚本，进行redis批量迁移'
date: 2023-03-27
categories: helm
author: Jason
# cover: 'assets/img/profile.png'
tags: helm mongodb k3s
---

## 使用helm部署mongodb副本集群

## 场景

边缘服务器部署k3s服务后，需要快速部署mongodb服务

## 集群要求

1主2从的副本集群模式，开启认证模式

## 部署工具

- helm
- k3s

## 准备动作

- 确保helm已安装，如若未安装，按照下面指引进行安装

  - 在线安装方式

    1. 下载最新的安装脚本

       ```shell
       curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
       ```

    2. 给脚本添加执行权限

       ```shell
       chmod +x get_helm.sh
       ```

    3. 执行安装脚本，此脚本将自动为您的系统找到正确的二进制文件。

       ```shell
       ./get_helm.sh
       ```

    4. 通过执行[helm](https://so.csdn.net/so/search?q=helm&spm=1001.2101.3001.7020)命令来验证helm安装是否成功

       ```shell
       helm
       ```

- 加入指定的helm仓库

  ```shell
  helm repo add my-repo https://charts.bitnami.com/bitnami
  ```

- 编写values.yaml

  ```shell
  # mkdir /data/helm_mongodb && cd /data/helm_mongodb 
  # vim values.yaml
  global:
    # 按需填写storageClass，填写默认使用k3s自带的
    storageClass: ""
  # 集群方式使用replicaset，单节点的话配置成standalone
  architecture: "replicaset"
  auth:
    rootPassword: "5gTU4o7g5g"
    replicaSetKey: "mymongodb"
    usernames:
      - "user"
    passwords:
      - "123456"
    databases:
      - "message"
  # 配置副本数为3
  replicaCount: 3
  externalAccess:
    enabled: true
    service:
      externalTrafficPolicy: "Cluster"
      type: "NodePort"
      # 暴露端口数与上面的副本数一致
      nodePorts:
       - 30018
       - 30019
       - 30020
  ```

## 部署过程

1. 进入指定目录

   ```shell
   cd /data/helm_mongodb
   ```

2. helm命令指定文件部署mongodb集群

   ```shell
   $ helm install -f 1_values.yaml my-mongodb  my-repo/mongodb                                
   NAME: my-mongodb
   LAST DEPLOYED: Mon Mar 27 15:55:48 2023
   NAMESPACE: default
   STATUS: deployed
   REVISION: 1
   TEST SUITE: None
   NOTES:
   CHART NAME: mongodb
   CHART VERSION: 13.9.2
   APP VERSION: 6.0.5
   
   ** Please be patient while the chart is being deployed **
   
   MongoDB&reg; can be accessed on the following DNS name(s) and ports from within your cluster:
   
       my-mongodb-0.my-mongodb-headless.default.svc.cluster.local:27017
       my-mongodb-1.my-mongodb-headless.default.svc.cluster.local:27017
       my-mongodb-2.my-mongodb-headless.default.svc.cluster.local:27017
   
   To get the root password run:
   
       export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace default my-mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 -d)
   
   To get the password for "user" run:
   
       export MONGODB_PASSWORD=$(kubectl get secret --namespace default my-mongodb -o jsonpath="{.data.mongodb-passwords}" | base64 -d | awk -F',' '{print $1}')
   
   To connect to your database, create a MongoDB&reg; client container:
   
       kubectl run --namespace default my-mongodb-client --rm --tty -i --restart='Never' --env="MONGODB_ROOT_PASSWORD=$MONGODB_ROOT_PASSWORD" --image docker.io/bitnami/mongodb:6.0.5-debian-11-r1 --command -- bash
   
   Then, run the following command:
       mongosh admin --host "my-mongodb-0.my-mongodb-headless.default.svc.cluster.local:27017,my-mongodb-1.my-mongodb-headless.default.svc.cluster.local:27017,my-mongodb-2.my-mongodb-headless.default.svc.cluster.local:27017" --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD
   
   To connect to your database nodes from outside, you need to add both primary and secondary nodes hostnames/IPs to your Mongo client. To obtain them, follow the instructions below:
   
       MongoDB&reg; nodes domain: you can reach MongoDB&reg; nodes on any of the K8s nodes external IPs.
   
           kubectl get nodes -o wide
   
       MongoDB&reg; nodes port: You will have a different node port for each MongoDB&reg; node. You can get the list of configured node ports using the command below:
   
           echo "$(kubectl get svc --namespace default -l "app.kubernetes.io/name=mongodb,app.kubernetes.io/instance=my-mongodb,app.kubernetes.io/component=mongodb,pod" -o jsonpath='{.items[*].spec.ports[0].nodePort}' | tr ' ' '\n')"
   ```

3. 检查部署情况

   ```shell
   # 检查svc，pod，deployment情况
   $ kubectl get svc,po,deployment |grep mongo                                                    root@c71
   service/my-mongodb-0-external         NodePort       10.43.98.64    <none>        27017:30018/TCP   96s
   service/my-mongodb-1-external         NodePort       10.43.12.154   <none>        27017:30019/TCP   96s
   service/my-mongodb-2-external         NodePort       10.43.181.44   <none>        27017:30020/TCP   96s
   service/my-mongodb-headless           ClusterIP      None           <none>        27017/TCP         96s
   service/my-mongodb-arbiter-headless   ClusterIP      None           <none>        27017/TCP         96s
   pod/my-mongodb-0                               1/1     Running     0              96s
   pod/my-mongodb-arbiter-0                       1/1     Running     0              96s
   pod/my-mongodb-1                               1/1     Running     0              79s
   pod/my-mongodb-2                               1/1     Running     0              62s
   ```

## 验证过程

1. 验证可以正常登录（在宿主机上执行）

   ```shell
   $ mongosh "mongodb://root:5gTU4o7g5g@localhost:30018"                                         
   Current Mongosh Log ID: 64214ccf3cbe33818f8d1f54
   Connecting to:          mongodb://<credentials>@localhost:30018/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+1.8.0
   Using MongoDB:          6.0.5
   Using Mongosh:          1.8.0
   
   For mongosh info see: https://docs.mongodb.com/mongodb-shell/
   
      The server generated these startup warnings when booting
      2023-03-27T07:56:15.146+00:00: /sys/kernel/mm/transparent_hugepage/enabled is 'always'. We suggest setting it to 'never'
      2023-03-27T07:56:15.146+00:00: /sys/kernel/mm/transparent_hugepage/defrag is 'always'. We suggest setting it to 'never'
      2023-03-27T07:56:15.146+00:00: vm.max_map_count is too low
      Enable MongoDB's free cloud-based monitoring service, which will then receive and display
      metrics about your deployment (disk utilization, CPU, operation statistics, etc).
   
      The monitoring data will be available on a MongoDB website with a unique URL accessible to you
      and anyone you share the URL with. MongoDB may use this information to make product
      improvements and to suggest MongoDB products and deployment options to you.
   
      To enable free monitoring, run the following command: db.enableFreeMonitoring()
      To permanently disable this reminder, run the following command: db.disableFreeMonitoring()
   
   rs0 [direct: primary] test> show dbs
   admin   140.00 KiB
   config  120.00 KiB
   local   436.00 KiB
   rs0 [direct: primary] test>
   ```

2. 查看集群状态

   ```shell
   rs0 [direct: primary] admin> rs.status();
   {
     set: 'rs0',
     date: ISODate("2023-03-27T08:01:24.596Z"),
     myState: 1,
     term: Long("2"),
     syncSourceHost: '',
     syncSourceId: -1,
     heartbeatIntervalMillis: Long("2000"),
     majorityVoteCount: 2,
     writeMajorityCount: 2,
     votingMembersCount: 3,
     writableVotingMembersCount: 2,
     optimes: {
       lastCommittedOpTime: { ts: Timestamp({ t: 1679904075, i: 7 }), t: Long("2") },
       lastCommittedWallTime: ISODate("2023-03-27T08:01:15.171Z"),
       readConcernMajorityOpTime: { ts: Timestamp({ t: 1679904075, i: 7 }), t: Long("2") },
       appliedOpTime: { ts: Timestamp({ t: 1679904075, i: 7 }), t: Long("2") },
       durableOpTime: { ts: Timestamp({ t: 1679904075, i: 7 }), t: Long("2") },
       lastAppliedWallTime: ISODate("2023-03-27T08:01:15.171Z"),
       lastDurableWallTime: ISODate("2023-03-27T08:01:15.171Z")
     },
     lastStableRecoveryTimestamp: Timestamp({ t: 1679904075, i: 7 }),
     electionCandidateMetrics: {
       lastElectionReason: 'electionTimeout',
       lastElectionDate: ISODate("2023-03-27T07:56:15.173Z"),
       electionTerm: Long("2"),
       lastCommittedOpTimeAtElection: { ts: Timestamp({ t: 0, i: 0 }), t: Long("-1") },
       lastSeenOpTimeAtElection: { ts: Timestamp({ t: 1679903770, i: 14 }), t: Long("1") },
       numVotesNeeded: 1,
       priorityAtElection: 5,
       electionTimeoutMillis: Long("10000"),
       newTermStartDate: ISODate("2023-03-27T07:56:15.175Z"),
       wMajorityWriteAvailabilityDate: ISODate("2023-03-27T07:56:15.177Z")
     },
     members: [
       {
         _id: 0,
         name: '192.168.216.3:30018',
         health: 1,
         state: 1,
         stateStr: 'PRIMARY',
         uptime: 310,
         optime: { ts: Timestamp({ t: 1679904075, i: 7 }), t: Long("2") },
         optimeDate: ISODate("2023-03-27T08:01:15.000Z"),
         lastAppliedWallTime: ISODate("2023-03-27T08:01:15.171Z"),
         lastDurableWallTime: ISODate("2023-03-27T08:01:15.171Z"),
         syncSourceHost: '',
         syncSourceId: -1,
         infoMessage: '',
         electionTime: Timestamp({ t: 1679903775, i: 1 }),
         electionDate: ISODate("2023-03-27T07:56:15.000Z"),
         configVersion: 6,
         configTerm: 2,
         self: true,
         lastHeartbeatMessage: ''
       },
       {
         _id: 1,
         name: '192.168.216.3:30019',
         health: 1,
         state: 2,
         stateStr: 'SECONDARY',
         uptime: 235,
         optime: { ts: Timestamp({ t: 1679904075, i: 7 }), t: Long("2") },
         optimeDurable: { ts: Timestamp({ t: 1679904075, i: 7 }), t: Long("2") },
         optimeDate: ISODate("2023-03-27T08:01:15.000Z"),
         optimeDurableDate: ISODate("2023-03-27T08:01:15.000Z"),
         lastAppliedWallTime: ISODate("2023-03-27T08:01:15.171Z"),
         lastDurableWallTime: ISODate("2023-03-27T08:01:15.171Z"),
         lastHeartbeat: ISODate("2023-03-27T08:01:23.451Z"),
         lastHeartbeatRecv: ISODate("2023-03-27T08:01:24.041Z"),
         pingMs: Long("0"),
         lastHeartbeatMessage: '',
         syncSourceHost: '192.168.216.3:30018',
         syncSourceId: 0,
         infoMessage: '',
         configVersion: 6,
         configTerm: 2
       },
       {
         _id: 2,
         name: 'my-mongodb-arbiter-0.my-mongodb-arbiter-headless.default.svc.cluster.local:27017',
         health: 1,
         state: 7,
         stateStr: 'ARBITER',
         uptime: 273,
         lastHeartbeat: ISODate("2023-03-27T08:01:23.440Z"),
         lastHeartbeatRecv: ISODate("2023-03-27T08:01:24.056Z"),
         pingMs: Long("0"),
         lastHeartbeatMessage: '',
         syncSourceHost: '',
         syncSourceId: -1,
         infoMessage: '',
         configVersion: 6,
         configTerm: 2
       }
     ],
     ok: 1,
     '$clusterTime': {
       clusterTime: Timestamp({ t: 1679904075, i: 7 }),
       signature: {
         hash: Binary(Buffer.from("32a27425bfdc46cb14263701fc4d7702dcf4dac0", "hex"), 0),
         keyId: Long("7215131752577105924")
       }
     },
     operationTime: Timestamp({ t: 1679904075, i: 7 })
   }
   rs0 [direct: primary] admin>
   ```

3. 发现id为2的member不符合预期的secondary，而是一个arbiter，此处使用替换方法，将arbiter替换成原定的secondary

   ```shell
   cfg = rs.conf()
   cfg.members[2].host = "192.168.216.3:30020"
   rs.reconfig(cfg)
   ```

## 参考文档

[通过helm部署mongodb的3种部署方式](https://blog.csdn.net/weixin_42555971/article/details/126243336)

[bitnami官网](https://artifacthub.io/packages/helm/bitnami/mongodb)

## 遗留问题

- 在集群的主节点上，通过 rs.status()查看节点状态，发现本来id 2应该是secondary的，但实际上是arbiter的配置

  经过多次反复部署发现，节点的数量与上一次部署时一致，怀疑是残留的配置导致的，**待证实**

## mongodb常用命令

1. 增加节点，可参考[mongodb官网增加节点](https://www.mongodb.com/docs/manual/tutorial/expand-replica-set/)部分

   ```shell
   rs.add( { host: "mongodb3.example.net:27017" } )
   ```

2. 删除节点，可参考[mongodb官网删除节点](https://www.mongodb.com/docs/manual/tutorial/remove-replica-set-member/)部分

   ```shell
   rs.remove("mongod3.example.net:27017")
   ```

3. 查看集群相关

   ```shell
   #查看集群状态
   rs.status()
   # 查看集群相关配置
   rs.conf()
   ```

4. 替换节点，可参考[mongodb官网替换节点](https://www.mongodb.com/docs/manual/tutorial/replace-replica-set-member/)部分

   ```shell
   cfg = rs.conf()
   cfg.members[0].host = "mongo2.example.net"
   rs.reconfig(cfg)
   ```
