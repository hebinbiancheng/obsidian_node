---
title: GMP 核心结构详设
source: https://zhuanlan.zhihu.com/p/869632834
created: 2026-06-12
tags:
  - Go
  - GMP
  - Runtime源码
  - runtime2.go
status: 已整理
---

# GMP 核心结构详设

> 文章源码版本以 Go `v1.19` 为主，核心定义多位于 `runtime/runtime2.go`。

## 1. g 结构

`g` 是 goroutine 的 runtime 表示。

核心字段可以按用途分为几类：

| 字段 / 类型 | 作用 |
|---|---|
| `stack` | G 的栈空间范围 |
| `stackguard0` | 栈保护边界，也可用于传递抢占标记 |
| `_panic` / `_defer` | panic / defer 链表 |
| `m` | 当前执行该 G 的 M |
| `sched` | 保存 G 的调度上下文，例如 PC、SP 等 |
| `atomicstatus` | G 当前生命周期状态 |
| `goid` | goroutine ID，runtime 内部使用 |
| `startpc` | G 的入口函数地址 |

### g 的核心理解

G 不是简单的函数调用，而是一个带有完整执行上下文的任务对象：

- 有自己的栈；
- 有自己的状态；
- 能被挂起和恢复；
- 能在不同 M 上被调度执行。

## 2. g 的常见状态

| 状态 | 含义 |
|---|---|
| `_Gidle` | 刚分配，还未初始化 |
| `_Grunnable` | 已就绪，等待调度 |
| `_Grunning` | 正在某个 M 上执行 |
| `_Gwaiting` | 阻塞等待，例如 channel、mutex、IO |
| `_Gsyscall` | 正在系统调用中 |
| `_Gdead` | 已结束，可回收 |
| `_Gcopystack` | 正在栈拷贝 |
| `_Gpreempted` | 被抢占 |

## 3. m 结构

`m` 是 Go runtime 对操作系统线程的抽象。

核心字段：

| 字段 | 作用 |
|---|---|
| `g0` | 与 M 伴生的调度 G，负责执行调度逻辑 |
| `curg` | 当前正在 M 上运行的普通 G |
| `p` | 当前绑定的 P |
| `nextp` | 即将绑定的 P |
| `oldp` | 系统调用前绑定的 P |
| `spinning` | 是否处于自旋找任务状态 |
| `lockedg` | 与该 M 锁定的 G |
| `mcache` | 当前 P 对应的内存分配缓存 |

### m 的核心理解

M 是真正执行代码的线程，但它不是一直执行用户代码。它在两个角色间切换：

```text
执行 g0：找任务、调度、切换
执行普通 g：运行用户代码
```

## 4. p 结构

`p` 是 processor，是调度资源和本地队列的持有者。

核心字段：

| 字段 | 作用 |
|---|---|
| `status` | P 当前状态 |
| `m` | 当前绑定的 M |
| `runq` | 本地运行队列 lrq |
| `runqhead/runqtail` | 本地队列头尾索引 |
| `runnext` | 下一个优先运行的 G |
| `mcache` | 当前 P 的本地内存缓存 |
| `gcBgMarkWorker` | GC 后台标记 worker |

### p 的常见状态

| 状态 | 含义 |
|---|---|
| `_Pidle` | 空闲 |
| `_Prunning` | 正在被 M 持有并运行 |
| `_Psyscall` | 绑定的 M 进入系统调用 |
| `_Pgcstop` | GC stop-the-world 阶段停止 |
| `_Pdead` | 不再使用 |

## 5. schedt 全局调度器

`schedt` 是全局调度状态。

它维护：

- 全局运行队列 `runq`；
- 空闲 M 列表；
- 空闲 P 列表；
- 全局锁；
- netpoll 相关状态；
- 系统监控和调度统计信息。

## 6. g / m / p 的关系

```text
P 持有可运行 G 队列
M 绑定 P 后，从 P 或全局调度器拿 G
M 通过 g0 执行调度逻辑
M 通过 gogo 切换到普通 G 执行用户代码
普通 G 结束、阻塞或被抢占后，再切回 g0
```

## 7. 为什么 P 很关键

如果只有 G 和 M，会导致大量 goroutine 直接争抢全局队列，调度和内存分配竞争都很高。

P 的价值：

- 提供本地运行队列，降低全局锁竞争；
- 提供本地 mcache，降低内存分配竞争；
- 控制并行度；
- 让 M 的执行必须纳入 Go runtime 调度体系。

## 面试表达

> GMP 中，G 是任务，M 是执行线程，P 是调度资源。M 必须绑定 P 才能执行 G。P 维护本地运行队列和本地缓存，减少全局竞争，并通过 GOMAXPROCS 控制并行度。每个 M 有一个 g0，用于执行调度逻辑；普通 G 执行用户代码，两者通过 mcall、systemstack、gogo 等机制切换。
