---
title: Prometheus 监控全流程实战：部署调优排障
date: 2026-07-02
tags: [prometheus, 监控, grafana, alertmanager, 运维, 告警]
category: 知识资料
source: https://mp.weixin.qq.com/s/zBE2dqBvkik1Ac07dNSv1w
---

# Prometheus 监控全流程实战

> 原文: [微信文章](https://mp.weixin.qq.com/s/zBE2dqBvkik1Ac07dNSv1w)

---

## 一、Prometheus 核心特点

- **Pull 模式**：主动拉取，不等待推送
- **多维数据模型**：标签（label）标识，查询灵活
- **PromQL**：强大的查询语言
- **服务发现**：自动发现 K8s/Consul 中的服务
- **生态丰富**：各种 exporter 几乎能监控所有东西

---

## 二、部署方式

### 二进制部署

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz
tar xzf prometheus-2.53.0.linux-amd64.tar.gz
# --web.enable-lifecycle 支持热加载：curl -X POST localhost:9090/-/reload
```

基础配置：

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "node"
    static_configs:
      - targets: ["localhost:9100"]
```

### Docker Compose 全家桶

```yaml
services:
  prometheus:
    image: prom/prometheus:v2.53.0
    ports: ["9090:9090"]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
  grafana:
    image: grafana/grafana:10.4.0
    ports: ["3000:3000"]
  alertmanager:
    image: prom/alertmanager:v0.27.0
    ports: ["9093:9093"]
```

### K8s 部署（Prometheus Operator）

```bash
git clone https://github.com/prometheus-operator/kube-prometheus.git
kubectl apply --server-side -f manifests/setup/
kubectl apply -f manifests/
```

**ServiceMonitor**：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

**告警规则（PrometheusRule）**：

```yaml
spec:
  groups:
    - name: node
      rules:
        - alert: NodeDown
          expr: up{job="node"} == 0
          for: 1m
          labels:
            severity: critical
        - alert: HighCPU
          expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
          for: 5m
          labels:
            severity: warning
```

**数据持久化**：

```bash
kubectl -n monitoring patch prometheus k8s --type merge \
  -p '{"spec":{"storage":{"volumeClaimTemplate":{"spec":{"storageClassName":"nfs-storage","resources":{"requests":{"storage":"50Gi"}}}}}}}'
```

---

## 三、高可用方案

### 方案一：双 Prometheus + 负载均衡

两套相同 Prometheus，前面 Nginx/HAProxy 负载均衡。Alertmanager 内置告警去重。

### 方案二：Thanos（大规模推荐）

```
Prometheus → Thanos Sidecar → S3/OSS/MinIO（长期存储）
                              ↓
                        Thanos Querier（全局查询）
```

```bash
thanos sidecar --tsdb.path=/prometheus --objstore.config-file=/etc/thanos/objstore.yml
thanos query --store=thanos-sidecar-1:10901 --store=thanos-sidecar-2:10901
```

### 方案三：VictoriaMetrics（轻量替代）

单二进制，兼容 PromQL，天然集群模式：

```bash
docker run -d --name victoriametrics -p 8428:8428 victoriametrics/victoria-metrics
```

Prometheus remote_write：
```yaml
remote_write:
  - url: "http://victoriametrics:8428/api/v1/write"
```

---

## 四、批量部署 node_exporter（Ansible）

```yaml
# install_node_exporter.yml
- name: Install Node Exporter
  hosts: all
  tasks:
    - name: Download node_exporter
      get_url:
        url: "https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz"
    - name: Extract & Copy binary
    - name: Create systemd service & Start
```

自动注册到 Prometheus（file_sd）：

```yaml
scrape_configs:
  - job_name: "nodes"
    file_sd_configs:
      - files: ["/etc/prometheus/targets/nodes/*.json"]
        refresh_interval: 5m
```

```bash
curl -X POST http://localhost:9090/-/reload
```

---

## 五、网络设备监控（SNMP）

snmp_exporter + generator 自动生成配置：

```bash
docker run --rm -v $(pwd):/opt/ prom/snmp-exporter generator
```

Prometheus 采集：
```yaml
- job_name: "switch"
  snmp:
    target: 192.168.1.1
    module: [if_mib]
  static_configs:
    - targets: [192.168.1.1, 192.168.1.2]
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - target_label: __address__
      replacement: snmp_exporter:9116
```

常用指标：接口流量、CPU、内存、接口状态、丢包率。

---

## 六、告警对接

### 企业微信

Alertmanager → 转发服务 → 企业微信 Webhook：

```yaml
receivers:
  - name: "wechat"
    webhook_configs:
      - url: "http://wechat-relay:8060/wechat"
        send_resolved: true
```

### 短信（阿里云）

只对 **critical** 级别启用（约 0.04 元/条）。

### 邮件

Alertmanager 原生支持：

```yaml
global:
  smtp_smarthost: "smtp.exmail.qq.com:465"
  smtp_from: "alert@yourcompany.com"
```

---

## 七、生产调优

### 采集间隔分级

```yaml
scrape_configs:
  - job: critical  scrape_interval: 10s
  - job: normal    scrape_interval: 30s
  - job: low       scrape_interval: 60s
```

### 存储优化

```bash
--storage.tsdb.retention.time=30d
--storage.tsdb.retention.size=50GB
--storage.tsdb.wal-compression
```

### 查询优化

- 避免大范围 Range Query
- 善用 **recording rules** 预计算
- `topk()` / `limit` 限制返回量

```yaml
groups:
  - name: recording_rules
    interval: 1m
    rules:
      - record: job:node_cpu_usage:avg_rate5m
        expr: avg by(job) (1 - rate(node_cpu_seconds_total{mode="idle"}[5m]))
```

---

## 八、故障排查

### 启动失败

```bash
promtool check config /etc/prometheus/prometheus.yml
journalctl -u prometheus -f
```

### 采集目标不可达

```bash
curl -s http://target:9100/metrics | head -5
systemctl status node_exporter
ss -tlnp | grep 9100
```

### 磁盘不足

```bash
du -sh /data/prometheus/data/*
# 缩短保留期
--storage.tsdb.retention.time=7d
```

### 告警不触发

```bash
promtool check rules /etc/prometheus/rules/*.yml
curl http://localhost:9090/api/v1/alerts
curl http://localhost:9093/api/v2/alerts
```

---

## 九、资源规划

| 规模 | CPU | 内存 | 磁盘 |
|------|-----|------|------|
| <50 台 | 2 核 | 4 GB | 50 GB SSD |
| 50-500 台 | 4 核 | 8 GB | 200 GB SSD |
| 500-2000 台 | 8 核 | 16 GB | 500 GB SSD |
| >2000 台 | 用 Thanos/VictoriaMetrics 分布式 |

**磁盘估算公式**：每时间序列约 3-5 字节/小时（压缩后）。

```
100 台 × 1000 指标 × 15s 间隔 → 100K 时间序列
→ 72MB/h → 30天 ≈ 52GB → 建议 70GB
```
---

## K8s 集群健康检查五层指标

> 补充来源: [微信文章](https://mp.weixin.qq.com/s/yw-JLJoBDNpp3wUWbVyTag)

![K8s健康检查](assets/prometheus-k8s-health.png)

### 第一层：控制平面

**API Server QPS**：
```promql
sum(rate(apiserver_request_total[5m]))
```

**API Server P99 延迟**：
```promql
histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (le, verb))
```

| P99 延迟 | 状态 |
|-----------|------|
| < 100 ms | ✅ 健康 |
| 100–500 ms | ⚠️ 关注 |
| > 500 ms | 🔴 风险 |

> 注意：`list` 和 `watch` 延迟天然偏高，建议按 verb 分开观察。

**ETCD**：
```promql
etcd_server_has_leader       # 必须 = 1
etcd_mvcc_db_total_size_in_bytes / 1024 / 1024   # DB 大小，接近 2GB 需压缩
```

---

### 第二层：节点

**Node Ready**：
```promql
kube_node_status_condition{condition="Ready", status="true"}
```

**CPU**（建议 < 70%，告警 > 80%）：
```promql
(1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) > 0.8
```

**内存**（建议 < 80%，超过 90% 可能 OOM）：
```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.9
```

**磁盘**（容器目录 < 80%）：
```promql
(1 - node_filesystem_avail_bytes{mountpoint="/var/lib/containerd"} 
   / node_filesystem_size_bytes{mountpoint="/var/lib/containerd"}) > 0.8
```

---

### 第三层：Pod

**Pod 状态分布**：
```promql
sum(kube_pod_status_phase{phase=~"Running|Pending|Failed"}) by (phase)
```

**重启速率**（每小时有重启即需关注，每 5 分钟多次 = CrashLoopBackOff）：
```promql
rate(kube_pod_container_status_restarts_total[1h]) > 0
```

**CrashLoopBackOff 排查**：
```bash
kubectl logs <pod-name> --previous   # 看上轮日志，不是当前
```

---

### 第四层：业务流量

**Ingress QPS**：
```promql
sum(rate(nginx_ingress_controller_requests[2m])) by (host)
```

**5xx 错误率**（> 1% 立即排查）：
```promql
sum(rate(nginx_ingress_controller_requests{status=~"5.."}[2m]))
  / sum(rate(nginx_ingress_controller_requests[2m]))
```

**P99 响应时间**（> 1s = 业务卡顿）：
```promql
histogram_quantile(0.99, 
  sum(rate(nginx_ingress_controller_request_duration_seconds_bucket[2m])) by (le, host)) > 1
```

---

### 第五层：告警有效性

每天检查：
- Alertmanager 是否 alive
- 是否有 firing 状态的告警
- 告警是否真正送达

---

### 每日健康检查清单

| 顺序 | 层次 | 关键指标 | 期望 |
|:--:|------|------|------|
| ① | 控制平面 | API Server P99、ETCD Leader | < 100ms, Leader=1 |
| ② | 节点 | CPU/内存/磁盘 | <70%/80%/80% |
| ③ | Pod | Pending/Restart/CrashLoop | 尽量 0 |
| ④ | 业务 | QPS/5xx/P99 | 错误率<1%, P99 平稳 |
| ⑤ | 告警 | Alertmanager 未处理告警 | 0 条 |

---

## 相关笔记

- [[K8s网络 CNI规范 Flannel Calico原理]]
- [[Linux 假死现象内核级排障]]
- [[45道Nginx面试题]]
