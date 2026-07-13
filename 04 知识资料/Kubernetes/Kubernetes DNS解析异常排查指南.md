---
title: Kubernetes DNS解析异常排查指南：从超时到熔断的完整实战
type: knowledge
status: evergreen
source_type: 微信公众号
created: 2026-06-21
updated: 2026-07-06
tags:
  - Kubernetes
  - CoreDNS
  - DNS
  - SRE
  - 故障排查
source: https://mp.weixin.qq.com/s/MLchWcdHJCKtMgmZQ9oz2g
---

# Kubernetes DNS解析异常排查指南

> 原文：[微信文章](https://mp.weixin.qq.com/s/MLchWcdHJCKtMgmZQ9oz2g)

---

## 一句话总结

K8s DNS 问题的根因集中在 **CoreDNS 资源不足、conntrack 表满、ndots 搜索风暴、NodeLocal DNS Cache 崩溃** 四大类。本文提供覆盖 5 类异常 + 5 分钟 SOP + CoreDNS 优化的完整排查体系。

---

## 一、K8s DNS 解析链路

以 ClusterFirst 策略为例：

```
Pod 发起 DNS 查询
  ↓ /etc/resolv.conf（nameserver = kube-dns Service IP）
  ↓ kube-dns Service（ClusterIP: 通常 .10）
  ↓ CoreDNS Pod（默认3副本）
  ↓ 先查本地缓存 → 再查 K8s Service 记录（kubernetes 插件）
  ↓ 非集群内域名 → 上游 DNS（继承 Node 的 /etc/resolv.conf）
  ↓ 返回结果
```

⚠️ 链路上任何环节出问题，DNS 就挂了。

### 关键配置：/etc/resolv.conf

```
nameserver 172.20.0.10      # kube-dns Service IP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5 timeout:2 attempts:3
```

> 💡 **ndots:5 是最大性能杀手**：域名中点号少于 5 个时，先拼接 search 后缀再查询，导致大量不必要 DNS 查询。K8s 1.21+ 有所改善，存量集群仍广泛存在。

---

## 二、5 类常见 DNS 异常

### 🔴 类型 1：DNS 查询超时（最常见）

**典型症状：**

```
curl: (6) Could not resolve host: api.example.com
dial tcp: lookup xxx on 172.20.0.10:53: read udp 172.20.0.10:53: i/o timeout
Name or service not known
```

**快速定位：**

| 步骤 | 操作 | 目的 |
|------|------|------|
| 1 | `kubectl get pods -n kube-system -l k8s-app=kube-dns` | 确认 CoreDNS Pod 是否 Running |
| 2 | `kubectl logs -n kube-system <coredns-pod>` | 查看 CoreDNS 日志有无报错 |
| 3 | `kubectl exec -it <pod> -- nslookup google.com` | 在 Pod 内手动测试 DNS |
| 4 | `kubectl get svc -n kube-system kube-dns` | 确认 kube-dns Service 正常 |
| 5 | 检查 Node 的 conntrack 表是否满 | DNS 超时最常见根因之一 |

**根因 A：CoreDNS Pod 资源不足**

CoreDNS 默认资源限制很低（70m CPU / 170Mi 内存）。集群规模增长时，CoreDNS 会 OOM 或 CPU 节流，表现为间歇性 DNS 超时。

```bash
# 调整 CoreDNS 资源限制
kubectl edit deployment -n kube-system coredns
# 建议：CPU 100m-200m，内存 256Mi-512Mi
# 同时：副本数从 3 提升到 5（跨可用区部署）
```

**根因 B：conntrack 表满（高频！）**

Linux conntrack 表记录所有 NAT 连接。DNS 使用 UDP，每次查询占用一个条目，超时 30s。QPS 高时 conntrack 打满，新连接无法建立。

```bash
# 检查 conntrack 使用量
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# 调大 conntrack 表大小（在所有 Node 上）
echo "net.netfilter.nf_conntrack_max = 1310720" >> /etc/sysctl.conf
echo "net.netfilter.nf_conntrack_tcp_timeout_established = 86400" >> /etc/sysctl.conf
sysctl -p
```

---

### 🟡 类型 2：DNS 查询慢（不超时，但延迟高）

症状隐蔽：P99 延迟从 50ms 飙升到 800ms，无报错，最后发现是 DNS 查询耗时过长。

**根因：ndots:5 导致的 search 风暴**

访问外部域名 `api.weixin.qq.com`（3 个点号 < 5）时：

```
1. api.weixin.qq.com.default.svc.cluster.local  → NXDOMAIN
2. api.weixin.qq.com.svc.cluster.local          → NXDOMAIN
3. api.weixin.qq.com.cluster.local              → NXDOMAIN
4. api.weixin.qq.com.                           → 成功
# 每次失败 2s 超时，最多 attempts:3
# 最坏情况：3 次失败 × 3 次重试 = 18 秒延迟！
```

**修复方案（推荐方案 1）：**

1. **升级 K8s 到 1.21+**：默认 ndots 改为 2，Pod 的 dnsConfig 支持细粒度控制
2. 应用层指定完整域名：域名末尾加 `.`（如 `api.weixin.qq.com.`），跳过 search
3. 自定义 dnsConfig：在 Deployment 里设置 `ndots:2`，仅影响本应用

---

### 🟣 类型 3：集群内服务域名解析失败

症状：Pod 能解析外网域名，但无法解析集群内 Service（如 `redis.default.svc.cluster.local`）。

**常见根因：**

- Service 未创建或命名空间不匹配
- CoreDNS 的 kubernetes 插件配置错误
- NetworkPolicy 拦截了 DNS 流量（53 端口）
- kube-dns Service 的 Endpoints 不正常

**排查命令：**

```bash
# 检查 kube-dns Service 的 Endpoints
kubectl get endpoints -n kube-system kube-dns

# 检查 CoreDNS 配置
kubectl get configmap -n kube-system coredns -o yaml

# 在 Pod 内直接测试集群内域名
kubectl exec -it <pod> -- nslookup redis.default.svc.cluster.local 172.20.0.10
```

---

### 🔵 类型 4：DNS 缓存导致服务发现延迟

最容易被忽视的一类。应用使用 DNS 缓存库（Go `miekg/dns`、Java `dnsjava`），缓存 TTL 设置过长。当 Service 重建导致 ClusterIP 变化时，客户端要等缓存过期才能感知。

> 💡 **最佳实践**：K8s 集群内 DNS 缓存 TTL 建议 5-10 秒（服务可能随时变 IP）。外部域名可设置更长（60s+）。不要禁用缓存（会打爆 CoreDNS），而是设置合理短 TTL。

---

### 🟢 类型 5：NodeLocal DNS Cache 引发的新问题

NodeLocal DNS Cache 在每节点跑 DNS 缓存 DaemonSet，Pod 直接查本地缓存。大幅降低 CoreDNS 负载，但引入新故障模式。

**症状**：部分节点 Pod DNS 异常，其他节点正常。

**根因**：nodelocaldns Pod 在某节点崩溃或 OOM，Kubelet 的 `--cluster-dns` 指向本地缓存（169.254.20.10），导致该节点所有 Pod DNS 全部失效。

```bash
# 检查 nodelocaldns 状态
kubectl get pods -n kube-system -o wide | grep nodelocaldns

# OOMKilled → 调整内存限制
kubectl edit daemonset -n kube-system nodelocaldns
# 建议：512Mi+
```

---

## 三、DNS 故障排查 SOP（5 分钟）

### ■ STEP 1：确认故障范围（1 分钟）
- 全部 Pod 还是部分 Pod？
- 集群内域名还是外部域名？
- 新出现还是间歇出现？

### ■ STEP 2：检查 CoreDNS 健康状态（1 分钟）
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl top pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system <coredns-pod> --tail=50
```

### ■ STEP 3：在 Pod 内手动测试（1 分钟）
```bash
kubectl exec -it <pod> -- nslookup <target>
kubectl exec -it <pod> -- dig <target> @172.20.0.10
# 对比：直接指定 DNS Server 的查询结果
```

### ■ STEP 4：检查基础设施（1 分钟）
- 各 Node 的 conntrack 使用量
- NodeLocal DNS Cache 状态（如果启用）
- kube-dns Service 的 Endpoints 是否完整

### ■ STEP 5：根因修复 + 验证（1 分钟）
- 按对应类型修复
- 修复后在 Pod 内重跑 nslookup 验证

---

## 四、CoreDNS 优化配置

### 1. CoreDNS ConfigMap 优化

```yaml
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
    }
    cache 30           # 🔑 开启缓存，默认 TTL 30s
    reload 10s         # 🔑 限制并发载入
    forward . 8.8.8.8 8.8.4.4  # 🔑 上游 DNS（避免依赖 Node resolv.conf）
    prometheus :9153
}
```

### 2. 副本数与调度策略

- **副本数**：小集群 3 个，大集群 5-7 个，必须配置 podAntiAffinity
- **资源限制**：CPU 200m-500m，内存 512Mi（监控实际用量后调整）
- **探针**：确保 livenessProbe 和 readinessProbe 正确配置

### 3. 关键监控指标

| 指标 | 含义 | 告警阈值 |
|------|------|----------|
| `coredns_dns_request_duration_seconds` | DNS 查询延迟 | P99 > 100ms |
| `coredns_dns_responses_total{rcode="NXDOMAIN"}` | 解析失败次数 | 5 分钟内 > 100 |
| `coredns_cache_hits_total` | 缓存命中率 | < 50% |
| `process_resident_memory_bytes` | 内存使用量 | 接近限制值的 80% |

---

## 五、总结：DNS 稳定性建设三条原则

1. **预防胜于救火**：集群搭建阶段就配置好 CoreDNS（副本数、资源限制、cache 插件），效率比故障后排查高 10 倍。

2. **监控要前置**：DNS 是基础设施的基础，其异常影响是全局性的。给 CoreDNS 配置完善监控告警是 SRE 基本功。

3. **排查要有 SOP**：DNS 故障往往凌晨发生。提前准备好排查 SOP，MTTR 从 30 分钟压缩到 5 分钟。

### 📌 本周行动建议

1. 检查集群 CoreDNS 资源限制和副本数，排查隐患
2. 在 Grafana 加 CoreDNS 监控面板（用第四节的指标）
3. 把排查 SOP 存入团队故障响应文档

---



---

## 六、实战复盘：120+ 节点集群 DNS 超时故障实录 ^[raw/articles/wechat-article-*.html]

> 来源：[微信文章](https://mp.weixin.qq.com/s/wv5SlNX67aBj393VS4-8gQ) · 2026-06-25

某生产 K8s 集群（v1.24，120+ 节点，3000+ Pod）在业务高峰期间突然出现大面积 DNS 解析超时，大量 Pod 无法解析内部服务域名，导致服务雪崩。本文记录从故障发现到根因定位、从临时止血到永久修复的完整过程，并提炼出可复用的排查方法论。

故障现场：凌晨三点的告警风暴

那天凌晨 3:12，手机开始疯狂震动。打开电脑，Grafana 告警面板上 coredns_dns_request_duration_seconds_p99 已经飙到 8 秒——正常情况下应该 < 50ms。同时，多个业务 Pod 的日志里出现大量 i/o timeout 和 no such host 错误。

第一时间的应急响应动作：
• 确认故障范围：是所有 Namespace 还是特定业务？
• 确认故障起始时间：结合变更记录，是否有相关操作？
• 确认影响面：哪些服务受影响，是否有用户侧投诉？
• 临时止血：是否需要先扩容 CoreDNS 或重启 Pod？

登录集群，先看看 CoreDNS 的 Pod 状态和日志。这是任何 DNS 故障排查的第一步。

# 查看 CoreDNS Pod 状态
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
# 输出示例：
# NAME                       READY STATUS RESTARTS AGE   IP
# coredns-6d4b89d8b7-2xqkz  1/1   Running  0        3d    10.244.5.12
# coredns-6d4b89d8b7-7mkc9  1/1   Running  2        3d    10.244.3.28

# 注意：有一个 Pod 重启了 2 次，这不太正常

进一步查看 CoreDNS 日志，重点关注错误和警告：

kubectl -n kube-system logs -l k8s-app=kube-dns --tail=200

# 日志里发现大量以下错误：
# [ERROR] plugin/forward: could not resolve "api.internal.cluster.local. A"
# [ERROR] plugin/forward: no route to host
# [WARNING] No files matching import of "custom.db"

逐层排查：从 Pod 到 Node 再到内核参数

第一层：确认是 DNS 本身的问题还是网络问题

在业务 Pod 里直接发 DNS 请求，绕过业务代码，确认问题边界：

# 进入任意业务 Pod
kubectl exec -it my-app-xxx -- sh

# 使用 dig 测试 DNS 解析
dig @10.96.0.10 api.internal.cluster.local +short
# 10.96.0.10 是 K8s 集群 DNS Service 的 ClusterIP

# 如果 dig 也超时，说明问题在 DNS 层面
# 如果 dig 正常但业务代码仍报错，说明问题在业务侧

💡 始终用 dig 或 nslookup 做裸 DNS 测试，排除业务代码的干扰。

第二层：检查 CoreDNS 配置（Corefile）

CoreDNS 的行为完全由 ConfigMap kube-system/coredns 里的 Corefile 控制。配置错误是 DNS 故障的常见根因。

kubectl -n kube-system get configmap coredns -o yaml

# 正常的 Corefile 示例：
# Corefile: |
#     .:53 {
#         errors
#         health
#         ready
#         kubernetes cluster.local in-addr.arpa ip6.arpa {
#           pods insecure
#           fallthrough in-addr.arpa ip6.arpa
#         }
#         forward . /etc/resolv.conf
#         cache 30
#         loop
#         reload
#         loadbalance
#     }

重点关注以下配置项：
• forward . /etc/resolv.conf：CoreDNS 将上游 DNS 转发给节点的 /etc/resolv.conf。如果节点 DNS 配置错误，所有解析都会失败。
• cache 30：缓存有效期 30 秒。如果业务有大量唯一域名查询，缓存命中率会很低，导致大量请求转发到上游。
• loop：检测转发循环。如果 CoreDNS 把自己作为上游，会造成无限循环。

❌ 我们当时就踩了 forward 循环 的坑：某节点上的 /etc/resolv.conf 被误操作写成了集群 DNS Service IP，导致 CoreDNS 转发请求给自己，形成死循环。

第三层：网络层面排查

DNS 是 UDP（偶尔 TCP）协议，网络策略、iptables 规则、conntrack 表溢出都可能导致解析失败。

检查 DNS 端口是否可达：

# 从业务 Pod 测试能否连接 CoreDNS Pod 的 53 端口
kubectl exec -it my-app-xxx -- nc -u -z 10.244.5.12 53
# -u: UDP 模式  -z: 只扫描不发送数据

# 如果连接失败，检查 NetworkPolicy
kubectl get networkpolicy -A

# 检查 iptables NAT 规则是否异常
iptables -t nat -L KUBE-SERVICES -n -v

根因定位：conntrack 表溢出 + Pod 密度过高

经过逐层排查，我们最终定位到两个叠加的根因。这两个问题单独存在时都不会触发故障，但在业务高峰时叠加在一起，导致了大面积 DNS 超时。

根因一：conntrack 表溢出

Linux 内核的 conntrack 模块用于跟踪网络连接状态，iptables 的 NAT 依赖它。当集群内 DNS 查询并发量很大时，conntrack 表会被快速填满，新的连接无法建立，表现为 DNS 查询超时或丢包。

验证方法：在任意 Node 上执行以下命令：

# 查看当前 conntrack 表使用量
cat /proc/sys/net/netfilter/nf_conntrack_count
# 输出：487329（当前表项数）

cat /proc/sys/net/netfilter/nf_conntrack_max
# 输出：262144（表容量上限）

# 如果 count 接近或超过 max，说明 conntrack 表已满
# 查看溢出统计
cat /proc/net/stat/nf_conntrack | awk 'NR==1{print};NR>1{s+=$NF}END{print "overflow:",s}'

# dmesg 里也会有提示
journalctl -k | grep -i conntrack
# 输出示例：nf_conntrack: table full, dropping packet

我们的集群里 nf_conntrack_count 已经到了 48 万，而上限只有 26 万——表已经严重溢出，大量 DNS 包被内核直接丢弃。

根因二：Pod 密度过高导致 CoreDNS 负载不均

集群有 120+ 节点，但 CoreDNS 只部署了 2 个副本（默认配置），且都调度到了同一个 Node 上。当业务高峰来临时，3000+ Pod 的 DNS 请求全部打到这两个 CoreDNS 实例，CPU 使用率飙到 800%+，大量请求排队超时。

❌ 默认部署的 CoreDNS 只有 2 副本，且没有反亲和性配置，容易集中在少数 Node 上。生产集群务必修改此配置。

解决方案：临时止血 + 永久修复

临时止血（故障发生 15 分钟内完成）

在根因完全确认前，先做以下操作快速止血：
• 扩容 CoreDNS 副本数：kubectl -n kube-system scale deployment coredns --replicas=6
• 将 CoreDNS Pod 分散到不同 Node：配置 podAntiAffinity
• 临时调大 Node 的 conntrack 表容量
• 如仍无法止血，可临时将业务 Pod 的 resolv.conf 改成直连上游 DNS（最后手段）

永久修复一：CoreDNS 部署架构优化

修改 CoreDNS Deployment，确保高可用：

# 编辑 CoreDNS Deployment
kubectl -n kube-system edit deployment coredns

# 关键修改点：
# 1. 副本数至少 4（经验值：每 500 Node 配 4 副本）
# 2. 添加反亲和性，确保 Pod 分散在不同 Node
# 3. 添加资源限制和请求

# 完整 YAML 片段示例：
# spec:
#   replicas: 4
#   template:
#     spec:
#       affinity:
#         podAntiAffinity:
#           requiredDuringSchedulingIgnoredDuringExecution:
#           - labelSelector:
#               matchLabels:
#                 k8s-app: kube-dns
#             topologyKey: kubernetes.io/hostname
#       containers:
#       - name: coredns
#         resources:
#           limits:
#             cpu: "1"
#             memory: 1Gi
#           requests:
#             cpu: 100m
#             memory: 256Mi

修改后滚动更新：

kubectl -n kube-system rollout restart deployment coredns
kubectl -n kube-system rollout status deployment coredns

# 确认新 Pod 分布在不同 Node
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

永久修复二：conntrack 表容量调优

在所有 Node 上调大 conntrack 表容量。推荐通过 DaemonSet 统一配置，避免手动登录每个 Node：

# conntrack 调优 DaemonSet (node-sysctl-ds.yaml)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-sysctl-ds
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: node-sysctl
  template:
    metadata:
      labels:
        app: node-sysctl
    spec:
      initContainers:
      - name: sysctl
        image: busybox:1.36
        command: ["sh","-c"]
        args: ["echo 524288 > /proc/sys/net/netfilter/nf_conntrack_max && echo 65536 > /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established"]
        securityContext:
          privileged: true
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.9

💡 conntrack 表容量不是越大越好。每一条记录占用约 300 字节内存，Node 内存较小时设置过大反而会导致 OOM。经验值：内存 ≥ 16GB 的 Node 设为 524288，8GB 的 Node 设为 262144。

CoreDNS 配置优化：大规模集群调优实践

优化一：调整缓存策略

默认 cache 30 表示 DNS 记录缓存 30 秒。对于内部服务（ClusterIP Service），其 DNS 记录几乎不会变化，可以适当延长缓存时间，减轻 CoreDNS 的解析压力。

# 优化后的 Corefile 配置
Corefile: |
  .:53 {
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
      pods insecure
      fallthrough in-addr.arpa ip6.arpa
      ttl 30
    }
    cache 300
    forward . 10.0.0.2 10.0.0.3
    loop
    reload 10s
    loadbalance
  }

将 cache 从 30 秒改为 300 秒后，缓存命中率从 35% 提升到 82%，CoreDNS CPU 使用率下降了约 40%。

优化二：ndots 参数优化（效果最显著）

默认 ndots=5，意味着每次 DNS 查询都会尝试 5 次不同长度的域名。对于集群内服务，ndots=2 就够了。将 ndots 从 5 改为 2，DNS 查询次数可以减少 60% 以上。

# 在 Deployment 里配置 dnsConfig
spec:
  template:
    spec:
      dnsConfig:
        options:
        - name: ndots
          value: "2"
        - name: timeout
          value: "1"
        - name: attempts
          value: "2"

✅ ndots 优化效果非常显著！将 ndots 从 5 改为 2，DNS 查询次数可以减少 60% 以上，且不需要修改 CoreDNS 配置。所有生产集群都应该做这个优化。

监控体系建设：提前发现问题

这次故障后，我们给 CoreDNS 建立了完整的监控告警体系。CoreDNS 通过 prometheus 插件在 :9153/metrics 暴露指标。

关键监控指标
• coredns_dns_request_duration_seconds{quantile="0.99"}：P99 延迟，超过 500ms 需要告警
• rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m])：SERVFAIL 速率，超过 100/min 需要告警
• rate(coredns_dns_responses_total{rcode="NXDOMAIN"}[5m])：NXDOMAIN 速率，异常增高可能是配置错误
• coredns_cache_misses_total / (coredns_cache_misses_total + coredns_cache_hits_total)：缓存未命中率，超过 50% 需要关注
• container_cpu_usage_seconds_total{pod=~"coredns-.*"}：CoreDNS Pod CPU 使用率，超过 80% 需要扩容

Prometheus 告警规则示例

groups:
- name: coredns
  rules:
  - alert: CoreDNSHighLatency
    expr: |
      histogram_quantile(0.99,
        rate(coredns_dns_request_duration_seconds_bucket[5m])
      ) > 0.5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "CoreDNS P99 延迟超过 500ms"

  - alert: CoreDNSServfail
    expr: |
      rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m]) > 100
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "CoreDNS SERVFAIL 响应过多"

  - alert: CoreDNSPodRestart
    expr: |
      increase(kube_pod_container_restarts{container="coredns"}[30m]) > 0
    labels:
      severity: warning
    annotations:
      summary: "CoreDNS Pod 在过去 30 分钟内重启过"

延伸方案：大规模集群 DNS 的演进方向

方案一：NodeLocal DNS Cache

NodeLocal DNS Cache 在每个 Node 上运行一个本地 DNS 缓存，Pod 的 DNS 请求首先到达本机缓存，未命中才转发到集群 CoreDNS。这样可以大幅减少 CoreDNS 的负载，同时降低 DNS 解析延迟。

# 部署 NodeLocal DNS Cache (K8s v1.18+)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dns/master/docs/node-local-dns/localdns.yaml

# 验证：在任意 Pod 里查看 /etc/resolv.conf
# nameserver 应该变为 169.254.20.10
kubectl exec -it my-app-xxx -- cat /etc/resolv.conf

✅ NodeLocal DNS Cache 在我们的生产集群上降低了 CoreDNS P99 延迟从 200ms 到 15ms，同时将 CoreDNS CPU 使用率降低了 70%。强烈推荐所有生产集群开启。

方案二：CoreDNS 分片（Sharding）

对于超大规模集群，可以将 CoreDNS 按 Namespace 或 Service 分片。不同业务 Pod 的 /etc/resolv.conf 指向不同的 DNS Service IP，避免某个业务的 DNS 风暴影响其他业务。

方案三：Istio DNS Proxy

Istio 1.8+ 引入了 DNS Proxy 功能，Sidecar 可以直接代理 DNS 请求，在本地缓存服务发现结果，完全绕过 CoreDNS。对于已使用 Istio 的集群，这是一个值得尝试的方向。

总结与行动建议

立即行动（今天就做）
• 检查你的 CoreDNS 副本数和反亲和性配置，确保高可用
• 在所有 Node 上检查 conntrack 表使用量，必要时调大
• 将 Pod 的 ndots 从默认值 5 改为 2

本周行动
• 部署 NodeLocal DNS Cache，降低 CoreDNS 负载和解析延迟
• 配置 CoreDNS 监控告警（P99 延迟 + SERVFAIL 速率 + Pod 重启）
• 审查 CoreDNS Corefile 配置，优化缓存策略和上游 DNS

本月行动
• 对 CoreDNS 进行压测，确认当前配置能支撑业务增长
• 编写 DNS 故障应急演练手册，定期组织团队演练
• 评估是否需要 CoreDNS 分片或迁移到服务网格 DNS Proxy

SRE 云原生实战笔记
作者：huyouba1 | 聚焦 K8s · SRE · 可观测性 · AI 运维
关注公众号，每周获取最新云原生运维实战内容

## 相关笔记

- [[03 CoreDNS 与服务发现]] — CoreDNS 基础原理与服务发现机制
- [[Kubernetes运维-集群升级节点管理故障排查]] — 集群运维故障排查
- [[04 Kubernetes 排障与面试题]] — K8s 排障方法论与面试题 — CoreDNS 基础原理与服务发现机制
- [[Kubernetes运维-集群升级节点管理故障排查]] — 集群运维故障排查
- [[04 Kubernetes 排障与面试题]] — K8s 排障方法论与面试题
