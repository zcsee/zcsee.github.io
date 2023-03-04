---
layout: post
title: 'docker部署kafka集群'
date: 2023-03-01
categories: kafka
author: Jason
tags: docker kafka 集群
---



## docker部署kafka集群

### 集群规划

| 机器 | zookeeper | kafka |
| ---- | --------- | ----- |
| C71  | 1         | 1     |
| C72  |           | 1     |
| C73  |           | 1     |

### 部署zookeeper

#### 拉取镜像

```shell
docker pull wurstmeister/zookeeper
```

#### 启动容器

```shell
docker run -d --network dockerKafka --ip 172.173.0.9 --name zookeeper -p 2181:2181 -t wurstmeister/zookeeper

# 检查监听
ss -nltp | grep 2181
```

### 搭建kafka集群

#### 拉取镜像

```shell
docker pull wurstmeister/kafka
```

#### 启动容器

```shell
# 修改id和端口，以在同一台虚机上跑kafka集群
# kafka0
docker run -d --network dockerKafka --ip 172.173.0.10 --name kafka0 -p 9092:9092 -e KAFKA_BROKER_ID=0 -e KAFKA_ZOOKEEPER_CONNECT=172.173.0.9:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://172.173.0.10:9092 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 -t wurstmeister/kafka
# kafka1
docker run -d --network dockerKafka --ip 172.173.0.11 --name kafka1 -p 9093:9093 -e KAFKA_BROKER_ID=1 -e KAFKA_ZOOKEEPER_CONNECT=172.173.0.9:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://172.173.0.11:9093 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9093 -t wurstmeister/kafka
# kafka2
docker run -d --network dockerKafka --ip 172.173.0.12 --name kafka2 -p 9094:9094 -e KAFKA_BROKER_ID=2 -e KAFKA_ZOOKEEPER_CONNECT=172.173.0.9:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://172.173.0.12:9094 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9094 -t wurstmeister/kafka

# 查看运行结果
➜  ~ docker ps | grep kafka
3600d00e55aa   wurstmeister/kafka       "start-kafka.sh"         26 minutes ago   Up 26 minutes   0.0.0.0:9094->9094/tcp, :::9094->9094/tcp                                              kafka2
62d2213746af   wurstmeister/kafka       "start-kafka.sh"         26 minutes ago   Up 26 minutes   0.0.0.0:9093->9093/tcp, :::9093->9093/tcp                                              kafka1
737c61919753   wurstmeister/kafka       "start-kafka.sh"         28 minutes ago   Up 28 minutes   0.0.0.0:9092->9092/tcp, :::9092->9092/tcp  
```

### Kafka相关操作

#### Topics

##### kafka-topics.sh常用操作

