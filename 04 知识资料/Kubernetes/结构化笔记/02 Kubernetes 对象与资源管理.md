---
type: knowledge
status: evergreen
created: "2026-06-17"
updated: "2026-06-17"
tags:
  - 知识库
  - Kubernetes
  - 对象管理
  - 标签
  - 名字空间
aliases:
  - Kubernetes 对象与资源管理
confidence: high
---

# 02 Kubernetes 对象与资源管理

> Kubernetes 中一切皆对象。理解对象的创建、标识、组织与删除机制，是高效使用 Kubernetes 的基础。

## 一句话结论

Kubernetes 对象通过名称（同类唯一）和 UID（全局唯一）标识，使用标签和选择器灵活分组，通过名字空间实现多租户隔离，借助注解存储非标识元数据，并依赖 Finalizers 机制确保资源安全删除。

## 对象管理方式

### 命令式 vs 声明式

| 方式 | 命令 | 特点 |
|------|------|------|
| 创建 | `kubectl create -f nginx.yaml` | 命令式，创建新对象 |
| 替换 | `kubectl replace -f nginx.yaml` | 命令式，覆盖活动配置 |
| 删除 | `kubectl delete -f nginx.yaml -f redis.yaml` | 命令式，删除指定对象 |
| 应用 | `kubectl apply -f configs/` | 声明式，创建或更新，推荐方式 |
| 预览 | `kubectl diff -f configs/` | 查看变更差异后再 apply |
| 递归 | `kubectl apply -R -f configs/` | 递归处理目录 |

> **最佳实践**：优先使用 `kubectl apply` 声明式管理，配合 `kubectl diff` 预览变更。

## 对象标识

| 标识 | 作用域 | 说明 |
|------|--------|------|
| **名称（Name）** | 同类资源内唯一 | 客户端提供，如 `nginx-deployment` |
| **UID** | 整个集群唯一 | Kubernetes 自动生成，对象生命周期内不变 |

> ⚠️ 注意：当对象代表物理实体（如 Node），若物理主机被删除后重建同名主机，Kubernetes 会将其视为同一对象，可能引发不一致。

## 标签和选择算符

### 标签（Labels）

标签是附加到对象上的**键值对**，用于组织和选择资源。**标签不支持唯一性**。

```json
"metadata": {
  "labels": {
    "key1": "value1",
    "key2": "value2"
  }
}
```

常用操作：

```shell
# 显示指定标签列
kubectl get pods -Lapp -Ltier -Lrole

# 更新标签
kubectl label pods -l app=nginx tier=fe
```

### 标签 vs 注解 vs 字段选择器

| 特性 | 标签 | 注解 | 字段选择器 |
|------|------|------|------------|
| 用途 | 标识和选择对象 | 存储非标识元数据 | 按资源字段筛选 |
| 查询 | 标签选择器 | 不支持选择 | `--field-selector` |
| 值限制 | 键值对字符串 | 任意字符串（可结构化） | 取决于字段类型 |
| 典型场景 | `app=nginx`, `env=prod` | 构建信息、联系人、工具配置 | `status.phase=Running` |

### 字段选择器

支持按资源字段筛选对象：

```shell
# 筛选运行中的 Pod
kubectl get pods --field-selector status.phase=Running

# 链式选择
kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always
```

| 特性 | 说明 |
|------|------|
| 通用字段 | 所有资源支持 `metadata.name` 和 `metadata.namespace` |
| 操作符 | `=`、`==`、`!=`（`=` 和 `==` 等价） |
| 链式 | 逗号分隔，AND 逻辑 |

## 名字空间（Namespace）

### 初始名字空间

| 名字空间 | 用途 | 可见性 |
|----------|------|--------|
| `default` | 默认空间，无需创建即可使用 | 集群内 |
| `kube-node-lease` | 存放节点租约对象，用于心跳检测 | 集群内 |
| `kube-public` | 所有客户端（含未认证）可读 | 全局可读 |
| `kube-system` | Kubernetes 系统创建的对象 | 集群内 |

### 常用操作

```shell
# 查看所有名字空间
kubectl get namespace

# 在指定名字空间中创建资源
kubectl run nginx --image=nginx --namespace=<名称>

# 设置默认名字空间
kubectl config set-context --current --namespace=<名称>

# 查看哪些资源在名字空间中
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

> ⚠️ 并非所有对象都在名字空间中（如 Node、PersistentVolume、Namespace 本身）。

### DNS 与名字空间

> 创建与公共 TLD 同名的名字空间可能导致 DNS 劫持风险：该空间中的服务会获得与公共 DNS 记录重叠的短名称，且优先级高于公共 DNS。应限制名字空间创建权限。

## Finalizers

### 什么是 Finalizer

Finalizer 是带有命名空间的键，告诉 Kubernetes **等到特定条件满足后，再完全删除被标记为删除的资源**。它提醒控制器清理被删除对象拥有的资源。

### Finalizer 工作流程

```
1. 用户发起 DELETE 请求
        ↓
2. API Server 设置 metadata.deletionTimestamp
        ↓
3. 对象被禁止删除，直到 metadata.finalizers 为空
        ↓
4. API Server 返回 202 (Accepted)
        ↓
5. 控制器检测到 deletionTimestamp 被设置
        ↓
6. 控制器逐一满足 Finalizer 条件，删除对应 Finalizer 条目
        ↓
7. finalizers 列表为空 → 对象被自动删除
```

### 关键规则

| 规则 | 说明 |
|------|------|
| 删除后不可添加 | `deletionTimestamp` 设置后，不能添加新 Finalizer |
| 可删除已有 | 可以从 `finalizers` 列表删除已有条目 |
| 不可复活 | 删除请求发出后，对象无法复活，只能删除后重建 |
| `deletionTimestamp` 不可改 | 一旦设置，不可修改 |

## 常见问题

### Q1：标签和注解的使用场景如何区分？

标签用于 Kubernetes 内部的对象选择和分组（如 Service 选择 Pod、Deployment 管理 ReplicaSet），是 Kubernetes 核心选择机制。注解用于存储外部工具或用户需要的元数据（如构建版本号、联系人信息、监控配置），不参与对象选择。简单说：**标签给 Kubernetes 看，注解给人或工具看**。

### Q2：什么时候需要使用 Finalizers？

当你的自定义控制器管理了外部资源（如云负载均衡器、外部数据库），需要在删除 Kubernetes 对象前先清理这些外部资源时，应使用 Finalizers。例如：删除一个 Service 时，需要先删除云提供商的负载均衡器，就可以通过 Finalizer 确保这个清理步骤先完成。

### Q3：`kubectl apply` 和 `kubectl create` 的核心区别是什么？

`kubectl create` 是命令式操作，如果对象已存在会报错。`kubectl apply` 是声明式操作，会自动判断是创建还是更新，并将配置保存到 `last-applied-configuration` 注解中用于三路合并。生产环境推荐使用 `apply`。

## 关联笔记

- [[04 知识资料/Kubernetes/结构化笔记/00 Kubernetes 知识库索引|Kubernetes 知识库索引]]
- [[04 知识资料/Kubernetes/结构化笔记/01 Kubernetes 核心架构与组件|01 核心架构与组件]]
- [[04 知识资料/Kubernetes/结构化笔记/04 Kubernetes 控制器与垃圾收集|04 控制器与垃圾收集]]
