---
type: knowledge
status: evergreen
created: "2026-06-17"
updated: "2026-06-17"
tags:
  - 知识库
  - Kubernetes
  - 容器
  - 运维
  - 索引
aliases:
  - Kubernetes 知识库索引
confidence: high
---

# 00 Kubernetes 知识库索引

> Kubernetes 容器编排平台的结构化知识库，涵盖架构、对象管理、节点、控制器、面试问答与集群部署。

## 一句话总结

Kubernetes 是容器编排的事实标准，核心掌握：控制平面与节点组件架构、对象管理（标签/注解/Finalizers）、调度与控制器模式、以及基于 kubeadm 的生产级集群部署。

## 知识库目录

| 序号 | 笔记 | 内容概要 |
|------|------|----------|
| 01 | [[04 知识资料/Kubernetes/结构化笔记/01 Kubernetes 核心架构与组件\|01 Kubernetes 核心架构与组件]] | 控制平面、节点组件、插件 |
| 02 | [[04 知识资料/Kubernetes/结构化笔记/02 Kubernetes 对象与资源管理\|02 Kubernetes 对象与资源管理]] | 对象管理、标签/选择器、名字空间、注解、字段选择器、Finalizers |
| 03 | [[04 知识资料/Kubernetes/结构化笔记/03 Kubernetes 节点管理\|03 Kubernetes 节点管理]] | 节点自注册、节点控制器、逐出速率、资源容量、节点通信 |
| 04 | [[04 知识资料/Kubernetes/结构化笔记/04 Kubernetes 控制器与垃圾收集\|04 Kubernetes 控制器与垃圾收集]] | 控制器模式、属主引用、级联删除、租约 |
| 05 | [[04 知识资料/Kubernetes/结构化笔记/05 Kubernetes 面试问答精粹\|05 Kubernetes 面试问答精粹]] | 10 个高频面试问题与详细解答 |
| 06 | [[04 知识资料/Kubernetes/结构化笔记/06 Kubernetes 集群部署实战\|06 Kubernetes 集群部署实战]] | kubeadm 部署关键步骤与排障 |

## 核心地图

```text
Kubernetes 集群
├── 控制平面 (Control Plane)
│   ├── kube-apiserver — 统一入口，所有操作的中枢
│   ├── etcd — 分布式 KV 存储，集群状态持久化
│   ├── kube-scheduler — Pod 调度决策
│   └── kube-controller-manager — 运行各类控制器
└── 工作节点 (Worker Nodes)
    ├── kubelet — 节点代理，管理 Pod 生命周期
    ├── kube-proxy — 网络代理，实现 Service 规则
    └── Container Runtime — 容器运行时（containerd 等）
```

## 精读优先级

### 第一梯队：必读基础
- [[04 知识资料/Kubernetes/结构化笔记/01 Kubernetes 核心架构与组件|01 核心架构与组件]] — 理解集群骨架
- [[04 知识资料/Kubernetes/结构化笔记/02 Kubernetes 对象与资源管理|02 对象与资源管理]] — 日常操作基础

### 第二梯队：深入理解
- [[04 知识资料/Kubernetes/结构化笔记/03 Kubernetes 节点管理|03 节点管理]] — 节点生命周期
- [[04 知识资料/Kubernetes/结构化笔记/04 Kubernetes 控制器与垃圾收集|04 控制器与垃圾收集]] — 自动化机制

### 第三梯队：实战应用
- [[04 知识资料/Kubernetes/结构化笔记/06 Kubernetes 集群部署实战|06 集群部署实战]] — 动手部署
- [[04 知识资料/Kubernetes/结构化笔记/05 Kubernetes 面试问答精粹|05 面试问答精粹]] — 面试准备

## 适合沉淀的问题

- Pod 的完整生命周期是怎样的？（创建 → Running → 终止 → 清理）
- Service 的 ClusterIP、NodePort、LoadBalancer 有什么区别？
- Deployment 和 StatefulSet 的适用场景分别是什么？
- Kubernetes 网络模型（CNI）如何工作？
- Finalizers 和级联删除的机制是怎样的？
- 节点故障时，Kubernetes 如何检测和响应？

## 后续整理建议

- [ ] 补充 Ingress / Gateway API 专题
- [ ] 补充 ConfigMap、Secret、PV/PVC 存储专题
- [ ] 补充 HPA/VPA 弹性伸缩专题
- [ ] 增加常见故障排查手册
- [ ] 补充 RBAC 安全专题

## 关联笔记

- [[04 知识资料/运维/运维知识库索引|运维知识库]]
- [[04 知识资料/知识库总索引|知识库总索引]]

## 2026-06-24 补充专题

- [[04 知识资料/Kubernetes/Kubernetes 面试与排障专题/00 Kubernetes 面试与排障专题索引|Kubernetes 面试与排障专题]]：由 8 篇公众号文章整理，覆盖容器基础、Pod 创建流程、CoreDNS、排障方法、kubectl 命令与面试题。
