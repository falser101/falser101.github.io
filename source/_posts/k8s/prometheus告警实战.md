---
title: prometheus告警实战
date: 2024-09-10 16:26:14
category:
  - k8s
tag:
  - 监控告警
---

## 配置

### alertmanager配置
```yaml
global:
  resolve_timeout: 5m # 设置解决（resolve）告警的超时时间为5分钟。
route:
  group_by: ['alertname'] # 告警按照 alertname 分组。
  group_wait: 10s # 每个分组等待10秒，以便将相关的告警聚合在一起。
  group_interval: 10s # 每个分组之间的间隔时间为10秒。
  repeat_interval: 1h # 告警的重复间隔时间为1小时。
  receiver: 'webhook' # 默认的接收器是 'webhook'。
  routes:
  - match:
      cluster: 'tlq-cn' # 满足条件的使用这个接收器
    receiver: 'webhook'
receivers: # 定义接收器
- name: 'webhook'
  webhook_configs:
  - url: 'http://127.0.0.1:5001/'
inhibit_rules: # 定义抑制规则，用于在某些条件下抑制告警
  - source_match:
      severity: 'critical' # 如果源告警的严重性为 'critical'。
    target_match:
      severity: 'warning' # 如果目标告警的严重性为 'warning'
    equal: ['alertname', 'dev', 'instance'] # 源告警和目标告警必须在这些标签上相等才会触发抑制。
```

### Prometheus配置
```yaml
global:
  scrape_interval: 15s # 采集间隔为15秒。
  evaluation_interval: 15s # 评估间隔为15秒。
rule_files:
  - /home/prometheus/rule/*.yml # 下面配置的告警规则文件路径
alerting:
  alert_relabel_configs: # : 配置了动态修改 alert 属性的规则。
  - source_labels: [dc] # 使用标签 dc 作为源标签。
    regex: (.+)\d+ # 使用正则表达式从源标签中提取数据，并且去掉结尾的数字。
    target_label: dc1 # 将提取的数据赋值给目标标签 dc1。
  alertmanagers:
  - static_configs: # 静态配置，指定了 Alertmanager 的地址为 localhost:9093。
    - targets:
      - localhost:9093
scrape_configs:
# 联邦集群配置
- job_name: 'federate'
  scrape_interval: 15s
  honor_labels: true
  metrics_path: '/federate'
  params:
    'match[]':
      - '{job="prometheus"}'
      - '{job="broker"}'
      - '{job="bookie"}'
      - '{job="zookeeper"}'
      - '{job="proxy"}'
      - '{job="node-exporter"}'
  static_configs:
    - targets:
      - '10.10.22.162:52119'
- job_name: 'prometheus' # 任务名称。
  static_configs:
  - targets: ['localhost:9090'] # 采集prometheus自身。
- job_name: 'node' # 任务名称。
  static_configs:
  - targets: ['localhost:9100'] # 采集本机。
```

### 告警规则
```yaml
groups:
  - name: alert.rules # 告警规则组名称
    rules:
      - alert: InstanceDown # 告警的名称，命名为 InstanceDown
        expr: up == 0 # 触发告警的表达式，如果 up 的值为 0，表示实例处于宕机状态。
        for: 1m # 即当满足条件并持续1分钟时触发告警
        labels: # 标签，用于标识告警的一些元信息
          severity: critical # 告警的严重性标签，设置为 'critical'。
        annotations: # 注释，提供更详细的描述信息。
          summary: "{{ $labels.instance }}: no data for 1 minute" # 摘要信息，描述实例在1分钟内没有数据。
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute." # 描述信息，指明哪个作业的哪个实例在1分钟以上处于宕机状态。
```

## 启动服务

### 启动 Prometheus
参数`--web.enable-lifecycle`的作用是启用 Prometheus 的热加载功能，允许通过 HTTP 接口动态加载和卸载规则文件。
当我们修改了 Prometheus 的配置文件或者规则文件，需要重新加载配置文件，可以通过 POST 请求 `/-/reload` 接口实现。

```shell
# 启动服务
./prometheus --config.file=prometheus.yml --web.enable-lifecycle
```

### 启动 Alertmanager
```shell
# 启动服务
./alertmanager --config.file=alertmanager.yml
```
当我们修改了Alertmanager的配置文件，需要重新加载配置文件，可以通过 POST 请求 `/-/reload` 接口实现。

## 查看Prometheus
在浏览器输入localhost:9090，即可看到Prometheus的监控页面。点击alerts，可以看到当前告警信息。如下图所示
![Alerts](/imgs/2023/k8s/07/alerts.png)

### 状态值
- Inactive（未激活）：Alert 的初始状态，表示规则条件尚未满足。
- Pending（等待）：表示 Alert 已经被触发，但是在确认一定时间内保持在 Pending 状态，以防止短时间内的噪声或瞬时问题。
- Firing（触发）：表示 Alert 已经被确认，规则条件持续满足。

### 热加载
修改Prometheus规则文件,将expr表达式修改`up{job='broker'}`
```yaml
groups:
  - name: alert.rules # 告警规则组名称
    rules:
      - alert: InstanceDown # 告警的名称，命名为 InstanceDown
        expr: up{job='broker'} == 1 # 触发告警的表达式，如果 up{job='broker'} 的值为 1，表示实例处于启动状态。
        for: 1m # 即当满足条件并持续1分钟时触发告警
        labels: # 标签，用于标识告警的一些元信息
          severity: critical # 告警的严重性标签，设置为 'critical'。
        annotations: # 注释，提供更详细的描述信息。
          summary: "{{ $labels.instance }}: no data for 1 minute" # 摘要信息，描述实例在1分钟内没有数据。
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute." # 描述信息，指明哪个作业的哪个实例在1分钟以上处于宕机状态。
```

修改后调用POST请求`http://localhost:9090/-/reload`，即可更新Prometheus的规则文件。
```shell
curl -X POST http://localhost:9090/-/reload
```

再次查看alerts，可以看到告警信息已经更新了。
![Alerts](/imgs/2023/k8s/07/alerts-update.png)

## 查看Alertmanager
在浏览器输入localhost:9093，即可看到Alertmanager的监控页面。点击alerts，可以看到当前告警信息。如下图所示
![Alerts](/imgs/2023/k8s/07/alertmanager.png)

可以看到告警信息已经发送到Alertmanager，并且被Alertmanager处理了。