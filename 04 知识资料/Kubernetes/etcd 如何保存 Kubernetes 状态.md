---
title: 守住集群的记忆：etcd 如何保存 Kubernetes 状态
date: 2026-07-02
tags: [kubernetes, etcd, raft, 分布式, 备份恢复, 故障处理]
category: 知识资料
source: https://mp.weixin.qq.com/s/MpHPOgD1uk_DmrsVuZ5Zog
---

# etcd 如何保存 Kubernetes 状态

> 原文: [微信文章](https://mp.weixin.qq.com/s/MpHPOgD1uk_DmrsVuZ5Zog)

---

## 为什么需要 etcd

如果 K8s 只把状态保存在单台 API Server 本地文件，宕机即丢失所有资源定义；多台 API Server 又可能记录不同内容。因此需要：

| 要求 | 说明 |
|------|------|
| 一致 | 多客户端看到同一份已确认状态 |
| 持久 | 节点重启后数据不丢失 |
| 高可用 | 少数成员故障时仍可读写 |
| 可观察变化 | Watch 事件通知，避免全量扫描 |
| 支持事务 | 并发写入不互相覆盖 |

---

## etcd 是什么

etcd = 强一致、分布式键值存储。

在 K8s 中，写入链路：

```
用户/Controller → kube-apiserver（认证/鉴权/准入/校验）→ etcd
                                          ↓
Controller/Scheduler/kubelet ← List/Watch ← 获得变化 ← 执行操作
```

> etcd 是底层状态数据库，API Server 是资源访问入口。不能直接 `etcdctl put/del` 修复 K8s 对象。

---

## 架构与 Raft 共识

### 写入流程

```
客户端 → etcd 成员 → (非 Leader 则转发) → Leader
  → Leader 写入 Raft 日志 → 复制给 Follower
  → 多数派持久化确认 → Leader 提交 → 应用到 MVCC
  → 客户端成功响应
```

### 多数派 (Quorum)

```
quorum = floor(n / 2) + 1
```

| 成员数 | 多数派 | 可容忍故障 |
|--------|--------|:---:|
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| **3** | **2** | **1** |
| 4 | 3 | 1 |
| **5** | **3** | **2** |

> 4 成员和 3 成员容错相同但写入开销更大 → 生产用 3 或 5 奇数成员。

### 选举决策

- Follower 故障 → Leader + 另一个 Follower 形成多数派 ✅
- Leader 故障 → 重新选举，短暂停顿后恢复 ✅
- 只剩 1 个成员 → 无法形成多数派，停止写入 ⛔

---

## 核心功能

### 1. 保存并观察 K8s 状态

数据在 `/registry/` 前缀下，MVCC 保存历史版本。

**分层排查顺序**：

1. 应用：`kubectl get pod`、Events、日志
2. 平台：检查 Service selector、EndpointSlice
3. SRE：检查 CoreDNS、测试 Pod 解析
4. 底层：只读查看 etcd 键（仅当 API Server 异常时）

DNS 说明：CoreDNS 通过 Watch API Server 动态生成解析，**不是直接把 DNS 记录存 etcd**。

```bash
# 只读查看键名
etcdctl get /registry/pods/ --prefix --keys-only
etcdctl get /registry/services/specs/ --prefix --keys-only
```

### 2. 快照备份与灾难恢复

> 复制 ≠ 备份。误删 Namespace 会被一致地复制到所有成员。

**备份策略**：

| RPO 要求 | 策略 |
|----------|------|
| 24h 内 | 每日快照 + 多个恢复点 |
| 变更频繁 | 提高频率，升级/变更前额外备份 |
| 敏感数据 | 快照加密、独立凭据、审计 |

```bash
# 创建快照
etcdctl snapshot save /backup/etcd-$(date +%F-%H%M).db
etcdutl snapshot status /backup/etcd-2026-06-04-0200.db --write-out=table
```

> etcd 快照**不包含** PersistentVolume 业务数据和容器镜像。

### 3. 性能与维护

影响稳定性的核心资源（按优先级）：

1. **磁盘写入延迟** — WAL fsync 太慢 → 提交延迟、Leader 不稳定
2. **成员网络 RTT** — 多数派确认需要跨成员通信
3. **CPU/内存** — 大量 Watch 和请求
4. **数据库碎片** — MVCC 需要压缩和碎片整理

**关键指标阈值**：

| 指标 | p99 建议 |
|------|----------|
| `wal_fsync_duration_seconds` | < 10ms |
| `backend_commit_duration_seconds` | < 25ms |
| `network_peer_round_trip_time_seconds` | < 50ms |

**常规硬件起点**：2-4 核、8GB、独立 SSD/NVMe、8GB backend quota。

```bash
# 压缩 + 碎片整理（维护窗口执行）
REVISION=$(etcdctl endpoint status --write-out=json | jq -r '.[0].Status.header.revision')
etcdctl compact "$REVISION"
etcdctl defrag --cluster
```

---

## 三成员故障影响

| 故障 | etcd | Control Plane | 业务 Pod |
|------|------|:---:|:---:|
| 3 正常 | 多数派 OK | 正常 | 正常 |
| 挂 1 Follower | 2 成员形成多数派 | 继续读写 | 无感 |
| 挂 1 Leader | 重选后恢复 | 短暂停顿 | 继续 |
| 挂 2 成员 | 失去多数派 ⛔ | API 大量失败，调度停止 | 已运行 Pod 暂时存活，但无法重建/扩缩容 |

> "业务 Pod 还在运行" ≠ "集群健康"。失去多数派后：新 Pod 无法调度、Deployment 无法扩缩、故障副本无法补齐、ConfigMap/Secret 变更无法传播。

---

## 故障处理

### 损失 1 个成员

先保护剩余多数派，按官方流程替换。**不要直接添加第 4 个成员**（多数派从 2 变 3，新成员有问题则直接失去多数派）。

### 损失 2 个成员

1. 停止变更，保护剩余数据
2. 优先恢复原多数派（网络/节点修复）
3. 无法恢复 → 从快照重建
4. 验证 API Server、Controller、节点、Pod、Service、DNS
5. 对比快照时间确认丢失的变更

> ⚠️ 禁止关闭一致性检查、直接修改数据目录、手工编辑 K8s 键。

---

## 总结

etcd 的关键不是更大节点规格，而是完整的可靠性链路：

> 奇数成员 + 清晰故障域 + 低延迟磁盘和网络 + 持续监控 + 定期维护 + 受保护快照 + 执行过的恢复 Runbook

---

## 参考资料

- [CNCF etcd](https://www.cncf.io/projects/etcd/)
- [etcd 官方文档](https://etcd.io/docs/v3.6/)
- [K8s Operating etcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)

## 相关笔记

- [[Kubernetes 学习]]
- [[K8s Pod 调度流程面试题]]
- [[K8s PVC 绑定 PV 全过程]]
