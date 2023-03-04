---
layout: post
title: 'k3s部署mongodb副本集集群'
date: 2023-02-28
categories: k3s
author: Jason
tags: k3s mongodb 集群
---

## 部署nfs作为存储

### 通过ansible一键部署nfs服务，需要调整参数

> /etc/ansible/install_nfs.yaml  

```yaml
- hosts: 192.168.216.5
  tasks:
    - name: 01-install nfs rpc
      yum:
        name: ['nfs-utils','rpcbind']
        state: installed
    - name: 02-make sure nfs directory exists
      file: path="/nfs" state=directory
    - name: 03-create nfs conf file
      copy: content="/nfs   192.168.216.0/24(rw,sync)" dest=/etc/exports
    - name: 04-create user nfsnobody
      user: name=nfsnobody create_home=no shell=/sbin/nologin
    - name: 05-create /nfs
      file: path=/nfs state=directory owner=nfsnobody group=nfsnobody
    - name: 06-start rpc nfs
      service: name={{ item }} state=started enabled=yes
      with_items:
        - rpcbind
        - nfs

- hosts: 192.168.216.3
  vars:
    data_dir: /nfs
  tasks:
    - name: 01-install nfs
      yum: name=nfs-utils state=installed
    - name: 02-test mounted
      mount: src=192.168.216.5:{{ data_dir }} path=/mnt fstype=nfs state=mounted
    - name: 03-check mount info
      shell: df -h|grep /nfs
      register: mount_info
    - name: 04-display mount info
      debug: msg={{ mount_info.stdout_lines }}
    - name: 05-resume /mnt
      mount: src=192.168.216.5:{{ data_dir }} path=/mnt fstype=nfs state=unmounted
```

> 如果出现`mount.nfs: No route to host`，请检查防火墙是否关闭

### 部署k3s的插件 nfs-subdir-external-provisioner

```shell

# 创建nfs插件的目录
mkdir -p /data/nfs-subdir-external-provisioner/deploy
cd /data/nfs-subdir-external-provisioner/deploy
# 按照环境的NFS参数修改后使用，主要是SERVER HOST，SHARE PATH

# 修改class.yaml文件中的name为"nfs-client"，archiveOnDelete为"true"以储存数据
# class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "true"
# ==========================================================================================
# 修改<NFS_SERVER_HOST>和<NFS_SERVER_DIR>为对应值
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: <NFS_SERVER_HOST>
            - name: NFS_PATH
              value: <NFS_SERVER_DIR>
      volumes:
        - name: nfs-client-root
          nfs:
            server: <NFS_SERVER_HOST>
            path: <NFS_SERVER_DIR>
#=============================================================================================
# rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
# =============================================================================================
# 部署
cd /data/mongodb-cluster/ && ls | xargs -i kubectl apply -f {}

# 检查部署的情况
kubectl get deployment,pod,svc,sc -n default

➜ kubectl get deployment,pod,svc,sc -n default
NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-client-provisioner    1/1     1            1           82m

NAME                                           READY   STATUS      RESTARTS       AGE
pod/nfs-client-provisioner-97bc6b95-74ghx      1/1     Running     0              82m

NAME                          TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
service/kubernetes            ClusterIP      10.43.0.1     <none>        443/TCP        7d

NAME                                               PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/nfs-client    k8s-sigs.io/nfs-subdir-external-provisioner   Delete          Immediate              false 

# 测试nfs插件是否正常运行
# test-claim.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
# test-pod.yaml
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: busybox:stable
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim
```

## 2.MongoDB集群部署

### 2.1 创建mongodb-cluster命名空间

```shell
mkdir -p /data/mongodb-cluster/
cd /data/mongodb-cluster/
kubectl create namespace mongodb-cluster
```

### 2.2 从秘钥文件创建secret

```shell
# 秘钥文件用于mongodb副本集之间进行数据复制时免密码，MongoDB将使用此密钥与内部集群通信。` `kubectl create secret generic: 表示根据配置文件、目录或指定的literal-value创建secret。
openssl rand -base64 741 > ./key.txt
kubectl create secret generic mongodb-replica-sets-key -n mongodb-cluster \
--from-file=internal-auth-mongodb-keyfile=./key.txt
```

### 2.3 编写部署脚本 

```shell
vim mongodb-statefulset.yaml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-cluster
  namespace: mongodb-cluster
spec:
  serviceName: mongodb-cluster
  replicas: 3
  selector:
    matchLabels:
      role: mongo
      environment: produce
      replicaset: MainRepSet
  template:
    metadata:
      labels:
        role: mongo
        environment: produce
        replicaset: MainRepSet
    spec:
      containers:
      - name: mongodb-container
        image: registry.cn-hangzhou.aliyuncs.com/k8s-image01/mongodb:4.2.21-bionic
        command:
        - "numactl"
        - "--interleave=all"
        - "mongod"
        - "--bind_ip"
        - "0.0.0.0"
        - "--replSet"
        - "MainRepSet"
        - "--auth"
        - "--clusterAuthMode"
        - "keyFile"
        - "--keyFile"
        - "/etc/secrets-volume/internal-auth-mongodb-keyfile"
        - "--setParameter"
        - "authenticationMechanisms=SCRAM-SHA-1"
        resources:
          requests:
            cpu: 0.5
            memory: 500Mi
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: secrets-volume
          readOnly: true
          mountPath: /etc/secrets-volume
        - name: mongodb-persistent-storage-claim
          mountPath: /data/db
      volumes:
      - name: secrets-volume
        secret:
          secretName: mongodb-replica-sets-key
          defaultMode: 256
  volumeClaimTemplates:
  - metadata:
      name: mongodb-persistent-storage-claim
      #annotations:
      #  volume.beta.kubernetes.io/storage-class: "standard"
    spec:
      storageClassName: "nfs-client"
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-cluster
  namespace: mongodb-cluster
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-cs
  namespace: mongodb-cluster
  labels:
    name: mongo
spec:
  ports:
    - port: 27017
      targetPort: 27017
      nodePort: 30717
  selector:
    role: mongo
  type: NodePort
```

