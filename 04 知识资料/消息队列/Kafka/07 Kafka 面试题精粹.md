---
title: Kafka 面试题精粹
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

# Kafka 面试题精粹

## 一句话结论

Kafka 面试聚焦三大核心：架构与概念（Topic/Partition/ISR）、可靠性（消息不丢失/ACK/副本）、性能（顺序写/零拷贝/批量），辅以对比选型和实践场景题。

## 架构与概念

### Q1：Kafka 的整体架构是怎样的？

Producer → Broker 集群（Topic/Partition/Replica）→ Consumer，ZooKeeper 负责集群协调。每个 Partition 有 Leader + Follower，Leader 处理读写。

→ 详见 [[01 Kafka 核心概念与架构]]

### Q2：为什么有了 Topic 还要 Partition？

- **提升吞吐量**：多 Partition 并行处理
- **负载均衡**：消费者组内均匀分配
- **扩展性**：增加 Partition 即可水平扩展

### Q3：Kafka 为什么不支持读写分离？

主写从读有两个缺点：数据一致性问题（同步延迟窗口）、延时问题（Kafka 主从同步路径更长）。Kafka 采用主写主读，简化逻辑、无延迟、副本稳定时数据一致。

### Q4：ZooKeeper 在 Kafka 中的作用？

存放集群元数据、成员管理、Controller 选举、主题删除等管理任务。KIP-500 后将被 KRaft（Raft 共识算法）替代。

## 生产者

### Q5：ACK 机制中 0、1、-1 分别代表什么？

| ACK | 含义 | 可靠性 |
|-----|------|--------|
| 0 | 不等待确认 | 最低 |
| 1 | Leader 确认即可 | 中等 |
| -1 | 所有 ISR 确认 | 最高 |

→ 详见 [[02 Kafka 生产者]]

### Q6：Kafka 如何实现幂等和事务？

- **幂等**：PID + Sequence Number，保证单分区单会话内不重复
- **事务**：`transactional.id` + 幂等，保证跨分区原子写入

### Q7：Kafka 消息发送流程是怎样的？

Main 线程 → 拦截器 → 序列化器 → 分区器 → RecordAccumulator → Sender 线程 → Broker。

## 消费者

### Q8：什么是消费者组？

消费者组是 Kafka 实现负载均衡和容错的机制。组内多个消费者共同消费 Topic，每个 Partition 只被组内一个消费者消费。组间广播，组内单播。

### Q9：重平衡是什么？触发条件有哪些？

消费者组内分区重新分配的过程。触发条件：组成员数量变化、订阅主题数量变化、分区数变化。重平衡期间所有消费者停止消费（STW）。

→ 详见 [[03 Kafka 消费者与消费组]]

### Q10：Kafka 如何保证消息顺序消费？

- 同一 Partition 内消息有序
- 全局有序：Topic 只设 1 个 Partition
- 局部有序：指定 Partition Key，相同 Key 进入同一 Partition
- 多线程消费：相同 Key 放入同一内存队列，单线程处理

### Q11：有哪些情形会造成重复消费？如何解决？

| 场景 | 解决方案 |
|------|---------|
| 消费者故障（手动提交前） | 幂等性 / EOS |
| 自动提交间隔内失败 | 手动提交 Offset |
| 生产者重试（未开启幂等） | 开启幂等性 |
| 消费者组再平衡 | 消费者幂等处理 |

## 可靠性

### Q12：Kafka 如何保证消息不丢失？

**三端保障**：

| 端 | 方案 |
|----|------|
| Producer | `acks=-1` + `retries` + 回调重试 |
| Broker | `replication.factor > 1` + `min.insync.replicas > 1` + `unclean.leader.election.enable=false` |
| Consumer | 关闭自动提交，手动提交 Offset |

→ 详见 [[05 Kafka 高可用与可靠性]]

### Q13：ISR 是什么？什么情况下副本会被踢出 ISR？

ISR（In-Sync Replicas）是与 Leader 保持同步的副本集合。Follower 的 LEO 落后 Leader 超过 `replica.lag.time.max.ms`（默认 10s）时被踢出。

### Q14：HW 和 LEO 分别是什么？

- **LEO**（Log End Offset）：日志下一条待写入消息的 Offset
- **HW**（High Watermark）：ISR 中最小的 LEO，消费者只能消费 HW 之前的消息

### Q15：为什么 Kafka 没办法 100% 保证消息不丢失？

- 生产者异步发送后崩溃，消息无人知晓
- Page Cache 未刷盘前宕机
- 副本同步延迟时 Leader 宕机
- 极端情况下所有副本同时故障

Kafka 只对"已提交"消息做最大保证，极端场景需外部机制（分布式事务、本地消息表）。