```shell
# 查看topics支持的相关操作
root@737c61919753:/opt/kafka_2.13-2.8.1/bin# ./kafka-topics.sh
Create, delete, describe, or change a topic.
Option                                   Description
------                                   -----------
--alter                                  Alter the number of partitions,
                                           replica assignment, and/or
                                           configuration for the topic.
--at-min-isr-partitions                  if set when describing topics, only
                                           show partitions whose isr count is
                                           equal to the configured minimum. Not
                                           supported with the --zookeeper
                                           option.
--bootstrap-server <String: server to    REQUIRED: The Kafka server to connect
  connect to>                              to. In case of providing this, a
                                           direct Zookeeper connection won't be
                                           required.
--command-config <String: command        Property file containing configs to be
  config property file>                    passed to Admin Client. This is used
                                           only with --bootstrap-server option
                                           for describing and altering broker
                                           configs.
--config <String: name=value>            A topic configuration override for the
                                           topic being created or altered. The
                                           following is a list of valid
                                           configurations:
                                                cleanup.policy
                                                compression.type
                                                delete.retention.ms
                                                file.delete.delay.ms
                                                flush.messages
                                                flush.ms
                                                follower.replication.throttled.
                                           replicas
                                                index.interval.bytes
                                                leader.replication.throttled.replicas
                                                max.compaction.lag.ms
                                                max.message.bytes
                                                message.downconversion.enable
                                                message.format.version
                                                message.timestamp.difference.max.ms
                                                message.timestamp.type
                                                min.cleanable.dirty.ratio
                                                min.compaction.lag.ms
                                                min.insync.replicas
                                                preallocate
                                                retention.bytes
                                                retention.ms
                                                segment.bytes
                                                segment.index.bytes
                                                segment.jitter.ms
                                                segment.ms
                                                unclean.leader.election.enable
                                         See the Kafka documentation for full
                                           details on the topic configs. It is
                                           supported only in combination with --
                                           create if --bootstrap-server option
                                           is used (the kafka-configs CLI
                                           supports altering topic configs with
                                           a --bootstrap-server option).
--create                                 Create a new topic.
--delete                                 Delete a topic
--delete-config <String: name>           A topic configuration override to be
                                           removed for an existing topic (see
                                           the list of configurations under the
                                           --config option). Not supported with
                                           the --bootstrap-server option.
--describe                               List details for the given topics.
--disable-rack-aware                     Disable rack aware replica assignment
--exclude-internal                       exclude internal topics when running
                                           list or describe command. The
                                           internal topics will be listed by
                                           default
--force                                  Suppress console prompts
--help                                   Print usage information.
--if-exists                              if set when altering or deleting or
                                           describing topics, the action will
                                           only execute if the topic exists.
--if-not-exists                          if set when creating topics, the
                                           action will only execute if the
                                           topic does not already exist.
--list                                   List all available topics.
--partitions <Integer: # of partitions>  The number of partitions for the topic
                                           being created or altered (WARNING:
                                           If partitions are increased for a
                                           topic that has a key, the partition
                                           logic or ordering of the messages
                                           will be affected). If not supplied
                                           for create, defaults to the cluster
                                           default.
--replica-assignment <String:            A list of manual partition-to-broker
  broker_id_for_part1_replica1 :           assignments for the topic being
  broker_id_for_part1_replica2 ,           created or altered.
  broker_id_for_part2_replica1 :
  broker_id_for_part2_replica2 , ...>
--replication-factor <Integer:           The replication factor for each
  replication factor>                      partition in the topic being
                                           created. If not supplied, defaults
                                           to the cluster default.
--topic <String: topic>                  The topic to create, alter, describe
                                           or delete. It also accepts a regular
                                           expression, except for --create
                                           option. Put topic name in double
                                           quotes and use the '\' prefix to
                                           escape regular expression symbols; e.
                                           g. "test\.topic".
--topics-with-overrides                  if set when describing topics, only
                                           show topics that have overridden
                                           configs
--unavailable-partitions                 if set when describing topics, only
                                           show partitions whose leader is not
                                           available
--under-min-isr-partitions               if set when describing topics, only
                                           show partitions whose isr count is
                                           less than the configured minimum.
                                           Not supported with the --zookeeper
                                           option.
--under-replicated-partitions            if set when describing topics, only
                                           show under replicated partitions
--version                                Display Kafka version.
--zookeeper <String: hosts>              DEPRECATED, The connection string for
                                           the zookeeper connection in the form
                                           host:port. Multiple hosts can be
                                           given to allow fail-over.
root@737c61919753:/opt/kafka_2.13-2.8.1/bin#


# 创建一个名字为"second"的topic，分区数为3，副本数为3
root@737c61919753:/opt/kafka_2.13-2.8.1/bin# ./kafka-topics.sh --topic second --create --bootstrap-server 172.173.0.10:9092,172.173.0.11:9093,172.173.0.12:9094 --partitions 3 --replication-factor 3
Created topic second.

# 查看second这个topic的详细信息 
root@737c61919753:/opt/kafka_2.13-2.8.1/bin# ./kafka-topics.sh --describe --bootstrap-server 172.173.0.10:9092,172.173.0.11:9093,172.173.0.12:9094 --topic second
Topic: second   TopicId: unUo33zVQwK1aKQfG9SeiA PartitionCount: 3       ReplicationFactor: 1    Configs: segment.bytes=1073741824
        Topic: second   Partition: 0    Leader: 0       Replicas: 0     Isr: 0
        Topic: second   Partition: 1    Leader: 2       Replicas: 2     Isr: 2
        Topic: second   Partition: 2    Leader: 1       Replicas: 1     Isr: 1

# 更改second这个topic的分区数为4
root@737c61919753:/opt/kafka_2.13-2.8.1/bin# ./kafka-topics.sh --alter --bootstrap-server 172.173.0.10:9092,172.173.0.11:9093,172.173.0.12:9094 --topic second  --partitions 4
root@737c61919753:/opt/kafka_2.13-2.8.1/bin# ./kafka-topics.sh --describe --bootstrap-server 172.173.0.10:9092,172.173.0.11:9093,172.173.0.12:9094 --topic second
Topic: second   TopicId: ap4hfQEnTCyeEIdN11aaXw PartitionCount: 4       ReplicationFactor: 3    Configs: segment.bytes=1073741824
        Topic: second   Partition: 0    Leader: 1       Replicas: 1,0,2 Isr: 1,0,2
        Topic: second   Partition: 1    Leader: 0       Replicas: 0,2,1 Isr: 0,2,1
        Topic: second   Partition: 2    Leader: 2       Replicas: 2,1,0 Isr: 2,1,0
        Topic: second   Partition: 3    Leader: 1       Replicas: 1,2,0 Isr: 1,2,0
```