### 2.4部署3节点的MongoDB集群

```shell
kubectl apply -f mongodb-statefulset.yaml
```

## 3 查看MongoDB集群状态

## 3.1查看pod

```shell
➜  ~ kubectl get statefulset/mongodb-cluster -n mongodb-cluster -o wide
NAME              READY   AGE   CONTAINERS          IMAGES
mongodb-cluster   3/3     45m   mongodb-container   registry.cn-hangzhou.aliyuncs.com/k8s-image01/mongodb:4.2.21-bionic
➜  ~ kubectl get pod -n mongodb-cluster -o wide
NAME                READY   STATUS    RESTARTS   AGE   IP          NODE   NOMINATED NODE   READINESS GATES
mongodb-cluster-0   1/1     Running   0          45m   10.42.0.4   c71    <none>           <none>
mongodb-cluster-1   1/1     Running   0          45m   10.42.0.5   c71    <none>           <none>
mongodb-cluster-2   1/1     Running   0          45m   10.42.0.6   c71    <none>           <none>

```



## 3.2 查看pvc

```shell
➜  ~ kubectl get pvc -n mongodb-cluster -o wide
NAME                                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE    VOLUMEMODE
mongodb-persistent-storage-claim-mongodb-cluster-0   Bound    pvc-4316e16b-ec91-48ae-a821-0a65c07f5d50   1Gi        RWO            nfs-client   114m   Filesystem
mongodb-persistent-storage-claim-mongodb-cluster-1   Bound    pvc-9584e76f-a667-4a51-9446-7ad431782b50   1Gi        RWO            nfs-client   45m    Filesystem
mongodb-persistent-storage-claim-mongodb-cluster-2   Bound    pvc-468d3954-7f55-4b8d-bbc4-52f25c049141   1Gi        RWO            nfs-client   45m    Filesystem

```



### 3.3 查看pv

```shell
➜  ~ kubectl get pv -o wide
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                                STORAGECLASS          REASON   AGE   VOLUMEMODE
pvc-4316e16b-ec91-48ae-a821-0a65c07f5d50   1Gi        RWO            Delete           Bound    mongodb-cluster/mongodb-persistent-storage-claim-mongodb-cluster-0   nfs-client            73m   Filesystem
pvc-9584e76f-a667-4a51-9446-7ad431782b50   1Gi        RWO            Delete           Bound    mongodb-cluster/mongodb-persistent-storage-claim-mongodb-cluster-1   nfs-client            45m   Filesystem
pvc-468d3954-7f55-4b8d-bbc4-52f25c049141   1Gi        RWO            Delete           Bound    mongodb-cluster/mongodb-persistent-storage-claim-mongodb-cluster-2   nfs-client            45m   Filesystem
```



### 3.4 查看nfs的存储路径

```shell
[root@c73 nfs]# ls -l /nfs/
drwxrwxrwx. 4 nfsnobody nfsnobody 4096 Feb 28 17:00 mongodb-cluster-mongodb-persistent-storage-claim-mongodb-cluster-0-pvc-4316e16b-ec91-48ae-a821-0a65c07f5d50
drwxrwxrwx. 4 nfsnobody nfsnobody 4096 Feb 28 17:00 mongodb-cluster-mongodb-persistent-storage-claim-mongodb-cluster-1-pvc-9584e76f-a667-4a51-9446-7ad431782b50
drwxrwxrwx. 4 nfsnobody nfsnobody 4096 Feb 28 17:00 mongodb-cluster-mongodb-persistent-storage-claim-mongodb-cluster-2-pvc-468d3954-7f55-4b8d-bbc4-52f25c049141

```



### 3.5 查看svc相关信息

```shell
➜  ~ kubectl get svc/mongodb-cluster -n mongodb-cluster -o wide
NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE   SELECTOR
mongodb-cluster   ClusterIP   None         <none>        27017/TCP   43m   role=mongo

➜  ~ kubectl get ep/mongodb-cluster -n mongodb-cluster -o wide
NAME              ENDPOINTS                                         AGE
mongodb-cluster   10.42.0.4:27017,10.42.0.5:27017,10.42.0.6:27017   43m

```



### 3.6 查看mongodb域名解析

```shell
➜  nfs-subdir-external-provisioner-master kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm /bin/sh
If you don't see a command prompt, try pressing enter.
/ # nslookup mongodb-cluster.mongodb-cluster.svc.cluster.local
Server:    10.43.0.10
Address 1: 10.43.0.10 kube-dns.kube-system.svc.cluster.local

Name:      mongodb-cluster.mongodb-cluster.svc.cluster.local
Address 1: 10.42.0.6 mongodb-cluster-2.mongodb-cluster.mongodb-cluster.svc.cluster.local
Address 2: 10.42.0.5 mongodb-cluster-1.mongodb-cluster.mongodb-cluster.svc.cluster.local
Address 3: 10.42.0.4 mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local
```

