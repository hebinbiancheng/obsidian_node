---
title: K8s控制面故障应急响应与复盘实录
date: 2026-07-02
tags: [kubernetes, 故障复盘, sre, etcd, apiserver, 应急响应]
category: 知识资料
source: https://mp.weixin.qq.com/s/_8uuhcXo5eZN2HmCntYH9g
---

# K8s 控制面故障应急响应与复盘

> 原文: [微信文章](https://mp.weixin.qq.com/s/_8uuhcXo5eZN2HmCntYH9g)

---

## 故障背景

- 集群：v1.26，跨 AZ 三 Master
- 规模：~120 微服务，峰值 QPS ~8000
- etcd 数据目录 5.2GB，未配置自动压缩
- 故障时机：下午业务高峰（14:00-16:00），HPA 大量触发

---

## 故障时间线

| 时间 | 事件 |
|------|------|
| 14:02 | `kubectl get pods` 响应 5s（正常 <1s），未触发告警 |
| 14:18 | 多个 Pod OOM 驱逐，HPA 扩容，新建 Pod Pending |
| 14:23 | `KubeletOutOfDisk` 告警，值班未升级 |
| 14:35 | apiserver 间歇 503，DevOps 平台不可用，升级 SRE |
| 14:42 | SRE 确认 etcd leader 30 秒内切换 3 次 |
| 14:50 | 流量切到仅剩的稳定 Master，暂停发布 |
| 15:12 | **根因定位：etcd 磁盘 I/O 延迟飙升 → 心跳超时** |
| 15:28 | etcd 磁盘迁移到本地 NVMe，集群恢复 |
| 15:45 | 所有 Pending Pod 完成调度 |
| 17:00 | Postmortem 会议 |

---

## 故障链路分析

```
业务高峰 HPA 大量扩容
  → etcd 写入压力 ×5-8 倍
  → 磁盘 I/O 饱和（+ 未自动压缩，5.2GB 数据）
  → etcd 心跳超时
  → Leader 频繁切换（30s 内 3 次）
  → apiserver 间歇 503
  → Pod 全部 Pending
```

### 根因

**etcd 磁盘 I/O 延迟飙升**，触发心跳超时。

叠加因素：
- etcd 数据目录 5.2GB 未自动压缩
- 业务高峰 HPA 触发大量 Pod 创建
- 磁盘非 NVMe，I/O 性能不足

---

## 值班操作复盘

### ✗ 误判一：忽略慢查询信号

14:02 `kubectl` 响应 5s 是 etcd I/O 毛刺的**第一个信号**，但巡检脚本只有日志没告警。

**改进**：

```yaml
groups:
  - name: apiserver-latency
    rules:
      - alert: ApiserverRequestSlow
        expr: |
          histogram_quantile(0.99,
            rate(apiserver_request_duration_seconds_bucket[5m])) > 1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "apiserver P99 延迟 > 1s"
```

### ✗ 误判二：未升级 KubeletOutOfDisk

14:23 `KubeletOutOfDisk` 告警本质是节点磁盘不足导致 kubelet 无法创建 Pod。值班当成普通告警未重视。

---

## 应急处置（SRE 介入后）

```bash
# 1. 确认 etcd 状态
etcdctl endpoint health --cluster
etcdctl endpoint status --cluster --write-out=table

# 2. 确认 Leader 切换频率
# 看指标：etcd_server_leader_changes_seen_total

# 3. 切流：将 apiserver 流量引向稳定 Master
# （通过 LB 调整权重或临时下线异常节点）

# 4. 暂停所有发布和 HPA 变更
kubectl scale deployment --all --replicas=0  # 非业务 Pod
```

---

## 改进措施

| 类别 | 措施 |
|------|------|
| **etcd** | 配置 `--auto-compaction-retention=1h` 自动压缩 |
| **磁盘** | etcd 数据目录迁移到独立 NVMe SSD |
| **监控** | apiserver P99 延迟 > 1s 即告警 |
| **告警** | KubeletOutOfDisk 升级为 critical，关联 etcd 指标 |
| **巡检** | `kubectl` 响应时间纳入 SLI |
| **演练** | 每季度一次 etcd 故障演练 |

---

## 避坑经验

1. **慢查询是 etcd 故障的最早期信号**，不要忽略
2. **etcd 必须配自动压缩**，否则历史 revision 堆积 → 磁盘 I/O 飙升
3. **etcd 磁盘优先 NVMe**，I/O 延迟是 etcd 稳定性的第一要素
4. **告警要关联**：KubeletOutOfDisk + apiserver 慢 → 可能是 etcd 磁盘问题
5. **HPA 高峰期放大效应**：业务高峰 + HPA 扩容 = 控制面压力倍增

---

## 相关笔记

- [[etcd 如何保存 Kubernetes 状态]]
- [[Prometheus 监控全流程实战]]
- [[K8s Deployment 实战指南]]
