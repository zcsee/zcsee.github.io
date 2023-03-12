---
layout: post
title: 'Kafka设定指定groupId的消费者组的指定日期的offset'
date: 2023-03-12
categories: 中间件
author: Jason
tags: Kafka 日期 offset 消费者组
---



```shell
# 设置groupId为group_test的消费者组的offset为20230308的12点，NEW-OFFSET为0,0
root@737c61919753:/opt/kafka_2.13-2.8.1/bin# kafka-consumer-groups.sh  --bootstrap-server 172.173.0.10:9092 --group group_test --topic kafka_demo --reset-offsets --to-datetime 2023-03-08T12:00:00.000 -execute

GROUP                          TOPIC                          PARTITION  NEW-OFFSET
group_test                     kafka_demo                     0          0
group_test                     kafka_demo                     1          0

# 消费到所有的消息
root@737c61919753:/opt/kafka_2.13-2.8.1/bin# kafka-console-consumer.sh --bootstrap-server 172.173.0.10:9092 --topic kafka_demo --group group_test
hello
hello
to
high
22
time
go

# 设置到20230309的12点，NEW-OFFSET调整为1，1
root@737c61919753:/opt/kafka_2.13-2.8.1/bin# kafka-consumer-groups.sh  --bootstrap-server 172.173.0.10:9092 --group group_test --topic kafka_demo --reset-offsets --to-datetime 2023-03-09T12:00:00.000 -execute

GROUP                          TOPIC                          PARTITION  NEW-OFFSET
group_test                     kafka_demo                     0          1
group_test                     kafka_demo                     1          1
# 消费消息减少2个
root@737c61919753:/opt/kafka_2.13-2.8.1/bin# kafka-console-consumer.sh --bootstrap-server 172.173.0.10:9092 --topic kafka_demo --group group_test
time
go
hello
to
high

```