## 4 mongodb复制集配置

```
# 我们需要连接到"mongodb"容器进程之一来配置副本集，运行以下命令以连接到第一个容器，在shell中启动副本集，由于使用了StatefulSet，我们可以依赖主机名始终相同

➜  ~ kubectl exec -it pod/mongodb-cluster-0 -n mongodb-cluster -- bash
root@mongodb-cluster-0:/# mongo
MongoDB shell version v4.2.21
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("f48c0f37-430d-4ab1-a674-a6270f8e7bec") }
MongoDB server version: 4.2.21
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
        https://docs.mongodb.com/
Questions? Try the MongoDB Developer Community Forums
        https://community.mongodb.com
> use admin
switched to db admin
> rs.initiate({_id: "MainRepSet", version: 1, members: [
...       { _id: 0, host : "mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017" },
...       { _id: 1, host : "mongodb-cluster-1.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017" },
...       { _id: 2, host : "mongodb-cluster-2.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017" }
... ]});
{ "ok" : 1 }

```

### 检查mongodb副本集的状态

```shell
检查mongodb副本集的状态，直到mongodb副本集完全初始化并且存在一个主副本和两个辅助副本

MainRepSet:SECONDARY> rs.status();
{
        "set" : "MainRepSet",
        "date" : ISODate("2023-02-28T09:47:42.548Z"),
        "myState" : 1,
        "term" : NumberLong(1),
        "syncingTo" : "",
        "syncSourceHost" : "",
        "syncSourceId" : -1,
        "heartbeatIntervalMillis" : NumberLong(2000),
        "majorityVoteCount" : 2,
        "writeMajorityCount" : 2,
        "optimes" : {
                "lastCommittedOpTime" : {
                        "ts" : Timestamp(1677577660, 1),
                        "t" : NumberLong(1)
                },
                "lastCommittedWallTime" : ISODate("2023-02-28T09:47:40.181Z"),
                "readConcernMajorityOpTime" : {
                        "ts" : Timestamp(1677577660, 1),
                        "t" : NumberLong(1)
                },
                "readConcernMajorityWallTime" : ISODate("2023-02-28T09:47:40.181Z"),
                "appliedOpTime" : {
                        "ts" : Timestamp(1677577660, 1),
                        "t" : NumberLong(1)
                },
                "durableOpTime" : {
                        "ts" : Timestamp(1677577660, 1),
                        "t" : NumberLong(1)
                },
                "lastAppliedWallTime" : ISODate("2023-02-28T09:47:40.181Z"),
                "lastDurableWallTime" : ISODate("2023-02-28T09:47:40.181Z")
        },
        "lastStableRecoveryTimestamp" : Timestamp(1677577620, 4),
        "lastStableCheckpointTimestamp" : Timestamp(1677577620, 4),
        "electionCandidateMetrics" : {
                "lastElectionReason" : "electionTimeout",
                "lastElectionDate" : ISODate("2023-02-28T09:47:00.109Z"),
                "electionTerm" : NumberLong(1),
                "lastCommittedOpTimeAtElection" : {
                        "ts" : Timestamp(0, 0),
                        "t" : NumberLong(-1)
                },
                "lastSeenOpTimeAtElection" : {
                        "ts" : Timestamp(1677577609, 1),
                        "t" : NumberLong(-1)
                },
                "numVotesNeeded" : 2,
                "priorityAtElection" : 1,
                "electionTimeoutMillis" : NumberLong(10000),
                "numCatchUpOps" : NumberLong(0),
                "newTermStartDate" : ISODate("2023-02-28T09:47:00.175Z"),
                "wMajorityWriteAvailabilityDate" : ISODate("2023-02-28T09:47:00.705Z")
        },
        "members" : [
                {
                        "_id" : 0,
                        "name" : "mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 3038,
                        "optime" : {
                                "ts" : Timestamp(1677577660, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2023-02-28T09:47:40Z"),
                        "syncingTo" : "",
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "could not find member to sync from",
                        "electionTime" : Timestamp(1677577620, 1),
                        "electionDate" : ISODate("2023-02-28T09:47:00Z"),
                        "configVersion" : 1,
                        "self" : true,
                        "lastHeartbeatMessage" : ""
                },
                {
                        "_id" : 1,
                        "name" : "mongodb-cluster-1.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 52,
                        "optime" : {
                                "ts" : Timestamp(1677577660, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1677577660, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2023-02-28T09:47:40Z"),
                        "optimeDurableDate" : ISODate("2023-02-28T09:47:40Z"),
                        "lastHeartbeat" : ISODate("2023-02-28T09:47:42.152Z"),
                        "lastHeartbeatRecv" : ISODate("2023-02-28T09:47:40.733Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncingTo" : "mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "syncSourceHost" : "mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "syncSourceId" : 0,
                        "infoMessage" : "",
                        "configVersion" : 1
                },
                {
                        "_id" : 2,
                        "name" : "mongodb-cluster-2.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 52,
                        "optime" : {
                                "ts" : Timestamp(1677577660, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1677577660, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2023-02-28T09:47:40Z"),
                        "optimeDurableDate" : ISODate("2023-02-28T09:47:40Z"),
                        "lastHeartbeat" : ISODate("2023-02-28T09:47:42.152Z"),
                        "lastHeartbeatRecv" : ISODate("2023-02-28T09:47:40.703Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncingTo" : "mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "syncSourceHost" : "mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "syncSourceId" : 0,
                        "infoMessage" : "",
                        "configVersion" : 1
                }
        ],
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1677577660, 1),
                "signature" : {
                        "hash" : BinData(0,"fdP+AZ422ekE1fmEPxDNDso43SU="),
                        "keyId" : NumberLong("7205141014401515524")
                }
        },
        "operationTime" : Timestamp(1677577660, 1)
}
MainRepSet:PRIMARY>
```

