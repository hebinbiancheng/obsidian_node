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

## 相关笔记

- [[03 CoreDNS 与服务发现]] — CoreDNS 基础原理与服务发现机制
- [[Kubernetes运维-集群升级节点管理故障排查]] — 集群运维故障排查
- [[04 Kubernetes 排障与面试题]] — K8s 排障方法论与面试题