##### kafka-console-producer.sh常用操作

```shell
# 支持的操作
root@737c61919753:/opt/kafka_2.13-2.8.1/bin# ./kafka-console-producer.sh
Missing required option(s) [bootstrap-server]
Option                                   Description
------                                   -----------
--batch-size <Integer: size>             Number of messages to send in a single
                                           batch if they are not being sent
                                           synchronously. (default: 200)
--bootstrap-server <String: server to    REQUIRED unless --broker-list
  connect to>                              (deprecated) is specified. The server
                                           (s) to connect to. The broker list
                                           string in the form HOST1:PORT1,HOST2:
                                           PORT2.
--broker-list <String: broker-list>      DEPRECATED, use --bootstrap-server
                                           instead; ignored if --bootstrap-
                                           server is specified.  The broker
                                           list string in the form HOST1:PORT1,
                                           HOST2:PORT2.
--compression-codec [String:             The compression codec: either 'none',
  compression-codec]                       'gzip', 'snappy', 'lz4', or 'zstd'.
                                           If specified without value, then it
                                           defaults to 'gzip'
--help                                   Print usage information.
--line-reader <String: reader_class>     The class name of the class to use for
                                           reading lines from standard in. By
                                           default each line is read as a
                                           separate message. (default: kafka.
                                           tools.
                                           ConsoleProducer$LineMessageReader)
--max-block-ms <Long: max block on       The max time that the producer will
  send>                                    block for during a send request
                                           (default: 60000)
--max-memory-bytes <Long: total memory   The total memory used by the producer
  in bytes>                                to buffer records waiting to be sent
                                           to the server. (default: 33554432)
--max-partition-memory-bytes <Long:      The buffer size allocated for a
  memory in bytes per partition>           partition. When records are received
                                           which are smaller than this size the
                                           producer will attempt to
                                           optimistically group them together
                                           until this size is reached.
                                           (default: 16384)
--message-send-max-retries <Integer>     Brokers can fail receiving the message
                                           for multiple reasons, and being
                                           unavailable transiently is just one
                                           of them. This property specifies the
                                           number of retries before the
                                           producer give up and drop this
                                           message. (default: 3)
--metadata-expiry-ms <Long: metadata     The period of time in milliseconds
  expiration interval>                     after which we force a refresh of
                                           metadata even if we haven't seen any
                                           leadership changes. (default: 300000)
--producer-property <String:             A mechanism to pass user-defined
  producer_prop>                           properties in the form key=value to
                                           the producer.
--producer.config <String: config file>  Producer config properties file. Note
                                           that [producer-property] takes
                                           precedence over this config.
--property <String: prop>                A mechanism to pass user-defined
                                           properties in the form key=value to
                                           the message reader. This allows
                                           custom configuration for a user-
                                           defined message reader. Default
                                           properties include:
                                                parse.key=true|false
                                                key.separator=<key.separator>
                                                ignore.error=true|false
--request-required-acks <String:         The required acks of the producer
  request required acks>                   requests (default: 1)
--request-timeout-ms <Integer: request   The ack timeout of the producer
  timeout ms>                              requests. Value must be non-negative
                                           and non-zero (default: 1500)
--retry-backoff-ms <Integer>             Before each retry, the producer
                                           refreshes the metadata of relevant
                                           topics. Since leader election takes
                                           a bit of time, this property
                                           specifies the amount of time that
                                           the producer waits before refreshing
                                           the metadata. (default: 100)
--socket-buffer-size <Integer: size>     The size of the tcp RECV size.
                                           (default: 102400)
--sync                                   If set message send requests to the
                                           brokers are synchronously, one at a
                                           time as they arrive.
--timeout <Integer: timeout_ms>          If set and the producer is running in
                                           asynchronous mode, this gives the
                                           maximum amount of time a message
                                           will queue awaiting sufficient batch
                                           size. The value is given in ms.
                                           (default: 1000)
--topic <String: topic>                  REQUIRED: The topic id to produce
                                           messages to.
--version                                Display Kafka version.
# 往topic second发送数据“hello”
root@737c61919753:/opt/kafka_2.13-2.8.1/bin# ./kafka-console-producer.sh --bootstrap-server 172.173.0.10:9092,172.173.0.11:9093,172.173.0.12:9094 --topic second
>hello

```

