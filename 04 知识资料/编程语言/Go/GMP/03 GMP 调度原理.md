---
title: GMP 调度原理
source: https://zhuanlan.zhihu.com/p/869632834
created: 2026-06-12
tags:
  - Go
  - GMP
  - 调度
  - schedule
  - findRunnable
status: 已整理
---

# GMP 调度原理

## 1. 调度是什么

这里的“调度”指：一个通过 `go func(){...}` 创建出的 G，如何被 M 上的 g0 找到，并切换成普通 G 执行。

本质流程是：

```text
g0 -> 普通 g
```

这是第一视角的转换：由 M 上正在运行的 g0 主动发起。

## 2. main 函数与第一个 G

Go 程序启动时，main 函数并不是直接裸跑在线程上，而是作为一个 goroutine 进入 runtime 调度体系。

启动链路可以概括为：

```text
m0 / g0 启动 runtime
        ↓
创建 main goroutine
        ↓
进入 schedule
        ↓
调度 main goroutine 执行
```

这说明 Go 从程序入口开始，就把执行统一纳入 GMP。

## 3. 创建 G：newproc

当用户执行：

```go
go func() {
    // user code
}()
```

runtime 会通过 `newproc` 创建新的 G。

典型动作：

1. 切换到 g0 或 system stack 执行创建逻辑；
2. 分配或复用一个 G；
3. 初始化 G 的栈、入口函数、调度上下文；
4. 将 G 状态置为 runnable；
5. 将 G 放入当前 P 的本地运行队列；
6. 必要时唤醒或创建 M 执行任务。

## 4. g0 与普通 g 的切换

每个 M 都有一个特殊的 g0。

| 切换方向 | 方法 | 含义 |
|---|---|---|
| 普通 g -> g0 | `mcall` | 普通 G 主动切到 g0，通常用于结束、让渡、阻塞 |
| 普通 g -> g0 -> 普通 g | `systemstack` | 临时切到 g0 栈执行 runtime 逻辑，结束后回到原 G |
| g0 -> 普通 g | `gogo` | 恢复普通 G 的执行上下文 |

## 5. schedule + execute

从 g0 的视角看，调度主要经过两个核心函数：

```text
schedule -> findRunnable -> execute -> gogo
```

### schedule

`schedule` 是调度主循环入口。它负责找到一个可运行 G，然后交给 `execute`。

### findRunnable

`findRunnable` 负责按优先级查找可运行 G。

### execute

`execute` 会：

- 绑定 G 与 M；
- 更新 G 状态为 running；
- 设置调度上下文；
- 调用 `gogo`，从 g0 切换到普通 G。

## 6. findRunnable 的查找顺序

文章中总结的核心顺序：

1. 每隔一定调度次数，先查一次全局队列，避免 grq 饥饿；
2. 从当前 P 的本地队列 `lrq` 获取；
3. 从全局队列 `grq` 获取；
4. 从 netpoll 获取 IO 就绪的 G；
5. 从其他 P 的本地队列偷取；
6. double check 全局队列；
7. 如果仍无任务，保留 M 以阻塞模式执行 netpoll 或进入休眠。

## 7. 本地队列 runqget

当前 P 的本地队列优先级最高。

原因：

- 访问快；
- 竞争少；
- 多数情况下无需全局锁；
- 符合缓存局部性。

## 8. 全局队列 globrunqget

当本地队列没有可运行 G 时，会尝试从全局队列获取。

全局队列访问需要加锁，所以不能作为最高频路径。

调度器会定期优先检查全局队列，避免本地队列繁忙导致全局队列中的 G 长期饥饿。

## 9. netpoll

当运行队列中没有 G 时，runtime 会尝试通过 netpoll 获取 IO 就绪的 G。

核心思想：

```text
网络 IO 阻塞不阻塞 M，而是 park 当前 G。
IO 就绪后，通过 netpoll + goready 把 G 放回可运行队列。
```

这样 Go 可以把 IO 阻塞控制在 goroutine 粒度，而不是线程粒度。

## 10. Work Stealing

如果本地队列、全局队列和 netpoll 都没有任务，当前 P 会尝试从其他 P 的本地队列偷取一部分 G。

作用：

- 平衡各 P 的任务负载；
- 避免某些 P 很忙、某些 P 空闲；
- 提高整体吞吐。

## 调度流程图

```text
schedule
  ↓
findRunnable
  ├─ 定期检查 grq
  ├─ 查当前 P 的 lrq
  ├─ 查全局 grq
  ├─ 查 netpoll 就绪 G
  ├─ stealWork 从其他 P 偷 G
  ├─ double check grq
  └─ 无任务则休眠 / 阻塞 netpoll
  ↓
execute
  ↓
gogo
  ↓
普通 G 执行用户代码
```

## 高频问题

### 为什么先查本地队列？

因为本地队列竞争低、缓存友好、无需频繁加全局锁。

### 为什么还需要全局队列？

本地队列满时需要兜底；某些全局 runnable G 也需要统一存放。调度器会定期检查全局队列避免饥饿。

### 为什么要 work stealing？

为了负载均衡，避免某个 P 任务堆积，而其他 P 空闲。
