---
title: GMP 让渡设计
source: https://zhuanlan.zhihu.com/p/869632834
created: 2026-06-12
tags:
  - Go
  - GMP
  - 让渡
  - gopark
  - Gosched
status: 已整理
---

# GMP 让渡设计

## 1. 什么是让渡

让渡指普通 G 在 M 上运行时，主动让出执行权，使 M 的运行对象重新回到 g0。

本质流程：

```text
普通 g -> g0
```

让渡是第一视角转换：由正在运行的 G 主动发起，不需要 sysmon 等第三方干预。

## 2. 三类让渡

| 类型 | 触发场景 | 关键函数 | G 后续状态 |
|---|---|---|---|
| 结束让渡 | G 函数执行完毕 | `goexit1` / `goexit0` | dead，可回收 |
| 主动让渡 | 用户调用 `runtime.Gosched()` | `Gosched` / `gosched_m` | runnable，重新排队 |
| 阻塞让渡 | channel、mutex、IO 等阻塞 | `gopark` / `park_m` | waiting，等待唤醒 |

## 3. 结束让渡

当 G 的用户函数执行完毕后，会调用 `goexit1`，通过 `mcall` 切换到 g0，再由 g0 执行 `goexit0`。

核心动作：

1. 清理 G 上的 defer / panic 等信息；
2. 解除 G 与 M 的绑定；
3. 将 G 状态置为 dead；
4. 回收或放入复用池；
5. 调用 `schedule` 开始下一轮调度。

流程：

```text
普通 G 执行结束
  ↓
goexit1
  ↓
mcall(goexit0)
  ↓
g0 清理 G
  ↓
schedule
```

## 4. 主动让渡：Gosched

用户可以显式调用：

```go
runtime.Gosched()
```

它表示当前 G 主动让出 CPU，让调度器先调度其他 G。

核心流程：

```text
Gosched
  ↓
mcall(gosched_m)
  ↓
g0 将当前 G 放回 runnable
  ↓
schedule
```

注意：主动让渡不是阻塞，当前 G 仍然可运行，只是重新进入调度队列。

## 5. 阻塞让渡：gopark

channel、mutex、网络 IO 等阻塞场景底层会使用 `gopark`。

核心流程：

1. 普通 G 调用 `gopark`；
2. 通过 `mcall` 切到 g0；
3. g0 执行 `park_m`；
4. 将 G 状态从 running 改为 waiting；
5. 当前 G 不进入 lrq/grq；
6. g0 重新 schedule 其他 G。

关键点：

> 阻塞后的 G 不在就绪队列里。使用方必须保存 G 的引用，并在条件满足后通过 `goready` 唤醒。

## 6. 唤醒：goready

`goready` 与 `gopark` 相对，用于唤醒等待中的 G。

核心动作：

1. 切到 g0 / system stack；
2. 将目标 G 状态从 waiting 改为 runnable；
3. 将 G 放入可运行队列；
4. 等待后续调度。

流程：

```text
外部条件满足
  ↓
goready
  ↓
waiting -> runnable
  ↓
进入 lrq/grq
  ↓
等待 schedule
```

## 7. channel / mutex 与 gopark

很多 Go 并发原语都建立在 `gopark/goready` 模式上。

例如：

- channel 读写阻塞；
- mutex 等锁；
- timer 等待；
- netpoll IO 等待。

这些机制的共同点是：

```text
条件不满足：gopark 挂起 G
条件满足：goready 唤醒 G
```

## 面试表达

> GMP 中的让渡是普通 G 主动把执行权交还给 g0 的过程。常见类型包括函数结束、Gosched 主动让渡和 gopark 阻塞让渡。结束让渡会清理 G 并进入下一轮调度；Gosched 会把当前 G 重新放回 runnable；gopark 会把 G 置为 waiting，不进入就绪队列，后续必须由 goready 唤醒。
