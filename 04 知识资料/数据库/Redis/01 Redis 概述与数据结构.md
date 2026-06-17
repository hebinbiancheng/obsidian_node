---
title: Redis 概述与数据结构
source: "[[04 知识资料/面试与技术复习/戴斌0805 XMind 解析]]"
created: 2026-06-12
updated: 2026-06-17
tags:
  - 知识库
  - 数据库
  - Redis
  - 面试
status: evergreen
confidence: medium
---

# Redis 概述与数据结构

## 一句话结论

Redis 提供 5 种核心数据类型 + 4 种扩展类型，底层使用 SDS、ziplist、quicklist、skiplist、hashtable 等高效数据结构，支撑缓存、分布式锁、排行榜、消息队列等丰富场景。

## Redis 是什么

Redis 是一种基于内存的数据库，读写操作都在内存中完成，速度极快（读写性能可达 10万/秒）。

### 主要用途
- **缓存**：实现分布式缓存的首选中间件
- **数据库**：实现点赞、关注、排行等高性能需求
- **计算工具**：统计 PV/UV、用户在线天数
- **其他**：分布式锁、消息队列

### Redis vs 关系型数据库

| 维度 | Redis | 关系型数据库 |
|------|-------|------------|
| 数据模型 | 键值对（NoSQL） | 二维表 |
| 存储位置 | 内存 | 磁盘 |
| 读写性能 | 10万/秒 | 远低于 Redis |
| 数据容量 | 受内存限制 | 海量 |

### Redis vs Memcached

| 维度 | Redis | Memcached |
|------|-------|-----------|
| 数据类型 | String/Hash/List/Set/ZSet 等 | 仅 key-value |
| 持久化 | 支持 RDB/AOF | 不支持 |
| 集群 | 原生支持 | 需客户端实现 |
| 发布订阅/Lua/事务 | 支持 | 不支持 |

## 数据类型与使用场景

### 五种核心数据类型

| 类型 | 使用场景 |
|------|---------|
| **String** | 缓存对象、常规计数、分布式锁、共享 session |
| **Hash** | 缓存对象、购物车 |
| **List** | 消息队列（需自行实现全局 ID） |
| **Set** | 点赞、共同关注、抽奖（交集/并集/差集） |
| **ZSet** | 排行榜、排序场景 |

### 四种扩展类型

| 类型 | 版本 | 使用场景 |
|------|------|---------|
| **BitMap** | 2.2 | 签到、用户登录状态、连续签到统计 |
| **HyperLogLog** | 2.8 | 百万级网页 UV 计数 |
| **GEO** | 3.2 | 地理位置（滴滴叫车） |
| **Stream** | 5.0 | 消息队列（自动全局 ID + 消费组） |

## 底层数据结构实现

### String → SDS（简单动态字符串）

SDS 相比 C 字符串的优势：
- **二进制安全**：可保存图片、音频等二进制数据
- **O(1) 获取长度**：`len` 字段记录长度
- **防止缓冲区溢出**：拼接前自动检查扩容

### Hash → ziplist / hashtable

- 键值对数量 < 512 且值 < 64 字节 → **ziplist**（压缩列表）
- 否则 → **hashtable**（字典）
- Redis 7.0 中 ziplist 已废弃，改用 **listpack**

### List → quicklist

- Redis 3.2 之前：ziplist 或双向链表
- Redis 3.2 之后：统一使用 **quicklist**

### Set → intset / hashtable

- 元素全是整数且数量 < 512 → **intset**（整数集合）
- 否则 → **hashtable**

### ZSet → ziplist / skiplist + dict

- 元素数量 < 128 且成员长度 < 64 字节 → **ziplist**
- 否则 → **zset**（skiplist + dict 复合结构）
  - dict：成员 → 分值映射（O(1) 查找）
  - skiplist：按分值排序（O(logN) 范围查询）
- Redis 7.0 中 ziplist 改为 **listpack**

### 跳跃表（Skiplist）

- 查找复杂度：平均 O(logN)，最坏 O(N)
- 原理：在有序链表上增加多级索引，查找时从高层开始
- 涉及结构体：`zskiplist`（头尾指针、长度、最高层级）、`zskiplistNode`

### 压缩列表（ziplist）

- Redis 为节约内存设计的线性数据结构
- 由连续内存块构成，每个节点可保存字节数组或整数值
- `previous_entry_length` 字段支持从表尾向表头遍历

### 字典（dict）与渐进式 Rehash

字典结构：`dict` → `dictht`（哈希表）→ `dictEntry`（哈希表节点）

Rehash 触发条件：
- 未执行 BGSAVE/BGREWRITEOF 且负载因子 ≥ 1
- 正在执行 BGSAVE/BGREWRITEOF 且负载因子 ≥ 5

渐进式 Rehash 步骤：
1. 为 `ht[1]` 分配空间
2. `rehashidx` 设为 0
3. 每次操作顺带迁移 `ht[0]` 中 `rehashidx` 位置的键值对到 `ht[1]`
4. 全部迁移完成后 `rehashidx = -1`

Rehash 期间访问原则：
- 新增 → 写入 `ht[1]`
- 查询/修改/删除 → 先查 `ht[0]`，再查 `ht[1]`

## 常见问题

- **Q：为什么 Redis 选择单线程？**
  A：CPU 不是瓶颈（瓶颈在内存和网络），单线程避免锁竞争和上下文切换。

- **Q：ZSet 为什么同时用 skiplist 和 dict？**
  A：dict 保证 O(1) 按成员查找分值，skiplist 保证 O(logN) 按分值范围查询，空间换时间。

- **Q：ziplist 和 listpack 的区别？**
  A：listpack 是 ziplist 的改进版，解决了 ziplist 的连锁更新问题，Redis 7.0 中替代 ziplist。

## 关联笔记

- [[00 Redis 知识库索引|Redis 知识库索引]]
- [[04 知识资料/数据库/SQL/00 SQL 知识库索引|SQL 知识库]]
- [[04 知识资料/知识库总索引|知识库总索引]]