### 4.3 配置"admin"用户

```
# 执行此操作会导致"mongodb匿名登录"被自动永久禁用
MainRepSet:PRIMARY> db.getSiblingDB("admin").createUser({
...       user : "root",
...       pwd  : "jason1989",
...       roles: [ { role: "root", db: "admin" } ]
... });
Successfully added user: {
        "user" : "root",
        "roles" : [
                {
                        "role" : "root",
                        "db" : "admin"
                }
        ]
}
MainRepSet:PRIMARY>
# 此时已需要登录才能执行命令
MainRepSet:PRIMARY> rs.status();
{
        "operationTime" : Timestamp(1677577776, 4),
        "ok" : 0,
        "errmsg" : "command replSetGetStatus requires authentication",
        "code" : 13,
        "codeName" : "Unauthorized",
        "$clusterTime" : {
                "clusterTime" : Timestamp(1677577790, 1),
                "signature" : {
                        "hash" : BinData(0,"bEzjKYWtyeVvxDnH2dF3qms/s9w="),
                        "keyId" : NumberLong("7205141014401515524")
                }
        }
}
MainRepSet:PRIMARY>

```

### 4.4 验证登录mongodb副本集

```
root@mongodb-cluster-0:/# mongo --host mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local \
> --port 27017 -uroot -p'jason1989' admin
MongoDB shell version v4.2.21
connecting to: mongodb://mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017/admin?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("41d5dd97-1313-4468-9db3-79d4ef1a7309") }
MongoDB server version: 4.2.21
Server has startup warnings:
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten] ** WARNING: You are running this process as the root user, which is not recommended.
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten]
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten]
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/enabled is 'always'.
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten] **        We suggest setting it to 'never'
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten]
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/defrag is 'always'.
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten] **        We suggest setting it to 'never'
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten]
---
Enable MongoDB's free cloud-based monitoring service, which will then receive and display
metrics about your deployment (disk utilization, CPU, operation statistics, etc).

The monitoring data will be available on a MongoDB website with a unique URL accessible to you
and anyone you share the URL with. MongoDB may use this information to make product
improvements and to suggest MongoDB products and deployment options to you.

To enable free monitoring, run the following command: db.enableFreeMonitoring()
To permanently disable this reminder, run the following command: db.disableFreeMonitoring()
---

MainRepSet:PRIMARY> db.getName();
admin
MainRepSet:PRIMARY>


# 验证不同的登录方式（非必须）
root@mongodb-cluster-0:/# mongo
MongoDB shell version v4.2.21
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("e432ba1b-7184-44ce-ad1b-c7631d4349f2") }
MongoDB server version: 4.2.21
MainRepSet:PRIMARY> db.getSiblingDB('admin').auth("root", "jason1989");
1
MainRepSet:PRIMARY> db.getName();
test
MainRepSet:PRIMARY>

```

## 5 mongodb复制集测试

### 5.1 将数据插入 mongodb-cluster-0 pod

```
root@mongodb-cluster-0:/# mongo --host mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local \
> --port 27017 -uroot -p'jason1989' admin
MongoDB shell version v4.2.21
connecting to: mongodb://mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017/admin?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("7c375fce-4f20-472f-8ea4-5bb763b70eb4") }
MongoDB server version: 4.2.21
Server has startup warnings:
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten] ** WARNING: You are running this process as the root user, which is not recommended.
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten]
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten]
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/enabled is 'always'.
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten] **        We suggest setting it to 'never'
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten]
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/defrag is 'always'.
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten] **        We suggest setting it to 'never'
2023-02-28T08:57:05.814+0000 I  CONTROL  [initandlisten]
---
Enable MongoDB's free cloud-based monitoring service, which will then receive and display
metrics about your deployment (disk utilization, CPU, operation statistics, etc).

The monitoring data will be available on a MongoDB website with a unique URL accessible to you
and anyone you share the URL with. MongoDB may use this information to make product
improvements and to suggest MongoDB products and deployment options to you.

To enable free monitoring, run the following command: db.enableFreeMonitoring()
To permanently disable this reminder, run the following command: db.disableFreeMonitoring()
---

MainRepSet:PRIMARY> use master_slave_test
switched to db master_slave_test
MainRepSet:PRIMARY> function add(){var i = 0;for(;i<7;i++){db.persons.insert({"name":"master_slave_test"+i})}}
MainRepSet:PRIMARY> add()
MainRepSet:PRIMARY> db.persons.find()
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c309d"), "name" : "master_slave_test0" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c309e"), "name" : "master_slave_test1" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c309f"), "name" : "master_slave_test2" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c30a0"), "name" : "master_slave_test3" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c30a1"), "name" : "master_slave_test4" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c30a2"), "name" : "master_slave_test5" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c30a3"), "name" : "master_slave_test6" }
MainRepSet:PRIMARY>

```

