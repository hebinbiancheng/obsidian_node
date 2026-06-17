---
type: knowledge
status: evergreen
created: "2026-06-17"
updated: "2026-06-17"
tags:
  - 知识库
  - Kubernetes
  - 面试
  - 运维
aliases:
  - Kubernetes 面试问答精粹
confidence: high
---

# 05 Kubernetes 面试问答精粹

> 10 个 Kubernetes 高频面试问题，涵盖调度、OOM、配置更新、Pod 稳定性、Service 负载均衡、日志采集、健康检查、弹性伸缩、kubectl exec 原理和故障排查。

## 一句话结论

Kubernetes 面试不仅考察概念记忆，更注重对调度机制、网络模型、存储和故障排查的深入理解。以下 10 个问题覆盖了日常运维中最容易踩坑的场景。

---

## Q1：集群有 2 个 node，一个有 pod，另一个没有，新 pod 调度到哪？

**核心要点**：取决于调度器的评分策略，不是简单的"调度到空闲节点"。

调度过程经过预选（过滤）→ 优选（打分）→ 绑定三个阶段。在节点资源相同、无特殊配置的情况下，`NodeResourcesFit` 插件的评分策略起关键作用：

| 策略 | 行为 | 新 Pod 去向 |
|------|------|-------------|
| `LeastAllocated`（默认） | 优先资源使用率最低的节点 | 没有 Pod 的节点 |
| `MostAllocated` | 优先资源使用率较高的节点 | 已有 Pod 的节点 |
| `RequestedToCapacityRatio` | 平衡资源使用率 | 取决于配置 |

> 此外，Pod 的亲和性、反亲和性、污点容忍、节点选择器等配置也会影响调度结果。

---

## Q2：容器 OOM 了，是容器重启还是 Pod 重建？

**核心要点**：默认是容器重启，极端情况下 Pod 被驱逐。

| 场景 | 结果 |
|------|------|
| 容器 OOM（一般情况） | 根据 `RestartPolicy`（默认 Always），**容器重启**，Pod 不重建 |
| 节点内存压力极大 | 可能触发 **Pod 驱逐**，Pod 被重建到其他节点 |

> `RestartPolicy` 可选值：`Always`（默认）、`OnFailure`、`Never`。

---

## Q3：ConfigMap 和环境变量可以不重建 Pod 动态更新吗？

**核心要点**：环境变量不行，ConfigMap 挂载可以但有条件。

| 方式 | 能否动态更新 | 条件/限制 |
|------|-------------|-----------|
| 环境变量（env） | ❌ 不能 | Pod 启动时注入，运行期不变 |
| ConfigMap 挂载（volume） | ✅ 可以 | 不能使用 `subPath` 挂载 |
| 挂载同步延迟 | — | 受 kubelet `syncFrequency`（默认 1 分钟）和 `configMapAndSecretChangeDetectionStrategy` 影响 |

---

## Q4：Pod 创建后是稳定的吗（用户不操作）？

**核心要点**：不稳定，可能因节点问题被驱逐。

即使用户不进行任何操作，以下情况也可能导致 Pod 被驱逐：
- 节点资源不足（内存/磁盘压力）
- 节点网络异常
- 节点被标记为不可调度
- 污点和容忍策略触发

---

## Q5：ClusterIP Service 能保证 TCP 负载均衡吗？

**核心要点**：不能完全保证，长连接场景下可能不均衡。

`ClusterIP` Service 无论使用 `iptables` 还是 `ipvs` 模式，都依赖 Linux 内核 Netfilter 的 `connection tracking` 机制。该机制会跟踪已建立的 TCP 连接状态：

- **短连接**：通常能较好均衡
- **长连接**：已建立的连接会一直使用同一后端 Pod，可能出现负载不均

> 解决方案：使用 Service 拓扑、Headless Service + 客户端负载均衡、或 Service Mesh。

---

