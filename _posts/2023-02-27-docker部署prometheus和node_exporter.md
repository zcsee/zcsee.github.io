---
layout: post
title: 'docker部署prometheus'
date: 2023-02-27
categories: 监控
author: Jason
tags: docker prometheus 监控
---

### 部署node-exporter

> 因为9100端口与elasticsearch-head冲突，所以更改端口为19100

```shell
docker run -d -p 19100:9100 \
  -v "/proc:/host/proc:ro" \
  -v "/sys:/host/sys:ro" \
  -v "/:/rootfs:ro" \
  prom/node-exporter
```




### 部署Prometheus
```shell
docker run  -d \
    -p 9090:9090 \
    -v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml  \
    prom/prometheus

# 增加了本机的node-export端口以获取本机的监控数据
cat /opt/prometheus/prometheus.yml
global:
  scrape_interval:     60s
  evaluation_interval: 60s

scrape_configs:

  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus

  - job_name: linux
    static_configs:
      - targets: ['192.168.216.3:19100']
        labels:
          instance: c71
```

