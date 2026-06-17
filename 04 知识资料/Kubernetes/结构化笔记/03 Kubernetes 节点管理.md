---
type: knowledge
status: evergreen
created: "2026-06-17"
updated: "2026-06-17"
tags:
  - 知识库
  - Kubernetes
  - 节点管理
  - 运维
aliases:
  - Kubernetes 节点管理
confidence: high
---

# 03 Kubernetes 节点管理

> 节点是 Kubernetes 集群的工作单元。理解节点的注册、生命周期、健康监控和资源管理，是运维 Kubernetes 集群的核心技能。

## 一句话结论

Kubernetes 节点通过 kubelet 自注册或手动添加两种方式加入集群，节点控制器负责健康监控和故障响应，逐出速率机制防止大规模 Pod 驱逐雪崩，资源容量跟踪确保调度器做出正确的调度决策。

## 节点添加方式

| 方式 | 机制 | 适用场景 |
|------|------|----------|
| **kubelet 自注册** | kubelet 启动时自动向 API Server 注册（`--register-node=true`，默认） | 标准方式，推荐 |
| **手动添加** | 管理员手动创建 Node 对象 | 特殊场景，如模拟节点 |

> ⚠️ Kubernetes 会一直保留非法节点对象并持续健康检查，必须**显式删除** Node 对象才能停止。

## 节点自注册参数

当 kubelet 以自注册模式启动时，关键参数：

| 参数 | 作用 |
|------|------|
| `--kubeconfig` | API Server 认证凭据路径 |
| `--cloud-provider` | 云驱动通信方式（读取节点元数据） |
| `--register-node` | 是否自动注册（默认 true） |
| `--register-with-taints` | 注册时添加污点，格式 `<key>=<value>:<effect>` |
| `--node-ip` | 节点 IP 地址（逗号分隔），不提供则自动检测 |
| `--node-labels` | 注册时添加的标签 |
| `--node-status-update-frequency` | 向 API Server 发送节点状态的频率 |

### 节点名称唯一性

节点名称标识 Node 对象，**同一集群中不能有两个节点使用相同名称**。

## 节点控制器

节点控制器在节点生命周期中扮演多个角色：

| 角色 | 说明 |
|------|------|
| CIDR 分配 | 节点注册时分配 CIDR 区段（需开启 CIDR 分配） |
| 节点列表同步 | 保持控制器内节点列表与云服务商可用机器列表同步 |
| 健康监控 | 默认每 **5 秒**检查一次节点状态 |

## 逐出速率限制

节点控制器通过逐出速率限制防止大规模 Pod 驱逐雪崩：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `--node-eviction-rate` | 0.1/秒 | 正常状态下，每 10 秒最多驱逐 1 个 Pod |
| `--unhealthy-zone-threshold` | 0.55 | 不健康节点比例超过此阈值时降低驱逐速率 |
| `--large-cluster-size-threshold` | 50 | 集群节点数阈值 |
| `--secondary-node-eviction-rate` | 0.01/秒 | 降级后的驱逐速率 |

### 逐出速率变化逻辑

```
不健康节点比例 > 55%？
  ├── 是 → 集群 ≤ 50 节点？
  │        ├── 是 → 停止驱逐
  │        └── 否 → 降为 0.01/秒
  └── 否 → 保持 0.1/秒
```

## 资源容量跟踪

Node 对象跟踪节点上的资源容量（CPU、内存等）。调度器保证：

> 节点上所有容器的**资源请求之和**不超过节点的**总容量**

这是调度决策的核心约束，确保不会将 Pod 调度到资源不足的节点。

## 节点与控制平面通信

### 通信方向

| 方向 | 路径 | 安全性 |
|------|------|--------|
| 节点 → 控制平面 | 所有 API 调用终止于 API Server（HTTPS 443） | 需客户端认证 |
| 控制平面 → kubelet | API Server → kubelet HTTPS 端点 | 默认不验证证书 ⚠️ |
| 控制平面 → 节点/Pod/Service | API Server 代理功能 | 默认纯 HTTP ⚠️ |

### 安全加固方案

| 方案 | 机制 | 适用场景 |
|------|------|----------|
| SSH 隧道 | API Server 建立到各节点的 SSH 隧道 | 传统方案 |
| Konnectivity 服务 | TCP 代理层，安全转发控制平面到集群的流量 | 推荐方案 |

## 常见问题

### Q1：节点 NotReady 的常见原因有哪些？

按优先级排查：
1. **kubelet 状态**：`systemctl status kubelet`，查看是否正常运行
2. **容器运行时**：containerd/docker 是否正常
3. **CNI 网络插件**：是否安装且正常运行
4. **节点资源**：磁盘空间、内存是否充足
5. **网络连通性**：节点到 API Server 的网络是否可达
6. **时间同步**：节点时间是否与集群一致

### Q2：节点被标记为 NotReady 后，上面的 Pod 会怎样？

默认情况下，节点 NotReady 后：
- **40 秒**后，节点被标记为 `ConditionUnknown`
- **5 分钟**后（`pod-eviction-timeout`），该节点上的 Pod 开始被驱逐到其他健康节点
- 如果有 Pod 使用了 `toleration` 容忍该污点，则不会被驱逐

### Q3：如何安全地从集群中移除一个节点？

```shell
# 1. 驱逐节点上的 Pod
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 2. 删除节点对象
kubectl delete node <node-name>

# 3. 在节点上清理
kubeadm reset -f
```

## 关联笔记

- [[04 知识资料/Kubernetes/结构化笔记/00 Kubernetes 知识库索引|Kubernetes 知识库索引]]
- [[04 知识资料/Kubernetes/结构化笔记/01 Kubernetes 核心架构与组件|01 核心架构与组件]]
- [[04 知识资料/Kubernetes/结构化笔记/04 Kubernetes 控制器与垃圾收集|04 控制器与垃圾收集]]
- [[04 知识资料/Kubernetes/结构化笔记/06 Kubernetes 集群部署实战|06 集群部署实战]]
