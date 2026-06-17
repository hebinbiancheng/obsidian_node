---
title: Go GMP 知识库索引
source: https://zhuanlan.zhihu.com/p/869632834
author: 小徐先生
created: 2026-06-12
tags:
  - Go
  - Golang
  - GMP
  - Runtime
  - 并发
  - 知识库索引
status: 已整理
---

# Go GMP 知识库索引

> 来源文章：[[07 已归档/原始笔记/温故知新——Golang GMP 万字洗髓经.md]]  
> 原文链接：<https://zhuanlan.zhihu.com/p/869632834>  
> 本知识库按“概念 → 结构 → 调度 → 让渡 → 抢占 → 面试复盘”的方式拆解。

## 一句话总结

GMP 是 Go runtime 的核心调度模型：`G` 表示 goroutine 任务，`M` 表示操作系统线程，`P` 表示调度器和本地队列资源。Go 通过 GMP 把协程调度、网络 IO、内存分配、GC、安全点和抢占统一到 goroutine 粒度上，从而实现高并发、低开销、可伸缩的运行时调度。

## 学习路径

1. [[01 GMP 基础概念]]：线程、协程、goroutine、G/M/P 的基本含义。
2. [[02 GMP 核心结构详设]]：`g`、`m`、`p`、`schedt` 的源码字段和状态。
3. [[03 GMP 调度原理]]：从 `go func` 创建 G，到 `g0 -> g` 执行的完整流程。
4. [[04 GMP 让渡设计]]：G 如何主动放弃执行权并回到 g0。
5. [[05 GMP 抢占设计]]：sysmon 如何处理系统调用和运行超时抢占。
6. [[06 GMP 面试要点与复盘]]：高频问答、流程图、记忆框架。

## 核心地图

```text
用户代码 go func
        ↓
创建 G
        ↓
放入 P 的本地队列 lrq，满了进入全局队列 grq
        ↓
M 绑定 P 后，通过 g0 执行 schedule
        ↓
findRunnable 查找可运行 G
        ↓
execute + gogo：g0 -> g
        ↓
G 执行用户函数
        ↓
结束 / Gosched / gopark / 抢占
        ↓
g -> g0
        ↓
进入下一轮调度
```

## 关键词

- `G`：goroutine，用户态任务。
- `M`：machine，操作系统线程。
- `P`：processor，调度资源与本地运行队列。
- `g0`：每个 M 伴生的调度 goroutine。
- `lrq`：P 的本地运行队列。
- `grq`：全局运行队列。
- `schedule`：调度入口。
- `findRunnable`：寻找可运行 G。
- `execute` / `gogo`：从 g0 切换到普通 G。
- `mcall` / `systemstack`：从普通 G 切换到 g0。
- `gopark` / `goready`：阻塞与唤醒。
- `sysmon`：监控线程，负责 netpoll、retake、抢占等。

## 适合回答的问题

- Go 为什么不用“线程池”直接调度 goroutine？
- G、M、P 分别是什么？为什么要引入 P？
- goroutine 是如何被创建、入队和执行的？
- g0 是什么？为什么普通 G 和 g0 要来回切换？
- `findRunnable` 的查找顺序是什么？
- 本地队列、全局队列、work stealing 如何配合？
- goroutine 阻塞在 channel / mutex 时发生什么？
- Go 如何处理系统调用导致的 M 阻塞？
- Go 如何抢占运行过久的 goroutine？
- netpoll 如何融入 GMP 调度？

## 后续维护建议

- 遇到 Go 并发、runtime、调度相关内容，优先补充到本知识库。
- 面试题放入 [[06 GMP 面试要点与复盘]]。
- 源码细节、版本差异、流程图可以继续拆成子笔记。
- 如果后续阅读 Go 1.20+ / 1.22+ runtime，需要单独标注版本差异。
