---
layout: post
title: 'PromethuesAlert初试'
# subtitle: '通过shell脚本，进行redis批量迁移'
date: 2023-03-20
categories: 监控
author: Jason
# cover: 'assets/img/profile.png'
tags: promethuesAlert monitoring grafana
---

### 背景

内网的机器需要将grafana上的告警发到企微群

### 思路

借助开源工具prometheusAlert，打通grafana和企微群

### 具体实现

#### 可以通过本地部署和docker部署，为了方便，采用docker部署的方式

> docker部署，并将配置本地化

```shell
# 运行
docker run -d \
-p 8080:8080 \
-e PA_LOGIN_USER=prometheusalert \
-e PA_LOGIN_PASSWORD=prometheusalert \
-e PA_TITLE=PrometheusAlert \
-e PA_OPEN_FEISHU=1 \
-e PA_OPEN_DINGDING=1 \
-e PA_OPEN_WEIXIN=1 \
--name PA
feiyu563/prometheus-alert:latest

# 持久化配置
mkdir -p /data/promethuesAlert && cd /data/promethuesAlert
docker cp PA:/app/conf .
docker rm -f PA
# 挂载本地配置，启动PA
docker run -d \
-p 8080:8080 \
-v /data/promethuesAlert/conf:/app/conf \
-e PA_LOGIN_USER=prometheusalert \
-e PA_LOGIN_PASSWORD=prometheusalert \
-e PA_TITLE=PrometheusAlert \
-e PA_OPEN_FEISHU=1 \
-e PA_OPEN_DINGDING=1 \
-e PA_OPEN_WEIXIN=1 \
--name PA \
feiyu563/prometheus-alert:latest
```

> 调整指定企微机器人的相关配置后，重启容器

```shell
# 修改企微机器人地址
vim /data/promethuesAlert/conf/app.conf
#---------------------↓webhook-----------------------
#是否开启微信告警通道,可同时开始多个通道0为关闭,1为开启
open-weixin=1
#默认企业微信机器人地址
wxurl=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxxx

# 重启容器
docker restart PA
```

### TODO 

#### prometheusAlert界面

#### grafana告警接入

### 参考资料

项目路径：https://github.com/feiyu563/PrometheusAlert

grafana告警配置：https://blog.csdn.net/qq_38571773/article/details/128735955