## Q6：应用日志怎么采集，会不会丢失？

**核心要点**：两种方式各有优劣，stdout 方式可能丢失。

| 采集方式 | 机制 | 丢失风险 |
|----------|------|----------|
| stdout/stderr | 容器日志写入节点文件，日志代理（Fluentd/Filebeat DaemonSet）采集 | ⚠️ 可能丢失：Pod 删除时日志文件被清理，代理可能未采集完 |
| 日志文件 + 持久卷 | 日志写入挂载的持久化存储，代理采集 | ✅ 基本不丢失 |

> 生产环境建议：关键日志使用持久卷 + Sidecar 采集模式。

---

## Q7：HTTP Server Pod 的 livenessProbe 正常就万事大吉吗？

**核心要点**：不够，livenessProbe 只检查"活着"，不检查"正常"。

| 维度 | livenessProbe 的局限 |
|------|----------------------|
| 应用层面 | 只检查进程是否响应，无法验证业务逻辑是否正常（如死锁但端口仍监听） |
| 网络层面 | 由本机 kubelet 发起请求，无法验证跨节点网络是否正常 |

> 建议：配合 `readinessProbe` 检查服务就绪状态，配合 `startupProbe` 处理慢启动应用。

---

## Q8：应用程序如何扩展应对流量波动？

**核心要点**：HPA 是主力，VPA 受限。

| 扩展方式 | 机制 | 适用场景 |
|----------|------|----------|
| **HPA**（水平扩展） | 根据指标动态调整 Pod 副本数 | ✅ 主流方案，支持 CPU/内存/自定义指标 |
| **VPA**（垂直扩展） | 调整 Pod 资源请求/限制 | ⚠️ 非原地扩展，需删除 Pod 重建，场景受限 |
| 外部触发 | 外部系统监测指标后调用 API 扩缩 | 自定义策略 |

> HPA 常用指标：CPU 使用率、请求速率（QPS）、自定义 Prometheus 指标。

---

## Q9：`kubectl exec -it <pod> -- bash` 是登录到 Pod 里吗？

**核心要点**："登录"的说法不准确，本质是在目标容器的隔离环境中创建新进程。

| 概念 | 说明 |
|------|------|
| Pod | 一组共享 Network、IPC、UTS namespace 的容器 |
| 容器 | 本质上是一个进程，PID 和 Mount namespace 独立 |
| `kubectl exec` | 在目标**容器**的隔离环境中创建新的 bash 进程 |

> 正确理解：不是"进入 Pod"或"登录容器"，而是在目标容器的 namespace 中启动了一个新的 shell 进程。当 Pod 只有一个容器时可省略 `-c` 参数。

---

## Q10：Pod 里容器反复退出重启，如何排查？

**核心要点**：无法 `kubectl exec` 时，用 `kubectl debug`。

| 排查手段 | 说明 |
|----------|------|
| `kubectl describe pod` | 查看事件和退出原因 |
| `kubectl logs <pod> --previous` | 查看上一次容器的日志 |
| 节点日志 | `journalctl -u kubelet` 查看节点层面信息 |
| `kubectl debug` | 在 Pod 上启动临时容器（Ephemeral Container），检查环境和依赖 |

> `kubectl debug` 是排查反复重启容器的利器：它创建一个临时容器共享 Pod 的 namespace，可以检查文件系统、网络、依赖等。

---

## 关联笔记

- [[04 知识资料/Kubernetes/结构化笔记/00 Kubernetes 知识库索引|Kubernetes 知识库索引]]
- [[04 知识资料/Kubernetes/结构化笔记/01 Kubernetes 核心架构与组件|01 核心架构与组件]]
- [[04 知识资料/Kubernetes/结构化笔记/02 Kubernetes 对象与资源管理|02 对象与资源管理]]
- [[04 知识资料/Kubernetes/结构化笔记/03 Kubernetes 节点管理|03 节点管理]]
