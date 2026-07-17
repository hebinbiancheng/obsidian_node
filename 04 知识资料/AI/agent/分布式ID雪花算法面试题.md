---
title: 分布式ID雪花算法面试题：原理 + 实现 + 三个深挖
type: knowledge
status: evergreen
source_type: 微信公众号
created: 2026-07-01
tags:
  - Java
  - 分布式
  - 雪花算法
  - Snowflake
  - 分布式ID
  - 面试
source: https://mp.weixin.qq.com/s/39HtDR7Bez0DpiLvagkPpQ
confidence: high
---

# 分布式ID雪花算法：原理 + 实现 + 深挖

> 原文：[微信文章](https://mp.weixin.qq.com/s/39HtDR7Bez0DpiLvagkPpQ) · 2026-07-01
> 原始资料：`^[raw/articles/wechat-snowflake-distributed-id-2026.html]`

---

## 一句话总结

雪花算法 = 64 位二进制身份证号：1 位符号 + 41 位时间戳 + 10 位机器 ID + 12 位序列号。同一毫秒靠序列号、不同机器靠机器 ID、不同时间靠时间戳，永不重复。

---

## 1. 为什么 MySQL 自增 ID 不够用

- 订单服务 5 台机器各自插入 → ID 重复
- 用户服务和商品服务分库 → 各自的订单表 ID 冲突

需要不依赖数据库的全局唯一 ID 生成器。

---

## 2. 雪花算法结构

类比中国身份证号（地区码 + 生日 + 顺序号）。

| 位 | 长度 | 用途 | 容量 |
|----|------|------|------|
| 符号位 | 1 bit | 永远是 0（正数） | — |
| 时间戳 | 41 bits | 毫秒级，相对起始时间 | 可用 69 年 |
| 机器 ID | 10 bits | 5 位数据中心 + 5 位机器号 | 1024 个节点 |
| 序列号 | 12 bits | 同一毫秒内自增 | 4096/ms |

---

## 3. 完整实现

```java
public class SnowflakeIdGenerator {
    private final long EPOCH = 1609459200000L; // 2020-01-01
    private final long WORKER_ID_BITS = 5L;
    private final long DATACENTER_ID_BITS = 5L;
    private final long SEQUENCE_BITS = 12L;

    private final long workerIdShift = SEQUENCE_BITS; // 12
    private final long datacenterIdShift = SEQUENCE_BITS + WORKER_ID_BITS; // 17
    private final long timestampShift = SEQUENCE_BITS + WORKER_ID_BITS + DATACENTER_ID_BITS; // 22

    private final long workerId;
    private final long datacenterId;
    private long sequence = 0L;
    private long lastTimestamp = -1L;

    public synchronized long nextId() {
        long timestamp = timeGen();
        if (timestamp < lastTimestamp) { timestamp = lastTimestamp; } // 时钟回拨
        if (timestamp == lastTimestamp) {
            sequence = (sequence + 1) & 4095;
            if (sequence == 0) { timestamp = tilNextMillis(lastTimestamp); }
        } else {
            sequence = 0L;
        }
        lastTimestamp = timestamp;
        return ((timestamp - EPOCH) << timestampShift)
             | (datacenterId << datacenterIdShift)
             | (workerId << workerIdShift)
             | sequence;
    }
}
```

---

## 4. 三个常见问题

| 问题 | 解决方案 |
|------|---------|
| **时钟回拨** | 检测到时间倒退 → 等待追上；或 Leaf 租约模式续命 |
| **ID 不连续** | 不影响业务，不连续反而是安全优势（防止猜订单量） |
| **单机性能** | 每秒 400 万 ID，QPS 更高时批量预生成 |

---

## 5. 美团 Leaf（工业级方案）

- **号段模式**：每次从数据库拿一批 ID 段，内存分配，不用每次都查 DB
- **双优化**：雪花算法和号段模式可切换
- **时钟治理**：内置时钟回拨处理
- 已在美团验证多年，开源

---

## 面试答题模板

> 常用方案：UUID（不推荐，太长无序）、数据库自增（分布式不够）、雪花算法（主流）、号段模式（Leaf）。
>
> 雪花算法原理：64 位 = 1 符号 + 41 时间戳 + 10 机器 ID + 12 序列号。同一毫秒靠序列号，不同机器靠机器 ID，不同时间靠时间戳。
>
> 常见问题：时钟回拨（等追回或拒绝生成）。生产推荐美团 Leaf，有号段模式和时钟治理。
