---
title: Redis 线程模型
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

# Redis 线程模型

## 一句话结论

Redis 的网络 IO 和命令执行在 6.0 之前是单线程的，通过 IO 多路复用（epoll）实现高并发；6.0 后引入多线程 IO 但命令执行仍保持单线程，性能瓶颈主要在内存和网络而非 CPU。

## Redis 是单线程吗？

**"单线程"的准确含义**：Redis 的「网络 IO」和「键值对读写」由主线程完成。

但实际上 Redis 不是纯粹的单线程：
- **2.6 版本**：启动 2 个后台线程（关闭文件、AOF 刷盘）
- **4.0 版本**：新增 lazyfree 后台线程（异步释放内存）
- **6.0 版本**：新增 IO 多线程（默认 3 个 IO 线程）

## 为什么单线程还这么快？

1. **内存操作**：大部分操作在内存中完成，CPU 不是瓶颈
2. **高效数据结构**：SDS、skiplist、quicklist 等
3. **IO 多路复用**：一个线程通过 epoll 处理多个客户端连接
4. **避免竞争**：无锁、无上下文切换开销

## IO 多路复用机制

Redis 使用 epoll 实现一个线程处理多个 IO 流：

### 初始化流程
1. `epoll_create()` 创建 epoll 对象
2. `socket()` + `bind()` + `listen()` 创建服务端 socket
3. `epoll_ctl()` 将 listen socket 加入 epoll，注册「连接事件」

### 事件循环
1. 处理发送队列（write）
2. `epoll_wait()` 等待事件：
   - **连接事件** → accept → 注册读事件
   - **读事件** → read → 解析命令 → 处理 → 加入发送队列
   - **写事件** → write 发送响应

## Redis 6.0 的多线程 IO

### 为什么引入？
网络硬件性能提升后，网络 IO 成为瓶颈。

### 多线程范围
- ✅ 网络 IO 读写 → 多线程
- ❌ 命令执行 → 仍单线程

### 配置
- `io-threads`：IO 线程数（建议 ≤ CPU 核数，4 核设 2-3，8 核设 6）
- `io-threads-do-reads`：是否多线程处理读请求（默认仅写）

### 线程构成（默认）
- `redis-server`：主线程，执行命令
- `bio_close_file` / `bio_aof_fsync` / `bio_lazy_free`：3 个后台线程
- `io_thd_1` / `io_thd_2` / `io_thd_3`：3 个 IO 线程

## 常见问题

- **Q：为什么不直接用多线程？**
  A：多线程引入锁竞争、上下文切换、死锁风险，而 Redis 的瓶颈在内存和网络，不在 CPU。

- **Q：lazyfree 是什么？**
  A：异步释放内存的机制。使用 `unlink key` 代替 `del key` 可避免删除大 key 时阻塞主线程。

- **Q：IO 线程数设置多少合适？**
  A：官方建议线程数 < CPU 核数，4 核设 2-3，8 核设 6。

## 关联笔记

- [[00 Redis 知识库索引|Redis 知识库索引]]
- [[05 Redis 过期删除与内存淘汰|Redis 过期删除与内存淘汰]]
- [[04 知识资料/知识库总索引|知识库总索引]]
