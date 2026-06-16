---
title: IB 与 RoCE 对比知识库索引
source:
  - https://mp.weixin.qq.com/s/gq-8YRI2KuB-1TZyVNauCg
  - https://mp.weixin.qq.com/s/46Mxg3cyEkt_49PefLLAIA
created: 2026-06-16
tags:
  - AI
  - 智算中心
  - 数据中心网络
  - RDMA
  - InfiniBand
  - RoCE
  - 无损网络
status: 已整理
---

# IB 与 RoCE 对比知识库索引

> 整合来源：
> - [InfiniBand(IB网络)与以太网(RoCE)差异对比--看完让你对架构/财务选型轻松决策](https://mp.weixin.qq.com/s/gq-8YRI2KuB-1TZyVNauCg)，作者：IT课代表，发布：2026-06-13
> - [【收藏】智算中心面试真题，第一道就把人问懵了](https://mp.weixin.qq.com/s/46Mxg3cyEkt_49PefLLAIA)，作者：2026一起加油，发布：2026-06-02

## 一句话总结

IB 买的是**低延迟、确定性、开箱即用和训练效率**；RoCEv2 买的是**以太网生态、成本优势、开放供应链和规模化运维弹性**。在 AI 训练集群中，选型不是简单比较“谁更快”，而是要综合 GPU 规模、训练窗口、预算、团队网络能力、供应链绑定风险与未来演进路线。

## 知识库目录

1. [[01 核心概念与选型总览]]
2. [[02 协议栈、流控与无损网络机制]]
3. [[03 性能、成本、生态与场景决策]]
4. [[04 智算中心面试答题框架]]
5. [[05 运维复杂度、排障与未来趋势]]
6. [[06 原文解析要点与图片索引]]

## 核心地图

```text
AI 训练通信
  └─ RDMA：绕过 CPU / 内核，降低通信开销
      ├─ InfiniBand：专有网络 + HCA + Credit 流控 + SM 管理
      │   ├─ 优势：低延迟、长尾稳定、天然无损、训练确定性强
      │   └─ 代价：成本高、生态封闭、供应商绑定、人才稀缺
      └─ RoCEv2：RDMA over UDP/IP Ethernet
          ├─ 优势：以太网生态、成本低、可路由、供应链开放
          └─ 代价：PFC/ECN/DCQCN 调优复杂，长尾与拥塞更难控
```

## 原文关键图片

### IB/RoCE 选型视角

![[assets/a1-01-ib-roce-title.png]]

### 智算中心面试视角

![[assets/a2-06-lossless-network-diagram.png]]

## 适合沉淀的问题

- 为什么 AI 训练网络不能简单套用传统以太网经验？
- IB 的 Credit 流控为什么比 RoCE 的 PFC 更“确定”？
- RoCEv2 如何通过 PFC + ECN + DCQCN 实现准无损网络？
- 什么时候应该为 IB 的成本买单？什么时候 RoCE 更合理？
- 面试中如何从协议栈、流控、无损、性能、运维五个维度回答 IB vs RoCE？

## 关联笔记

- [[04 知识资料/AI/AI 知识库索引.md]]
- [[05 网页与链接收藏/解析规则/知识库式解析规范.md]]
