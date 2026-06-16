---
title: GMP 面试要点与复盘
source: https://zhuanlan.zhihu.com/p/869632834
created: 2026-06-12
tags:
  - Go
  - GMP
  - 面试
  - 技术复习
status: 已整理
---

# GMP 面试要点与复盘

## 1. 必背核心

```text
G 是 goroutine，代表待执行任务。
M 是 OS thread，代表真正执行代码的线程。
P 是 processor，代表调度资源和本地队列。
M 必须绑定 P 才能执行普通 G。
每个 M 有一个 g0，g0 执行调度逻辑，普通 G 执行用户代码。
```

## 2. 一句话解释 GMP

> GMP 是 Go runtime 的用户态调度模型。它通过 P 控制并行度和本地队列，通过 M 执行代码，通过 G 表示 goroutine 任务，从而在用户态完成大量 goroutine 的创建、调度、阻塞、唤醒和抢占。

## 3. 为什么需要 P

如果只有 G 和 M，所有 G 都需要竞争全局队列，调度和内存分配都会产生较大锁竞争。

P 的价值：

- 持有本地运行队列，减少全局队列竞争；
- 持有 mcache，提升内存分配效率；
- 控制最大并行度；
- 作为 M 进入 Go 调度体系的令牌。

## 4. G 是怎么被调度的

```text
go func 创建 G
  ↓
优先放入当前 P 的本地队列
  ↓
M 绑定 P，通过 g0 执行 schedule
  ↓
findRunnable 找可运行 G
  ↓
execute 更新状态
  ↓
gogo 切换到普通 G 执行
```

## 5. findRunnable 查找顺序

1. 定期优先查全局队列，避免饥饿；
2. 查当前 P 的本地队列；
3. 查全局队列；
4. 查 netpoll 就绪 G；
5. 从其他 P 偷 G；
6. 再检查全局队列；
7. 无任务则休眠或阻塞 netpoll。

## 6. g0 是什么

g0 是每个 M 伴生的特殊 goroutine，不执行用户代码，而是执行 runtime 调度逻辑。

普通 G 和 g0 的关系：

```text
g0：找任务、切换、清理、调度
普通 G：执行用户函数
```

切换方式：

- `g0 -> g`：`gogo`；
- `g -> g0`：`mcall`；
- 临时切到 g0 栈：`systemstack`。

## 7. 让渡 vs 抢占

| 维度 | 让渡 | 抢占 |
|---|---|---|
| 发起方 | G 主动 | sysmon / runtime 外部干预 |
| 方向 | g -> g0 | g -> g0 |
| 场景 | 结束、Gosched、gopark | 系统调用过久、运行过久 |
| 目的 | 主动交出执行权 | 保证调度公平和系统活性 |

## 8. gopark / goready

- `gopark`：让 G 进入 waiting，不进入就绪队列；
- `goready`：把 waiting 的 G 改为 runnable，并放回可运行队列。

常见场景：

- channel 阻塞；
- mutex 等待；
- 网络 IO；
- timer。

## 9. 系统调用时发生什么

如果 G 进入系统调用，M 可能阻塞。

Go 不希望 P 被阻塞的 M 长期占用，所以 sysmon 会在必要时把 P 抢回，交给其他 M 继续调度其他 G。

## 10. netpoll 如何融入 GMP

网络 IO 不是让 M 长期阻塞，而是让 G park。

```text
G 等 IO
  ↓
gopark
  ↓
netpoll 检测 IO 就绪
  ↓
goready
  ↓
G 回到可运行队列
```

## 11. 常见面试题速答

### Q1：GMP 中 P 的数量由什么决定？

由 `GOMAXPROCS` 决定，默认通常与 CPU 核数相关。P 的数量决定 Go 程序同时并行执行 Go 代码的最大数量。

### Q2：M 的数量等于 P 吗？

不一定。P 控制并行度，M 是线程，数量可能因为系统调用、阻塞、runtime 需求而多于 P。

### Q3：goroutine 阻塞会阻塞线程吗？

不一定。channel、mutex、netpoll 等 runtime 可感知的阻塞通常会 park 当前 G，让 M/P 去执行其他 G。真正阻塞系统调用可能阻塞 M，但 runtime 会尝试把 P 交给其他 M。

### Q4：为什么全局队列不会饿死？

调度器会定期优先检查全局队列，例如每隔一定调度次数先取 grq，避免本地队列持续繁忙导致全局队列长期得不到执行。

### Q5：work stealing 偷多少？

通常会从其他 P 的本地队列中偷取一部分任务，用于平衡负载。具体策略随 Go 版本可能有差异。

## 12. 复习框架

```text
概念层：线程、协程、goroutine
结构层：G、M、P、schedt、g0
队列层：lrq、grq、netpoll、steal
流程层：newproc、schedule、findRunnable、execute、gogo
让渡层：goexit、Gosched、gopark、goready
抢占层：sysmon、syscall、retake、preempt
生态层：内存管理、GC、netpoll、channel、mutex
```

## 13. 和原文的关系

原文适合深度阅读源码，本笔记适合：

- 面试前快速复习；
- 建立 GMP 总体框架；
- 后续补充 runtime 源码细节；
- 作为 Go 并发知识库入口。
