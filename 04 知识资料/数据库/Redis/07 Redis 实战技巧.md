---
title: Redis 实战技巧
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

# Redis 实战技巧

## 一句话结论

Redis 实战中常见的进阶技巧包括：延迟队列（ZSet）、大 Key 处理（分批/异步删除）、分布式锁（SETNX + Lua）、管道（批量命令）、事务（无回滚）、分布式 Session 等。

## 延迟队列

### 实现方式
使用 ZSet，Score 存储延迟执行的时间戳：
```bash
zadd delay_queue <timestamp> <task_data>
# 定时任务扫描
zrangebyscore delay_queue 0 <current_timestamp> limit 0 10
```

### 使用场景
- 订单超时未支付自动取消
- 打车超时未接单自动取消
- 外卖商家超时未接单

## 大 Key 处理

### 什么是大 Key？
- String 类型 value > 10KB
- 集合类型元素 > 5000 个

### 大 Key 的影响
1. **客户端超时阻塞**：操作耗时长
2. **网络阻塞**：大流量（1MB × 1000 QPS = 1GB/s）
3. **阻塞工作线程**：del 删除大 key 时阻塞
4. **内存分布不均**：集群中部分节点压力大

### 查找大 Key

| 方法 | 命令/工具 | 注意 |
|------|----------|------|
| redis-cli | `redis-cli --bigkeys` | 仅返回每种类型最大的 key |
| SCAN 扫描 | `SCAN` + `STRLEN`/`MEMORY USAGE` | 更精确，需编程实现 |
| RdbTools | 第三方工具 | 解析 RDB 文件 |

### 删除大 Key

| 方法 | 说明 |
|------|------|
| **分批次删除** | Hash: hscan + hdel；List: ltrim；Set: sscan + srem；ZSet: zremrangebyrank |
| **异步删除**（4.0+） | `unlink key` 代替 `del key` |

### 异步删除配置（建议开启）
```conf
lazyfree-lazy-eviction yes      # 内存淘汰时异步删除
lazyfree-lazy-expire yes         # 过期键异步删除
lazyfree-lazy-server-del yes     # rename 等隐式删除异步
```

## 分布式锁

### 单节点实现
```bash
# 加锁（原子操作：不存在才设置 + 设过期时间 + 唯一标识）
SET lock_key unique_value NX PX 10000

# 解锁（Lua 脚本保证原子性：验证 value 再删除）
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

### 三个关键点
1. **NX**：key 不存在才设置（互斥）
2. **EX/PX**：设过期时间（防死锁）
3. **唯一 value**：解锁时验证身份（防误删）

### 锁超时问题
- 业务执行时间 > 锁超时时间 → 锁自动释放，其他线程获取锁
- 解决方案：**续约机制**（守护线程定期延长锁时间）

### Redlock（红锁）— 集群分布式锁
- 至少 5 个独立 Redis 主节点
- 加锁：依次向 N 个节点加锁，超半数成功且总耗时 < 锁过期时间 → 成功
- 解锁：向所有节点发送 DEL

### 优缺点
- ✅ 性能高效、实现方便、避免单点故障
- ❌ 超时时间难设置、主从异步复制导致锁不可靠

## Redis 管道（Pipeline）

### 原理
客户端将多个命令打包发送，服务端批量处理后统一返回，减少网络 RTT。

```
普通模式：CMD1 → 等待 → CMD2 → 等待 → CMD3 → 等待
管道模式：CMD1 + CMD2 + CMD3 → 发送 → 批量处理 → 统一返回
```

### 注意
- 管道是**客户端**功能，非服务端
- 避免单次管道命令过多导致网络阻塞

## Redis 事务

### 特点
- **不支持回滚**：DISCARD 只能放弃事务，不能回滚已执行命令
- **不保证原子性**：入队时报错全部不执行，执行时报错正确命令仍执行

### 为什么不支持回滚？
1. 错误通常是编程错误，生产环境少见
2. 与 Redis 简单高效的设计理念不符

### WATCH 命令
乐观锁机制：监视 key，事务执行前若 key 被修改则拒绝执行。

## 分布式 Session

### 问题
多服务器时，Session 只存在于创建它的服务器上，其他服务器无法验证用户身份。

### 解决方案
使用 Redis 存储 Session：
1. **创建令牌**：用户初次访问时生成唯一标识，存 Redis，通过 Cookie 返回
2. **验证令牌**：后续请求从 Cookie 取标识，查 Redis 验证

## 常见问题

- **Q：del 和 unlink 的区别？**
  A：del 在主线程同步删除，可能阻塞；unlink 异步删除，不阻塞主线程。

- **Q：分布式锁用 SETNX 有什么问题？**
  A：SETNX 无法同时设置过期时间（需两次操作，非原子）。应用 `SET key value NX PX` 代替。

- **Q：管道和事务的区别？**
  A：管道是批量发送减少 RTT，不保证原子性；事务保证命令连续执行但不支持回滚。

## 关联笔记

- [[00 Redis 知识库索引|Redis 知识库索引]]
- [[04 Redis 集群与高可用|Redis 集群与高可用]]
- [[06 Redis 缓存设计|Redis 缓存设计]]
- [[04 知识资料/知识库总索引|知识库总索引]]
