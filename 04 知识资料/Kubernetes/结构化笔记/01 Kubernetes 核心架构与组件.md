---
type: knowledge
status: evergreen
created: "2026-06-17"
updated: "2026-06-17"
tags:
  - 知识库
  - Kubernetes
  - 架构
  - 容器
  - 运维
aliases:
  - Kubernetes 核心架构与组件
confidence: high
---

# 01 Kubernetes 核心架构与组件

> Kubernetes 集群由一个控制平面和一组工作节点组成，各组件协同完成容器化应用的编排与管理。

## 一句话结论

Kubernetes 采用中心辐射型架构：控制平面（API Server、etcd、Scheduler、Controller Manager）负责决策与状态管理，工作节点（kubelet、kube-proxy、容器运行时）负责实际运行负载，所有通信以 API Server 为枢纽。

## 架构总览

```text
                    ┌─────────────────────────────────┐
                    │        Control Plane            │
                    │  ┌───────────┐ ┌─────────────┐  │
                    │  │ API Server│ │    etcd      │  │
                    │  └─────┬─────┘ └─────────────┘  │
                    │  ┌─────┴─────┐ ┌─────────────┐  │
                    │  │ Scheduler │ │Controller Mgr│  │
                    │  └───────────┘ └─────────────┘  │
                    └──────────────┬──────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
     ┌────────┴────────┐  ┌───────┴────────┐  ┌───────┴────────┐
     │    Node 1       │  │    Node 2      │  │    Node N      │
     │  ┌───────────┐  │  │  ┌───────────┐ │  │  ┌───────────┐ │
     │  │  kubelet  │  │  │  │  kubelet  │ │  │  │  kubelet  │ │
     │  ├───────────┤  │  │  ├───────────┤ │  │  ├───────────┤ │
     │  │kube-proxy │  │  │  │kube-proxy │ │  │  │kube-proxy │ │
     │  ├───────────┤  │  │  ├───────────┤ │  │  ├───────────┤ │
     │  │Container  │  │  │  │Container  │ │  │  │Container  │ │
     │  │ Runtime   │  │  │  │ Runtime   │ │  │  │ Runtime   │ │
     │  └───────────┘  │  │  └───────────┘ │  │  └───────────┘ │
     └─────────────────┘  └────────────────┘  └────────────────┘
```

## 控制平面组件

| 组件 | 角色 | 关键职责 |
|------|------|----------|
| **kube-apiserver** | 集群统一入口 | 公开 Kubernetes API，处理所有请求，是控制平面的前端 |
| **etcd** | 分布式 KV 存储 | 持久化集群所有配置和状态数据 |
| **kube-scheduler** | Pod 调度器 | 监视未绑定节点的 Pod，为其选择最优节点运行 |
| **kube-controller-manager** | 控制器集合 | 运行内置控制器（Node、Job、EndpointSlice、ServiceAccount 等） |
| **cloud-controller-manager** | 云平台桥接 | 嵌入云平台特定逻辑，连接集群与云提供商 API |

### kube-controller-manager 内置控制器

| 控制器 | 职责 |
|--------|------|
| Node 控制器 | 节点故障时通知和响应 |
| Job 控制器 | 监测 Job 对象，创建 Pod 执行任务 |
| EndpointSlice 控制器 | 填充 EndpointSlice，连接 Service 与 Pod |
| ServiceAccount 控制器 | 为新命名空间创建默认 ServiceAccount |

## 节点组件

| 组件 | 角色 | 关键职责 |
|------|------|----------|
| **kubelet** | 节点守护进程 | 确保容器在 Pod 中运行，是每个节点上的核心代理 |
| **kube-proxy** | 网络代理（可选） | 实现 Service 的网络规则（iptables/IPVS） |
| **容器运行时** | 容器执行环境 | 管理容器的执行和生命周期（containerd、CRI-O 等） |

## 插件（Addons）

插件使用 Kubernetes 资源（DaemonSet、Deployment 等）实现集群级别功能，资源属于 `kube-system` 命名空间。

| 插件类型 | 常见实现 |
|----------|----------|
| DNS | CoreDNS |
| Web 界面 | Kubernetes Dashboard |
| 容器资源监控 | Metrics Server、Prometheus |
| 集群日志 | Fluentd、Filebeat（DaemonSet） |
| 网络插件（CNI） | Calico、Flannel、Cilium |

## 通信模式

Kubernetes 采用**中心辐射型** API 模式：

- **节点 → 控制平面**：所有从节点发出的 API 调用终止于 API Server（HTTPS 443 端口），需客户端认证
- **控制平面 → 节点**：两条主要路径：
  - API Server → kubelet（HTTPS，默认不验证 kubelet 证书，存在中间人攻击风险）
  - API Server → 节点/Pod/Service（默认纯 HTTP，无认证无加密）

### 安全通信方案

| 方案 | 说明 |
|------|------|
| SSH 隧道 | API Server 建立到各节点的 SSH 隧道传输请求 |
| Konnectivity 服务 | TCP 代理，替代 SSH 隧道，支持控制平面到集群的安全通信 |

## 常见问题

### Q1：kubelet 和 kube-proxy 有什么区别？

kubelet 是节点上的核心守护进程，负责管理 Pod 的生命周期（创建、监控、销毁容器），与容器运行时交互。kube-proxy 是网络代理，负责维护节点上的网络规则（iptables/IPVS），实现 Service 的流量转发和负载均衡。两者职责不同：kubelet 管 Pod，kube-proxy 管网络。

### Q2：为什么 API Server 到 kubelet 的连接默认不安全？

默认情况下 API Server 不验证 kubelet 的服务证书，连接容易受到中间人攻击。这是因为历史设计原因。生产环境应配置 TLS 引导（TLS bootstrapping）或使用 Konnectivity 服务来保护这条通信路径。

### Q3：cloud-controller-manager 在本地部署时需要吗？

不需要。cloud-controller-manager 仅在云环境（AWS、GCP、Azure 等）中才有意义，用于与云提供商的 API 交互（如创建负载均衡器、管理存储卷）。本地裸金属或虚拟机部署不需要此组件。

## 关联笔记

- [[04 知识资料/Kubernetes/结构化笔记/00 Kubernetes 知识库索引|Kubernetes 知识库索引]]
- [[04 知识资料/Kubernetes/结构化笔记/02 Kubernetes 对象与资源管理|02 对象与资源管理]]
- [[04 知识资料/Kubernetes/结构化笔记/03 Kubernetes 节点管理|03 节点管理]]
- [[04 知识资料/Kubernetes/结构化笔记/04 Kubernetes 控制器与垃圾收集|04 控制器与垃圾收集]]