##### kafka-console-consumer.sh常用操作

```shell
# 支持的操作
root@737c61919753:/opt/kafka_2.13-2.8.1/bin# ./kafka-console-consumer.sh
This tool helps to read data from Kafka topics and outputs it to standard output.
Option                                   Description
------                                   -----------
--bootstrap-server <String: server to    REQUIRED: The server(s) to connect to.
  connect to>
--consumer-property <String:             A mechanism to pass user-defined
  consumer_prop>                           properties in the form key=value to
                                           the consumer.
--consumer.config <String: config file>  Consumer config properties file. Note
                                           that [consumer-property] takes
                                           precedence over this config.
--enable-systest-events                  Log lifecycle events of the consumer
                                           in addition to logging consumed
                                           messages. (This is specific for
                                           system tests.)
--formatter <String: class>              The name of a class to use for
                                           formatting kafka messages for
                                           display. (default: kafka.tools.
                                           DefaultMessageFormatter)
--from-beginning                         If the consumer does not already have
                                           an established offset to consume
                                           from, start with the earliest
                                           message present in the log rather
                                           than the latest message.
--group <String: consumer group id>      The consumer group id of the consumer.
--help                                   Print usage information.
--isolation-level <String>               Set to read_committed in order to
                                           filter out transactional messages
                                           which are not committed. Set to
                                           read_uncommitted to read all
                                           messages. (default: read_uncommitted)
--key-deserializer <String:
  deserializer for key>
--max-messages <Integer: num_messages>   The maximum number of messages to
                                           consume before exiting. If not set,
                                           consumption is continual.
--offset <String: consume offset>        The offset id to consume from (a non-
                                           negative number), or 'earliest'
                                           which means from beginning, or
                                           'latest' which means from end
                                           (default: latest)
--partition <Integer: partition>         The partition to consume from.
                                           Consumption starts from the end of
                                           the partition unless '--offset' is
                                           specified.
--property <String: prop>                The properties to initialize the
                                           message formatter. Default
                                           properties include:
                                          print.timestamp=true|false
                                          print.key=true|false
                                          print.offset=true|false
                                          print.partition=true|false
                                          print.headers=true|false
                                          print.value=true|false
                                          key.separator=<key.separator>
                                          line.separator=<line.separator>
                                          headers.separator=<line.separator>
                                          null.literal=<null.literal>
                                          key.deserializer=<key.deserializer>
                                          value.deserializer=<value.
                                           deserializer>
                                          header.deserializer=<header.
                                           deserializer>
                                         Users can also pass in customized
                                           properties for their formatter; more
                                           specifically, users can pass in
                                           properties keyed with 'key.
                                           deserializer.', 'value.
                                           deserializer.' and 'headers.
                                           deserializer.' prefixes to configure
                                           their deserializers.
--skip-message-on-error                  If there is an error when processing a
                                           message, skip it instead of halt.
--timeout-ms <Integer: timeout_ms>       If specified, exit if no message is
                                           available for consumption for the
                                           specified interval.
--topic <String: topic>                  The topic id to consume on.
--value-deserializer <String:
  deserializer for values>
--version                                Display Kafka version.
--whitelist <String: whitelist>          Regular expression specifying
                                           whitelist of topics to include for
                                           consumption.
# 从topic second消费数据，如果不加--from-beginning的话，只会增量的消费consumer进程启动后，发送到指定topic的消息
root@737c61919753:/opt/kafka_2.13-2.8.1/bin# ./kafka-console-consumer.sh --bootstrap-server 172.173.0.10:9092,172.173.0.11:9093,172.173.0.12:9094 --topic second --from-beginning
hello

```



