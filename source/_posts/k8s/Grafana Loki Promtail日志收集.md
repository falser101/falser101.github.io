---
title: Grafana Loki Promtail
author: falser101
date: 2024-06-28
headerDepth: 3
category:
  - k8s
tag:
  - 监控告警
star: true
---

# Grafana Loki Promtail

传统的日志收集框架如ELK（Logstash + Elasticsearch + Kibana）太过重量级，如果需求复杂，服务器资源不受限制，推荐使用ELK（`Logstash + Elasticsearch + Kibana`）方案；如果需求仅是将不同服务器上的日志采集上来集中展示，且需要一个轻量级的框架，那使用PLG（`Promtail + Loki + Grafana`）最合适不过了

> 各组件官网地址
> 
> [Loki](https://grafana.com/docs/loki/latest/)
> 
> [promtail](https://grafana.com/docs/loki/latest/send-data/promtail/)
> 
> [grafana](https://grafana.com/docs/grafana/latest/)

## 下载安装包

[Loki安装包](https://github.com/grafana/loki/releases/download/v2.9.8/loki-linux-amd64.zip)，[Promtail安装包](https://github.com/grafana/loki/releases/download/v2.9.8/promtail-linux-amd64.zip)

```shell
unzip loki-linux-amd64.zip
unzip promtail-linux-amd64.zip
```

## 配置

Loki配置，可以参考[官网配置](https://grafana.com/docs/loki/latest/configure/examples/configuration-examples/)，有很多例子，这里我使用的是本地配置

```yaml
auth_enabled: false

server:
  http_listen_port: 3100 # 服务监听端口

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory
  replication_factor: 1
  path_prefix: /tmp/loki

schema_config:
  configs:
  - from: 2020-05-15
    store: tsdb
    object_store: filesystem
    schema: v13
    index:
      prefix: index_
      period: 24h

storage_config:
  filesystem:
    directory: /tmp/loki/chunks
```

Promtail配置，也可以参考[官网配置](https://grafana.com/docs/loki/latest/send-data/promtail/configuration/)，本次演示的配置如下

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0
positions:
  filename: /home/zhangjf/positions.yaml
clients:
  - url: http://localhost:3100/loki/api/v1/push # loki推送log地址
scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost # 目标设置本机
    labels:
      job: loki
      __path__: /home/zhangjf/loki.log # loki进程的日志文件地址
```

## 启动

```shell
nohup ./loki-linux-amd64 -config.file=loki.yaml >/dev/null 2>loki.log 2>&1 &
nohup ./promtail-linux-amd64 -config.file=promtail.yaml > promtail.log 2>&1 &
```

## 可视化

在Grafana原生支持Loki数据源，在web页面添加

![Grafana](/images/k8s/GLP日志收集/2024-06-28-14-04-46-image.png)

然后添加一个dashboard，通过Import 的方式添加，id：13639

![dashboard](/images/k8s/GLP日志收集/2024-06-28-14-06-34-image.png)

导入成功后可以看到日志数据成功展示出来了

![](/images/k8s/GLP日志收集/2024-06-28-14-05-52-image.png)
