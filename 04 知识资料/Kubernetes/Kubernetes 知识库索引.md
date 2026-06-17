---
type: moc
status: evergreen
updated: 2026-06-17
tags:
  - 知识库
  - Kubernetes
  - 知识库索引
---

# Kubernetes 知识库索引

> Kubernetes 容器编排平台的学习笔记、部署手册和面试题。

## 一句话总结

Kubernetes 是容器编排的事实标准，核心掌握：Pod/Deployment/Service 等核心对象、调度与网络、存储、安全、以及生产级集群部署。

## 知识库目录

1. [[04 知识资料/Kubernetes/Kubernetes 学习|Kubernetes 学习]] — 核心概念、组件、对象管理
2. [[04 知识资料/Kubernetes/Kubernetes 问题|Kubernetes 问题]] — 常见问题收集
3. [[04 知识资料/Kubernetes/Kubernetes 答案|Kubernetes 答案]] — 问题解答
4. [[04 知识资料/Kubernetes/Kubernetes 1.36 单 Master 双 Node 集群部署手册|Kubernetes 1.36 部署手册]] — 实战部署

## 核心地图

```text
Kubernetes 集群
├── 控制平面 (Control Plane)
│   ├── kube-apiserver — 统一入口
│   ├── etcd — 分布式 KV 存储
│   ├── kube-scheduler — Pod 调度
│   └── kube-controller-manager — 控制器
└── 工作节点 (Worker Nodes)
    ├── kubelet — 节点代理
    ├── kube-proxy — 网络代理
    └── Container Runtime — 容器运行时
```

## 精读优先级

### 高频必看
- [[04 知识资料/Kubernetes/Kubernetes 学习|Kubernetes 学习]] — 基础概念
- [[04 知识资料/Kubernetes/Kubernetes 1.36 单 Master 双 Node 集群部署手册|部署手册]] — 动手实践

### 面试准备
- [[04 知识资料/Kubernetes/Kubernetes 问题|Kubernetes 问题]]
- [[04 知识资料/Kubernetes/Kubernetes 答案|Kubernetes 答案]]

## 适合沉淀的问题

- Pod 的生命周期是怎样的？
- Service 的 ClusterIP、NodePort、LoadBalancer 有什么区别？
- Deployment 和 StatefulSet 的适用场景？
- Kubernetes 网络模型（CNI）如何工作？
- 如何实现滚动更新和回滚？

## 后续整理建议

- [ ] 将「学习」和「问题/答案」整合为结构化知识库（参照 Redis 拆分模式）
- [ ] 补充 Ingress、ConfigMap、Secret、PV/PVC 等专题
- [ ] 增加常见故障排查手册

## 关联笔记

- [[04 知识资料/运维/运维知识库索引|运维知识库]]
- [[04 知识资料/知识库总索引|知识库总索引]]
