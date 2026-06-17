---
title: Kafka 高可用与可靠性
source: "[[04 知识资料/面试与技术复习/戴斌0805 XMind 解析]]"
created: 2026-06-12
updated: 2026-06-17
tags:
  - 知识库
  - 消息队列
  - Kafka
  - 面试
status: evergreen
confidence: medium
---

# Kafka 高可用与可靠性

## 一句话结论

Kafka 通过多副本机制（Leader + Follower）、ISR 同步集合、HW/LEO 水位线、Leader Epoch 等机制保证高可用和数据可靠性，但无法做到极端情况下 100% 不丢消息，需要 Producer/Consumer/Broker 三方配合。

## 副本机制

### 基本概念

| 概念 | 说明 |
|------|------|
| **Replica（副本）** | 每个 Partition 可有多个副本，分布在不同 Broker |
| **Leader Replica** | 处理所有读写请求的主副本 |
| **Follower Replica** | 被动从 Leader 同步数据的备份副本 |
| **AR（Assigned Replicas）** | 所有分配的副本集合 |
| **ISR（In-Sync Replicas）** | 与 Leader 保持同步的副本集合 |
| **OSR（Out-of-Sync Replicas）** | 已落后、被踢出 ISR 的副本集合 |

> AR = ISR + OSR

### 为什么不支持读写分离？

Kafka 采用**主写主读**模型（Leader 处理所有读写），不支持主写从读：

| 主写从读的缺点 | 说明 |
|---------------|------|
| 数据一致性 | 主从同步有延迟窗口，读从节点可能读到旧数据 |
| 延时问题 | Kafka 主从同步需经过网络→内存→磁盘→网络→内存→磁盘，比 Redis 更耗时 |

**主写主读的优点**：简化代码逻辑、负载粒度细化、无延迟影响、副本稳定时数据一致。

> Kafka 2.4+ 开始有限度支持 Follower 提供读服务（需配置开启）。

## ISR 机制

### ISR 维护

| 版本 | 判断标准 | 参数 |
|------|---------|------|
| 0.9.x 之前 | Follower 落后消息数超过阈值 | `replica.lag.max.messages` |
| 0.9.x 之后 | Follower LEO 落后 Leader 超过时间阈值 | `replica.lag.time.max.ms`（默认 10s） |

> 0.9.x 之前的实现有问题：Leader 瞬间收到大量消息时，所有 Follower 可能同时被踢出 ISR。改为时间维度后，只要在时间内追上即可。

### ISR 为空时怎么办？

| `unclean.leader.election.enable` | 行为 | 权衡 |
|----------------------------------|------|------|
| `true`（默认） | 允许非 ISR 副本成为 Leader | 可能丢数据，但可用性高 |
| `false` | 等待 ISR 中副本恢复 | 数据不丢，但可能长时间不可用 |

## 水位线机制

### HW 与 LEO

| 概念 | 全称 | 含义 |
|------|------|------|
| **LEO** | Log End Offset | 日志中下一条待写入消息的 Offset |
| **HW** | High Watermark | ISR 中最小的 LEO，消费者只能消费 HW 之前的消息 |
| **LSO** | Last Stable Offset | 事务相关：未完成事务取第一条消息位置，已完成同 HW |
| **LW** | Low Watermark | AR 集合中最小的 logStartOffset |

### HW 的作用

| 作用 | 说明 |
|------|------|
| 消费进度管理 | 消费者通过 HW 确定可安全消费的消息边界 |
| 数据可靠性 | 只有 ISR 全部确认的消息才被认为已提交 |

### Leader Epoch

**问题**：HW 机制在 Leader 连续变更时无法保证数据一致性。

**解决**：引入 Leader Epoch（单调递增的任期号），每次 Leader 切换时递增。新 Leader 验证旧 Leader 的 Epoch 和 HW，只有旧 Epoch ≤ 新 Epoch 且旧 HW ≤ 新 HW 时才接受数据，防止数据回滚。

## 选举机制

### Partition Leader 选举

| 选举类型 | 触发场景 |
|----------|---------|
| **OfflinePartition** | 分区上线（新建或重新上线） |
| **ReassignPartition** | 手动执行副本重分配 |
| **PreferredReplicaPartition** | 手动或自动触发 Preferred Leader 选举 |
| **ControlledShutdownPartition** | Broker 正常关闭 |

**选举策略**：从 AR 中挑选**首个在 ISR 中的副本**作为新 Leader。

### Controller 选举

| 步骤 | 说明 |
|------|------|
| 1 | 所有 Broker 向 ZooKeeper 注册并监听 `/controller` 节点 |
| 2 | Controller 故障时，ZK 删除 `/controller` 节点 |
| 3 | 各 Broker 竞争创建临时节点，先成功者成为新 Controller |
| 4 | 新 Controller 将选举结果写入 ZK，其他 Broker 更新元数据 |

## 消息不丢失方案

### 三端保障

| 端 | 丢失场景 | 解决方案 |
|----|---------|---------|
| **Producer** | 网络问题、缓冲区满、ACK 配置不当 | `acks=-1` + `retries` + 回调重试 |
| **Broker** | Leader 宕机未同步、Page Cache 未刷盘 | `replication.factor > 1`、`min.insync.replicas > 1`、`unclean.leader.election.enable=false` |
| **Consumer** | 自动提交 Offset 后处理失败 | 关闭自动提交，手动提交 Offset |

### 关键参数配置

| 参数 | 建议值 | 说明 |
|------|--------|------|
| `acks` | `-1` (all) | 所有 ISR 确认 |
| `retries` | `MAX` | 无限重试 |
| `replication.factor` | `> 1`（建议 3） | 多副本 |
| `min.insync.replicas` | `> 1`（建议 2） | ISR 最少副本数 |
| `unclean.leader.election.enable` | `false` | 不允许非 ISR 副本选为 Leader |
| `enable.auto.commit` | `false` | 手动提交 Offset |

### 为什么无法 100% 保证不丢？

| 原因 | 说明 |
|------|------|
| 生产者异步发送 | 发送后、回调前 Producer 崩溃，消息无人知晓 |
| Page Cache 未刷盘 | 操作系统在刷盘前宕机 |
| 副本同步延迟 | Leader 宕机时 Follower 未同步完成 |
| 极端故障 | 所有副本同时故障 |

> Kafka 只对"已提交"的消息做最大限度的持久化保证。极端场景需引入分布式事务或本地消息表等外部机制。

## 常见问题

### Q1：Kafka 如何保证高可用？

通过 replica 副本机制：每个 Partition 有多个副本分布在不同 Broker，Leader 处理读写，Follower 同步数据。Leader 宕机后自动从 ISR 中选举新 Leader，实现故障自动转移。

### Q2：ACK=-1 时，Leader 什么时候认为消息已提交？

当 ISR 中所有 Replica 都向 Leader 发送 ACK 确认后，Leader 才 commit，此时 Producer 才认为消息发送成功。配合 `min.insync.replicas` 确保至少有指定数量的副本同步完成。

## 关联笔记

- [[00 Kafka 知识库索引]] — 知识库总索引
- [[01 Kafka 核心概念与架构]] — 架构与副本概念
- [[02 Kafka 生产者]] — ACK 机制与幂等性
- [[03 Kafka 消费者与消费组]] — Offset 管理与手动提交
- [[07 Kafka 面试题精粹]] — 高频面试题