## 性能

### Q16：Kafka 为什么这么快？

四大核心技术：**磁盘顺序写入**、**零拷贝（sendfile）**、**页缓存**、**批量发送与压缩**。

→ 详见 [[06 Kafka 高性能原理]]

### Q17：Kafka 的零拷贝是如何实现的？

- 索引文件：mmap（MappedByteBuffer），内核态/用户态共享缓冲区
- 日志传输：`FileChannel.transferTo()` → `sendfile`，页缓存直接 DMA 到网卡

## 存储

### Q18：Kafka 的文件存储结构是怎样的？

Topic → Partition 目录 → Segment 文件（.index + .log）。Segment 以起始 Offset 命名，稀疏索引加速查找。

→ 详见 [[04 Kafka 存储机制]]

### Q19：Kafka 如何查找指定 Offset 的消息？

1. 二分查找定位 Segment 文件
2. 在 .index 中二分查找最近索引点
3. 在 .log 中顺序扫描到目标 Offset

## 实践场景

### Q20：Kafka 消息积压怎么处理？

| 原因 | 方案 |
|------|------|
| 消费能力不足 | 增加 Partition + 增加消费者（消费者数 = 分区数） |
| 下游处理不及时 | 提高每批次拉取数量，优化处理逻辑 |

其他：启用备用消费端、动态扩容、异步处理、设置监控告警。

### Q21：Kafka 消费端挂了，积压大量消息怎么处理？

- 冗余设计：多消费者实例 + 自动重启
- 动态扩容：增加消费者实例
- 批量处理：增大批量大小，提高吞吐
- 异步处理 + 多线程并发
- 降级策略：优先保证关键业务

### Q22：Kafka 如何实现延迟队列？

基于**时间轮（TimingWheel）** 实现，插入/删除 O(1)。TimingWheel 负责增删，DelayQueue 负责时间推进，避免"空推进"。

## 对比与选型

### Q23：Kafka 和 RabbitMQ 的区别？

| 维度 | Kafka | RabbitMQ |
|------|-------|----------|
| 吞吐量 | 高（百万级/秒） | 较低 |
| 可靠性 | 通过配置保证 | 天然支持事务和确认 |
| 消息模型 | 发布-订阅为主 | 点对点 + 发布-订阅 |
| 消息确认 | 无内置确认机制 | 有 ACK 确认机制 |
| 适用场景 | 大数据流处理、日志 | 金融交易、实时性要求高 |
| 集群负载均衡 | ZooKeeper 管理 | 需 LoadBalancer 支持 |

### Q24：四大 MQ 对比（Kafka / RocketMQ / RabbitMQ / ActiveMQ）

| 维度 | Kafka | RocketMQ | RabbitMQ | ActiveMQ |
|------|-------|----------|----------|----------|
| 吞吐量 | 最高 | 高 | 中 | 低 |
| 可靠性 | 高（需配置） | 高 | 高 | 中 |
| 消息模型 | 发布-订阅 | 发布-订阅 + 点对点 | 发布-订阅 + 点对点 | 发布-订阅 + 点对点 |
| 事务 | 支持 | 支持 | 不支持 | 支持 |
| 语言 | Scala/Java | Java | Erlang | Java |
| 社区活跃度 | 最高 | 高 | 高 | 低 |

### Q25：如何设计一个消息队列？

- **可伸缩性**：分布式设计，Broker → Topic → Partition，增加 Partition 扩容
- **持久化**：磁盘顺序写，避免随机 IO
- **高可用**：多副本 → Leader/Follower → 故障自动选举
- **数据零丢失**：参考 Kafka 三端保障方案

## 运维相关

### Q26：Kafka 集群数量建议？

不超过 7 个。节点越多，消息复制时间越长，吞吐量越低。建议单数（容错率更高，超过半数故障集群不可用）。

### Q27：Kafka 判断节点存活的条件？

1. 节点必须维护与 ZooKeeper 的连接（心跳机制）
2. Follower 必须能及时同步 Leader 的写操作，延迟不能太久

### Q28：Kafka 重启是否会导致数据丢失？

- 数据写入磁盘，一般不会丢失
- 重启过程中若有消费者消费，可能来不及提交 Offset，导致数据不准确（丢失或重复）

## 关联笔记

- [[00 Kafka 知识库索引]] — 知识库总索引
- [[01 Kafka 核心概念与架构]] — 核心概念
- [[02 Kafka 生产者]] — 生产者机制
- [[03 Kafka 消费者与消费组]] — 消费者机制
- [[04 Kafka 存储机制]] — 存储结构
- [[05 Kafka 高可用与可靠性]] — 可靠性方案
- [[06 Kafka 高性能原理]] — 性能原理