### 5.2 在 mongodb-cluster-1 pod 中验证数据

```
root@mongodb-cluster-0:/# mongo --host mongodb-cluster-1.mongodb-cluster.mongodb-cluster.svc.cluster.local \
> --port 27017 -uroot -p'jason1989' admin
MongoDB shell version v4.2.21
connecting to: mongodb://mongodb-cluster-1.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017/admin?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("0f4a6e37-8a9d-4890-b94c-eb4f6e66eb2a") }
MongoDB server version: 4.2.21
Server has startup warnings:
2023-02-28T08:57:08.896+0000 I  CONTROL  [initandlisten] ** WARNING: You are running this process as the root user, which is not recommended.
2023-02-28T08:57:08.896+0000 I  CONTROL  [initandlisten]
2023-02-28T08:57:08.896+0000 I  CONTROL  [initandlisten]
2023-02-28T08:57:08.896+0000 I  CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/enabled is 'always'.
2023-02-28T08:57:08.896+0000 I  CONTROL  [initandlisten] **        We suggest setting it to 'never'
2023-02-28T08:57:08.896+0000 I  CONTROL  [initandlisten]
2023-02-28T08:57:08.896+0000 I  CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/defrag is 'always'.
2023-02-28T08:57:08.896+0000 I  CONTROL  [initandlisten] **        We suggest setting it to 'never'
2023-02-28T08:57:08.896+0000 I  CONTROL  [initandlisten]
---
Enable MongoDB's free cloud-based monitoring service, which will then receive and display
metrics about your deployment (disk utilization, CPU, operation statistics, etc).

The monitoring data will be available on a MongoDB website with a unique URL accessible to you
and anyone you share the URL with. MongoDB may use this information to make product
improvements and to suggest MongoDB products and deployment options to you.

To enable free monitoring, run the following command: db.enableFreeMonitoring()
To permanently disable this reminder, run the following command: db.disableFreeMonitoring()
---

MainRepSet:SECONDARY> db.getMongo().setSecondaryOk()
MainRepSet:SECONDARY> use master_slave_test
switched to db master_slave_test
MainRepSet:SECONDARY> db.persons.find()
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c309d"), "name" : "master_slave_test0" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c309e"), "name" : "master_slave_test1" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c309f"), "name" : "master_slave_test2" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c30a1"), "name" : "master_slave_test4" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c30a0"), "name" : "master_slave_test3" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c30a2"), "name" : "master_slave_test5" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c30a3"), "name" : "master_slave_test6" }
MainRepSet:SECONDARY> exit
bye
root@mongodb-cluster-0:/#
```

### 5.3 验证PVC

```
1 删除整个mongodb副本集的pod
➜ kubectl delete -f /data/mongodb-cluster/mongodb-statefulset.yaml
service "mongodb-cluster" deleted
statefulset.apps "mongodb-cluster" deleted
➜ kubectl get all -n mongodb-cluster -o wide
No resources found in mongodb-cluster namespace.

# 注意:
这里只是删除了mongodb副本集的pod和service，对应的pvc、pv、存储并没有被删除。 
当重建pod时会重新占用对应的pvc, 因为pvc中使用了pod的名字，而statefulset又保持了pod的名称固定不变

2 重新创建mongodb副本集的pod
➜ kubectl apply -f /data/mongodb-cluster/mongodb-statefulset.yaml
service/mongodb-cluster created
statefulset.apps/mongodb-cluster created

➜ kubectl get all -n mongodb-cluster -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE   NOMINATED NODE   READINESS GATES
pod/mongodb-cluster-0   1/1     Running   0          18s   10.42.0.68   c71    <none>           <none>
pod/mongodb-cluster-1   1/1     Running   0          17s   10.42.0.69   c71    <none>           <none>
pod/mongodb-cluster-2   1/1     Running   0          15s   10.42.0.70   c71    <none>           <none>

NAME                      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE   SELECTOR
service/mongodb-cluster   ClusterIP   None         <none>        27017/TCP   18s   role=mongo

NAME                               READY   AGE   CONTAINERS          IMAGES
statefulset.apps/mongodb-cluster   3/3     18s   mongodb-container   registry.cn-hangzhou.aliyuncs.com/k8s-image01/mongodb:4.2.21-bionic

3 验证mongodb副本集的数据
➜  ~ kubectl exec -it pod/mongodb-cluster-0 -n mongodb-cluster -- bash
root@mongodb-cluster-0:/# mongo --host mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local \
> --port 27017 -uroot -p'jason1989' admin
MongoDB shell version v4.2.21
connecting to: mongodb://mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017/admin?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("ba1522d8-cf74-4579-9fcf-25b1f4589a6c") }
MongoDB server version: 4.2.21
Server has startup warnings:
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] ** WARNING: You are running this process as the root user, which is not recommended.
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten]
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten]
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/enabled is 'always'.
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] **        We suggest setting it to 'never'
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten]
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/defrag is 'always'.
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] **        We suggest setting it to 'never'
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten]
---
Enable MongoDB's free cloud-based monitoring service, which will then receive and display
metrics about your deployment (disk utilization, CPU, operation statistics, etc).

The monitoring data will be available on a MongoDB website with a unique URL accessible to you
and anyone you share the URL with. MongoDB may use this information to make product
improvements and to suggest MongoDB products and deployment options to you.

To enable free monitoring, run the following command: db.enableFreeMonitoring()
To permanently disable this reminder, run the following command: db.disableFreeMonitoring()
---

MainRepSet:SECONDARY> use master_slave_test
switched to db master_slave_test
MainRepSet:SECONDARY> db.persons.find()
Error: error: {
        "operationTime" : Timestamp(1677578635, 1),
        "ok" : 0,
        "errmsg" : "not master and slaveOk=false",
        "code" : 13435,
        "codeName" : "NotPrimaryNoSecondaryOk",
        "$clusterTime" : {
                "clusterTime" : Timestamp(1677578635, 1),
                "signature" : {
                        "hash" : BinData(0,"/hBmapWMme7BBWBuBD/VdWa1r2c="),
                        "keyId" : NumberLong("7205141014401515524")
                }
        }
}
MainRepSet:SECONDARY> db.getMongo().setSecondaryOk()
MainRepSet:SECONDARY> db.persons.find()
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c309d"), "name" : "master_slave_test0" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c309e"), "name" : "master_slave_test1" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c309f"), "name" : "master_slave_test2" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c30a0"), "name" : "master_slave_test3" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c30a1"), "name" : "master_slave_test4" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c30a2"), "name" : "master_slave_test5" }
{ "_id" : ObjectId("63fdcf3eeb7eba2e493c30a3"), "name" : "master_slave_test6" }
MainRepSet:SECONDARY>

```

