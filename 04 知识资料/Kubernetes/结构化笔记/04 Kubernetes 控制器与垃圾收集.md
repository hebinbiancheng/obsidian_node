---
type: knowledge
status: evergreen
created: "2026-06-17"
updated: "2026-06-17"
tags:
  - 知识库
  - Kubernetes
  - 控制器
  - 垃圾收集
  - 租约
aliases:
  - Kubernetes 控制器与垃圾收集
confidence: high
---

# 04 Kubernetes 控制器与垃圾收集

> 控制器是 Kubernetes 的大脑，垃圾收集是 Kubernetes 的清洁工。两者协同工作，确保集群状态始终向期望状态收敛。

## 一句话结论

Kubernetes 控制器通过监控资源当前状态并与期望状态对比，驱动集群向目标收敛；垃圾收集通过属主引用和级联删除机制自动清理孤立资源；租约则用于分布式协调（心跳检测和领导者选举）。

## 控制器模式

### 核心原理

```
期望状态 (spec)  ──→  控制器  ──→  当前状态 (status)
                         │
                   对比差异，执行操作
```

一个控制器至少追踪一种 Kubernetes 资源类型，该资源有代表期望状态的 `spec` 字段。控制器负责确保当前状态接近期望状态。

### 两种控制方式

| 方式 | 机制 | 示例 |
|------|------|------|
| **通过 API Server 控制** | 控制器通知 API Server 创建/删除资源，不直接操作 Pod/容器 | Job 控制器 |
| **直接控制** | 控制器直接操作集群外部资源 | 自动扩缩节点数量的控制器 |

### Job 控制器示例

```
1. Job 对象被创建（spec 描述期望的 Pod 数量和任务）
        ↓
2. Job 控制器检测到新 Job
        ↓
3. Job 控制器通知 API Server 创建 Pod
        ↓
4. kubelet 在节点上运行 Pod
        ↓
5. Job 控制器监控 Pod 状态，直到任务完成
```

> ⚠️ Job 控制器**不会自己运行 Pod 或容器**，而是通过 API Server 间接管理。

### 多控制器共存

多个控制器可以创建/更新相同类型的对象（如 Deployment 和 Job 都可以创建 Pod），但通过**标签**区分归属，互不干扰。

## 属主引用（Owner Reference）

属主引用描述 Kubernetes 中对象之间的**父子关系**：

| 特性 | 属主引用 | 标签 |
|------|----------|------|
| 用途 | 确定对象归属，用于级联删除 | 分组和选择对象 |
| 删除行为 | 父对象删除时，子对象被垃圾收集 | 不影响 |
| 示例 | Job → Pod 的属主引用 | `app=nginx` 标签 |

> 当 Job 创建 Pod 时，Job 控制器同时添加标签（用于跟踪）和属主引用（用于级联清理）。删除 Job 时，Kubernetes 使用属主引用（而非标签）确定哪些 Pod 需要清理。

## 垃圾收集

### 清理范围

Kubernetes 垃圾收集负责清理：

| 资源类型 | 说明 |
|----------|------|
| 终止的 Pod | 已退出但未清理的 Pod |
| 已完成的 Job | 执行完毕的 Job 对象 |
| 无属主引用的对象 | 孤立资源（如删除 ReplicaSet 后遗留的 Pod） |
| 未使用的容器和镜像 | 节点上的冗余容器/镜像 |
| 动态制备的 PV | StorageClass 回收策略为 Delete 的 PV 卷 |
| 过期 CSR | 阻滞或过期的 CertificateSigningRequest |
| 节点租约对象 | 过期或无效的 Lease |

### 级联删除

删除对象时，可以控制是否自动删除其依赖对象：

| 模式 | 行为 | 默认 |
|------|------|------|
| **后台级联删除** | API Server 立即删除属主对象，垃圾收集控制器在后台清理依赖 | ✅ 默认 |
| **前台级联删除** | 属主对象先进入"删除进行中"状态，等所有依赖删除后再删除属主 | 需手动指定 |
| **孤立删除** | 只删除属主对象，保留依赖对象 | 需手动指定 |

### 前台级联删除流程

```
1. 用户发起 DELETE（前台级联）
        ↓
2. API Server 设置 metadata.deletionTimestamp
3. API Server 设置 metadata.finalizers = ["foregroundDeletion"]
        ↓
4. 属主对象进入 "deletion in progress" 状态（仍可通过 API 查看）
        ↓
5. 控制器删除所有已知依赖对象
        ↓
6. 所有依赖删除后，控制器删除属主对象
        ↓
7. 属主对象从 API 中消失
```

### 后台级联删除流程

```
1. 用户发起 DELETE（默认行为）
        ↓
2. API Server 立即删除属主对象
        ↓
3. 垃圾收集控制器在后台异步清理所有依赖对象
```

## 租约（Lease）

租约提供了一种机制来**锁定共享资源**和**协调集合成员之间的活动**。在 Kubernetes 中表示为 `coordination.k8s.io` API 组中的 `Lease` 对象。

### 两大用途

| 用途 | 机制 | 说明 |
|------|------|------|
| **节点心跳** | kubelet 定期更新 `kube-node-lease` 命名空间中对应 Lease 的 `spec.renewTime` | 控制平面据此判断节点可用性 |
| **领导者选举** | 组件通过 Lease 竞争领导权，确保同时只有一个实例活跃 | kube-controller-manager、kube-scheduler 等高可用组件使用 |

### 节点心跳机制

```
每个 Node  →  kube-node-lease 命名空间中有一个同名 Lease 对象
kubelet     →  定期 UPDATE 该 Lease 的 spec.renewTime 字段
控制平面    →  读取 renewTime 时间戳判断节点是否存活
```

## 常见问题

### Q1：前台级联删除和后台级联删除应该如何选择？

- **后台级联删除（默认）**：适合大多数场景，删除速度快，依赖异步清理
- **前台级联删除**：当你需要在删除完成前观察依赖对象的清理状态时使用，例如调试或审计场景
- **孤立删除**：当你希望保留依赖对象（如保留 Pod 用于排查问题）时使用

### Q2：Finalizers 和属主引用有什么区别？

Finalizers 是删除**前置条件**，阻止对象被删除直到特定清理完成。属主引用是对象间的**父子关系**，用于级联删除。两者可以协同工作：前台级联删除就是通过设置 `foregroundDeletion` Finalizer 来实现的。

### Q3：节点心跳使用 Lease 而不是直接更新 Node 对象有什么好处？

Lease 对象更轻量，更新开销小。在高频心跳场景下（kubelet 默认每 10 秒更新一次），使用轻量的 Lease 对象可以减少 etcd 和 API Server 的负载。Node 对象本身包含大量状态信息，频繁更新成本高。

## 关联笔记

- [[04 知识资料/Kubernetes/结构化笔记/00 Kubernetes 知识库索引|Kubernetes 知识库索引]]
- [[04 知识资料/Kubernetes/结构化笔记/02 Kubernetes 对象与资源管理|02 对象与资源管理]]
- [[04 知识资料/Kubernetes/结构化笔记/03 Kubernetes 节点管理|03 节点管理]]