### 5.4 验证集群化

```
# 删除 mongodb-cluster-0 pod 并继续检查 rs.status()，最终剩下的两个节点中的一个节点将成为 PRIMARY
➜  ~ kubectl exec -it pod/mongodb-cluster-0 -n mongodb-cluster -- bash
root@mongodb-cluster-0:/# mongo --host mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local --port 27017 -uroot -p'jason1989' admin
MongoDB shell version v4.2.21
connecting to: mongodb://mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017/admin?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("273bdf86-7a70-4591-8415-a20dbbb124b5") }
MongoDB server version: 4.2.21
Server has startup warnings:
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] ** WARNING: You are running this process as the root user, which is not recommended.
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten]
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten]
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/enabled is 'always'.
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] **        We suggest setting it to 'never'
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten]
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/defrag is 'always'.
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] **        We suggest setting it to 'never'
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten]
---
Enable MongoDB's free cloud-based monitoring service, which will then receive and display
metrics about your deployment (disk utilization, CPU, operation statistics, etc).

The monitoring data will be available on a MongoDB website with a unique URL accessible to you
and anyone you share the URL with. MongoDB may use this information to make product
improvements and to suggest MongoDB products and deployment options to you.

To enable free monitoring, run the following command: db.enableFreeMonitoring()
To permanently disable this reminder, run the following command: db.disableFreeMonitoring()
---

MainRepSet:SECONDARY> rs.status()
{
        "set" : "MainRepSet",
        "date" : ISODate("2023-02-28T10:06:05.336Z"),
        "myState" : 2,
        "term" : NumberLong(4),
        "syncingTo" : "mongodb-cluster-2.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
        "syncSourceHost" : "mongodb-cluster-2.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
        "syncSourceId" : 2,
        "heartbeatIntervalMillis" : NumberLong(2000),
        "majorityVoteCount" : 2,
        "writeMajorityCount" : 2,
        "optimes" : {
                "lastCommittedOpTime" : {
                        "ts" : Timestamp(1677578756, 1),
                        "t" : NumberLong(4)
                },
                "lastCommittedWallTime" : ISODate("2023-02-28T10:05:56.024Z"),
                "readConcernMajorityOpTime" : {
                        "ts" : Timestamp(1677578756, 1),
                        "t" : NumberLong(4)
                },
                "readConcernMajorityWallTime" : ISODate("2023-02-28T10:05:56.024Z"),
                "appliedOpTime" : {
                        "ts" : Timestamp(1677578756, 1),
                        "t" : NumberLong(4)
                },
                "durableOpTime" : {
                        "ts" : Timestamp(1677578756, 1),
                        "t" : NumberLong(4)
                },
                "lastAppliedWallTime" : ISODate("2023-02-28T10:05:56.024Z"),
                "lastDurableWallTime" : ISODate("2023-02-28T10:05:56.024Z")
        },
        "lastStableRecoveryTimestamp" : Timestamp(1677578756, 1),
        "lastStableCheckpointTimestamp" : Timestamp(1677578756, 1),
        "electionParticipantMetrics" : {
                "votedForCandidate" : true,
                "electionTerm" : NumberLong(4),
                "lastVoteDate" : ISODate("2023-02-28T10:01:25.749Z"),
                "electionCandidateMemberId" : 1,
                "voteReason" : "",
                "lastAppliedOpTimeAtElection" : {
                        "ts" : Timestamp(1677578463, 1),
                        "t" : NumberLong(3)
                },
                "maxAppliedOpTimeInSet" : {
                        "ts" : Timestamp(1677578463, 1),
                        "t" : NumberLong(3)
                },
                "priorityAtElection" : 1,
                "newTermStartDate" : ISODate("2023-02-28T10:01:25.761Z"),
                "newTermAppliedDate" : ISODate("2023-02-28T10:01:27.260Z")
        },
        "members" : [
                {
                        "_id" : 0,
                        "name" : "mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 491,
                        "optime" : {
                                "ts" : Timestamp(1677578756, 1),
                                "t" : NumberLong(4)
                        },
                        "optimeDate" : ISODate("2023-02-28T10:05:56Z"),
                        "syncingTo" : "mongodb-cluster-2.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "syncSourceHost" : "mongodb-cluster-2.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "syncSourceId" : 2,
                        "infoMessage" : "",
                        "configVersion" : 1,
                        "self" : true,
                        "lastHeartbeatMessage" : ""
                },
                {
                        "_id" : 1,
                        "name" : "mongodb-cluster-1.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 286,
                        "optime" : {
                                "ts" : Timestamp(1677578756, 1),
                                "t" : NumberLong(4)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1677578756, 1),
                                "t" : NumberLong(4)
                        },
                        "optimeDate" : ISODate("2023-02-28T10:05:56Z"),
                        "optimeDurableDate" : ISODate("2023-02-28T10:05:56Z"),
                        "lastHeartbeat" : ISODate("2023-02-28T10:06:03.981Z"),
                        "lastHeartbeatRecv" : ISODate("2023-02-28T10:06:04.400Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncingTo" : "",
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "",
                        "electionTime" : Timestamp(1677578485, 1),
                        "electionDate" : ISODate("2023-02-28T10:01:25Z"),
                        "configVersion" : 1
                },
                {
                        "_id" : 2,
                        "name" : "mongodb-cluster-2.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 286,
                        "optime" : {
                                "ts" : Timestamp(1677578756, 1),
                                "t" : NumberLong(4)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1677578756, 1),
                                "t" : NumberLong(4)
                        },
                        "optimeDate" : ISODate("2023-02-28T10:05:56Z"),
                        "optimeDurableDate" : ISODate("2023-02-28T10:05:56Z"),
                        "lastHeartbeat" : ISODate("2023-02-28T10:06:03.923Z"),
                        "lastHeartbeatRecv" : ISODate("2023-02-28T10:06:05.168Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncingTo" : "mongodb-cluster-1.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "syncSourceHost" : "mongodb-cluster-1.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "syncSourceId" : 1,
                        "infoMessage" : "",
                        "configVersion" : 1
                }
        ],
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1677578756, 1),
                "signature" : {
                        "hash" : BinData(0,"oZy0BsY56kwcTxa49P92Aoi1U2M="),
                        "keyId" : NumberLong("7205141014401515524")
                }
        },
        "operationTime" : Timestamp(1677578756, 1)
}
MainRepSet:SECONDARY> exit
bye
root@mongodb-cluster-0:/# exit
exit

# 删除当前角色为primary的pod,cluster-1
➜  ~ kubectl delete po -n mongodb-cluster mongodb-cluster-1
pod "mongodb-cluster-1" deleted

# 此时primary为 cluster-0
➜  ~ kubectl exec -it pod/mongodb-cluster-0 -n mongodb-cluster -- bash
root@mongodb-cluster-0:/# mongo --host mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local --port 27017 -uroot -p'jason1989' admin
MongoDB shell version v4.2.21
connecting to: mongodb://mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017/admin?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("f895659e-ac2a-4b64-bc51-9e832c1e6c79") }
MongoDB server version: 4.2.21
Server has startup warnings:
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] ** WARNING: You are running this process as the root user, which is not recommended.
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten]
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten]
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/enabled is 'always'.
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] **        We suggest setting it to 'never'
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten]
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/defrag is 'always'.
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten] **        We suggest setting it to 'never'
2023-02-28T09:57:56.098+0000 I  CONTROL  [initandlisten]
---
Enable MongoDB's free cloud-based monitoring service, which will then receive and display
metrics about your deployment (disk utilization, CPU, operation statistics, etc).

The monitoring data will be available on a MongoDB website with a unique URL accessible to you
and anyone you share the URL with. MongoDB may use this information to make product
improvements and to suggest MongoDB products and deployment options to you.

To enable free monitoring, run the following command: db.enableFreeMonitoring()
To permanently disable this reminder, run the following command: db.disableFreeMonitoring()
---

MainRepSet:PRIMARY> rs.status()
{
        "set" : "MainRepSet",
        "date" : ISODate("2023-02-28T10:08:05.826Z"),
        "myState" : 1,
        "term" : NumberLong(5),
        "syncingTo" : "",
        "syncSourceHost" : "",
        "syncSourceId" : -1,
        "heartbeatIntervalMillis" : NumberLong(2000),
        "majorityVoteCount" : 2,
        "writeMajorityCount" : 2,
        "optimes" : {
                "lastCommittedOpTime" : {
                        "ts" : Timestamp(1677578879, 1),
                        "t" : NumberLong(5)
                },
                "lastCommittedWallTime" : ISODate("2023-02-28T10:07:59.341Z"),
                "readConcernMajorityOpTime" : {
                        "ts" : Timestamp(1677578879, 1),
                        "t" : NumberLong(5)
                },
                "readConcernMajorityWallTime" : ISODate("2023-02-28T10:07:59.341Z"),
                "appliedOpTime" : {
                        "ts" : Timestamp(1677578879, 1),
                        "t" : NumberLong(5)
                },
                "durableOpTime" : {
                        "ts" : Timestamp(1677578879, 1),
                        "t" : NumberLong(5)
                },
                "lastAppliedWallTime" : ISODate("2023-02-28T10:07:59.341Z"),
                "lastDurableWallTime" : ISODate("2023-02-28T10:07:59.341Z")
        },
        "lastStableRecoveryTimestamp" : Timestamp(1677578879, 1),
        "lastStableCheckpointTimestamp" : Timestamp(1677578879, 1),
        "electionCandidateMetrics" : {
                "lastElectionReason" : "stepUpRequestSkipDryRun",
                "lastElectionDate" : ISODate("2023-02-28T10:07:29.794Z"),
                "electionTerm" : NumberLong(5),
                "lastCommittedOpTimeAtElection" : {
                        "ts" : Timestamp(1677578846, 1),
                        "t" : NumberLong(4)
                },
                "lastSeenOpTimeAtElection" : {
                        "ts" : Timestamp(1677578846, 1),
                        "t" : NumberLong(4)
                },
                "numVotesNeeded" : 2,
                "priorityAtElection" : 1,
                "electionTimeoutMillis" : NumberLong(10000),
                "priorPrimaryMemberId" : 1,
                "numCatchUpOps" : NumberLong(0),
                "newTermStartDate" : ISODate("2023-02-28T10:07:29.800Z"),
                "wMajorityWriteAvailabilityDate" : ISODate("2023-02-28T10:07:31.841Z")
        },
        "electionParticipantMetrics" : {
                "votedForCandidate" : true,
                "electionTerm" : NumberLong(4),
                "lastVoteDate" : ISODate("2023-02-28T10:01:25.749Z"),
                "electionCandidateMemberId" : 1,
                "voteReason" : "",
                "lastAppliedOpTimeAtElection" : {
                        "ts" : Timestamp(1677578463, 1),
                        "t" : NumberLong(3)
                },
                "maxAppliedOpTimeInSet" : {
                        "ts" : Timestamp(1677578463, 1),
                        "t" : NumberLong(3)
                },
                "priorityAtElection" : 1
        },
        "members" : [
                {
                        "_id" : 0,
                        "name" : "mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 611,
                        "optime" : {
                                "ts" : Timestamp(1677578879, 1),
                                "t" : NumberLong(5)
                        },
                        "optimeDate" : ISODate("2023-02-28T10:07:59Z"),
                        "syncingTo" : "",
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "",
                        "electionTime" : Timestamp(1677578849, 1),
                        "electionDate" : ISODate("2023-02-28T10:07:29Z"),
                        "configVersion" : 1,
                        "self" : true,
                        "lastHeartbeatMessage" : ""
                },
                {
                        "_id" : 1,
                        "name" : "mongodb-cluster-1.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 27,
                        "optime" : {
                                "ts" : Timestamp(1677578879, 1),
                                "t" : NumberLong(5)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1677578879, 1),
                                "t" : NumberLong(5)
                        },
                        "optimeDate" : ISODate("2023-02-28T10:07:59Z"),
                        "optimeDurableDate" : ISODate("2023-02-28T10:07:59Z"),
                        "lastHeartbeat" : ISODate("2023-02-28T10:08:03.870Z"),
                        "lastHeartbeatRecv" : ISODate("2023-02-28T10:08:04.969Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncingTo" : "mongodb-cluster-2.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "syncSourceHost" : "mongodb-cluster-2.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "syncSourceId" : 2,
                        "infoMessage" : "",
                        "configVersion" : 1
                },
                {
                        "_id" : 2,
                        "name" : "mongodb-cluster-2.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 406,
                        "optime" : {
                                "ts" : Timestamp(1677578879, 1),
                                "t" : NumberLong(5)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1677578879, 1),
                                "t" : NumberLong(5)
                        },
                        "optimeDate" : ISODate("2023-02-28T10:07:59Z"),
                        "optimeDurableDate" : ISODate("2023-02-28T10:07:59Z"),
                        "lastHeartbeat" : ISODate("2023-02-28T10:08:03.870Z"),
                        "lastHeartbeatRecv" : ISODate("2023-02-28T10:08:03.903Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncingTo" : "mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "syncSourceHost" : "mongodb-cluster-0.mongodb-cluster.mongodb-cluster.svc.cluster.local:27017",
                        "syncSourceId" : 0,
                        "infoMessage" : "",
                        "configVersion" : 1
                }
        ],
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1677578879, 1),
                "signature" : {
                        "hash" : BinData(0,"uWTLz+tmuTjYnxWQsNWfGONPXd8="),
                        "keyId" : NumberLong("7205141014401515524")
                }
        },
        "operationTime" : Timestamp(1677578879, 1)
}
MainRepSet:PRIMARY>

```

## 6 mongodb副本集数据备份

```shell
只要连接到mongodb副本集中的任意一个pod都可以实现mongodb的数据备份，那么怎样才能连接上呢，这里提供两种思路。
(1) 在k8s集群中找一个节点，修改"/etc/resolv.conf"文件，将k8s集群的dns域名解析配置放到首位即可。
(2) 添加一个类型为nodePort的service，这样k8s的每个节点上都会有"27017"监听端口。
```

​	
