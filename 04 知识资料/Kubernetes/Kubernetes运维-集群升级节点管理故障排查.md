---
title: Kubernetes 运维：集群升级、节点管理与故障排查
date: 2026-07-05
tags: [Kubernetes, 运维, 集群升级, 节点管理, 故障排查]
category: 知识资料
source: https://mp.weixin.qq.com/s/QA1h8jY6l1uVdoFylo-jug
---

# Kubernetes 运维：集群升级、节点管理与故障排查

> 原文: [微信文章](https://mp.weixin.qq.com/s/QA1h8jY6l1uVdoFylo-jug)

集群跑着跑着，老板说："K8s 版本太低了，升级一下。"你心里一紧：生产集群说升就升？跑着几十个业务 Pod，升级过程中业务中断怎么办？回滚怎么操作？新版本兼容性问题谁负责？

别慌，今天把你关心的三个运维核心场景全讲透：**集群平滑升级**、**节点日常管理**、**常见故障排查**。

---

## 一、集群升级：控制平面先行，工作节点滚动

### 1.1 升级原则：控制平面先升，节点按批次升

Kubernetes 升级遵循"**先控制平面，后工作节点**"原则。控制平面（etcd、API Server、Controller Manager、Scheduler）向前兼容，支持运行比它版本旧的节点。这意味着：先升级控制平面，再升级节点，过程中业务 Pod 仍然可以通过旧版本节点调度，保持可用。

版本跨度建议**一次只升一个次版本**（如 1.26 → 1.27 → 1.28），跨越大版本升级风险极高。

### 1.2 升级前必做检查清单

```bash
# 查看当前集群版本
kubectl version --short

# 查看所有节点状态，确认没有异常节点
kubectl get nodes

# 查看所有 Pod 状态，确认无 CrashLoopBackOff 或 Pending
kubectl get pods --all-namespaces | grep -v Running

# 确认 etcd 数据目录有足够空间（升级前必须备份！）
df -h /var/lib/etcd

# 检查 API Server 日志有无 Error 或 Warning
kubectl logs -n kube-system -l component=kube-apiserver --tail=50
```

### 1.3 用 kubeadm 升级控制平面

```bash
# 查看可升级版本
kubeadm upgrade plan

# 升级控制平面（原地升级，不破坏 etcd 数据）
sudo kubeadm upgrade apply v1.28.0

# 手动升级 kubeadm 二进制后，升级 etcd（若使用外部 etcd）
kubectl exec -n kube-system etcd-<node> -- etcdctl version
```

### 1.4 升级工作节点：drain → 升级 → uncordon

**第一步：驱逐节点上的 Pod**（cordon + drain）：

```bash
# 安全驱逐：允许 Pod 优雅终止，保留副本数
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --force

# --ignore-daemonsets：忽略 DaemonSet Pod（无法被驱逐）
# --delete-emptydir-data：清理 emptyDir 卷数据
# --force：强制删除无法优雅终止的 Pod（如 standalone 态）
```

**第二步：升级 kubelet 和 kubectl**：

```bash
# Debian/Ubuntu
sudo apt-get update && sudo apt-get install -y kubelet=1.28.0-1.1 kubectl=1.28.0-1.1 kubeadm=1.28.0-1.1

# CentOS/RHEL
sudo yum install -y kubelet-1.28.0 kubectl-1.28.0 kubeadm-1.28.0
```

**第三步：重启 kubelet 并恢复节点**：

```bash
sudo systemctl restart kubelet

# 将节点重新加入调度
kubectl uncordon <node-name>

# 确认节点状态变为 Ready
kubectl get nodes <node-name>
```

**第四步：验证升级结果**：

```bash
# 确认节点版本应显示为新版本
kubectl get nodes
```

### 1.5 升级失败回滚

如果升级后控制平面故障，先检查 etcd 是否正常：

```bash
kubectl get etcd -n kube-system -o wide
kubectl logs -n kube-system -l app=etcd --tail=20
```

大多数控制平面问题重启 kubelet 组件容器即可恢复，回滚到旧版本的代价远高于此。

---

## 二、节点管理：日常运维三板斧

### 2.1 cordon / drain / uncordon 三个命令的区别

| 命令 | 效果 | 适用场景 |
|------|------|----------|
| `kubectl cordon` | 禁止新 Pod 调度到该节点，现有 Pod 不受影响 | 计划内维护前，临时隔离 |
| `kubectl drain` | 驱逐节点上所有 Pod（除 DaemonSet），等效于 cordon + 驱逐 | 节点下线/升级 |
| `kubectl uncordon` | 恢复节点调度 | 维护完成后 |

### 2.2 节点打标签与污点管理

给节点打标签，用于调度优化：

```bash
# 给节点打标签，声明硬件类型
kubectl label node <node-name> node-type=memory-intensive
kubectl label node <node-name> gpu=true

# 查看节点标签
kubectl get nodes --show-labels
```

污点（Taints）控制 Pod **不能**调度到某节点，除非 Pod 有对应 Toleration：

```bash
# 给节点打污点：禁止普通 Pod 调度
kubectl taint node <node-name> dedicated=infra:NoSchedule
```

```yaml
# Pod 必须声明对应 toleration 才能调度上去
apiVersion: v1
kind: Pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "infra"
    effect: "NoSchedule"
```

### 2.3 节点日常巡检

```bash
# 节点资源使用情况
kubectl top nodes

# 节点事件（最近 10 条异常事件）
kubectl get events --all-namespaces --field-selector involvedObject.kind=Node \
  --sort-by='.lastTimestamp' | tail -10

# 节点 Ready 状态异常检测
kubectl get nodes | grep -v Ready

# 查看节点 Conditions（重点看 MemoryPressure / DiskPressure / PIDPressure）
kubectl describe node <node-name> | grep -A 10 Conditions
```

---

## 三、故障排查：一看状态，二看日志，三看事件

### 3.1 Pod 状态速查表

`kubectl get pods` 查看状态，是排查的第一步：

| 状态 | 含义 | 常见原因 |
|------|------|----------|
| **Pending** | 调度失败 | 资源不足、标签选择器无匹配节点、污点限制 |
| **CrashLoopBackOff** | 启动后崩溃，反复重启 | 应用报错、健康检查失败、配置文件错误 |
| **ImagePullBackOff** | 镜像拉取失败 | 镜像名错误、私仓认证失败、网络不通 |
| **ErrImagePull** | 镜像拉取错误 | 同上，更早期的错误 |
| **Terminating** | 终止卡住 | finalizers 阻塞、API Server 连不上、磁盘满 |
| **Running 但 Not Ready** | 健康检查未通过 | 应用未就绪、启动慢、探测阈值过严 |

### 3.2 排查第一步：describe

Pod 有问题，先 describe 看详情：

```bash
kubectl describe pod <pod-name> -n <namespace>
```

重点关注：
- **Events 区域**：最下方会显示最近事件，告诉你调度失败/镜像拉取/启动失败的原因
- **Conditions**：PodScheduled、Initialized、ContainersReady、Ready 的状态
- **Node**：Pod 被调度到了哪个节点

### 3.3 排查第二步：看日志

```bash
# 查看当前日志（Pod 未崩溃时）
kubectl logs <pod-name> -n <namespace>

# 实时跟踪日志
kubectl logs -f <pod-name> -n <namespace>

# 查看上次运行的日志（Pod 崩溃后）
kubectl logs --previous <pod-name> -n <namespace>

# 查看特定容器日志（多容器 Pod）
kubectl logs <pod-name> -n <namespace> -c <container-name>

# 查看 init 容器日志（init 容器失败最容易被忽略！）
kubectl logs <pod-name> -n <namespace> -c <init-container-name> --previous
```

### 3.4 实战案例：Pod 一直 Pending

```bash
kubectl get pods -n app-prod
# NAME        READY   STATUS    RESTARTS   AGE
# myapp-0     0/1     Pending   0          5m
```

describe 看一下：

```bash
kubectl describe pod myapp-0 -n app-prod | tail -20
```

可能看到的事件：

```
0/3 nodes are available: 1 node(s) had taint {node.kubernetes.io/not-ready}, 
that the pod didn't tolerate, 2 node(s) didn't have free resource.
```

**解法：**
- `kubectl taint nodes <node> node.kubernetes.io/not-ready:NoExecute-` 临时移除污点（仅应急）
- 或者扩缩节点解决资源不足
- 检查 Pod 的 `resource.requests` 是否设置过高

### 3.5 实战案例：Pod 一直 CrashLoopBackOff

```bash
kubectl get pods -n app-prod
# myapp-6d8f9b4-xkpq2   0/1   CrashLoopBackOff   3     2m

# 查看上次崩溃的日志
kubectl logs --previous myapp-6d8f9b4-xkpq2 -n app-prod
```

看到输出：

```
Error: failed to connect to database: connection refused
```

**解法：**
1. 确认数据库是否正常运行
2. 检查 Pod 启动顺序（应用 Pod 依赖数据库，数据库 Pod 还未就绪）
3. 添加 init 容器或 readinessProbe 等待依赖服务就绪

### 3.6 排查第三步：kubectl exec 进容器诊断

当日志不够时，直接进容器看：

```bash
# 进入容器（如有多个容器，用 -c 指定）
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# 查看网络连接（确认 DNS 和网络插件正常）
kubectl exec -it <pod-name> -n <namespace> -- nslookup kubernetes.default
kubectl exec -it <pod-name> -n <namespace> -- cat /etc/resolv.conf

# 查看进程和资源占用
kubectl exec -it <pod-name> -n <namespace> -- top

# 查看挂载的配置文件
kubectl exec -it <pod-name> -n <namespace> -- cat /app/config.yaml
```

### 3.7 事件日志是排查的金钥匙

Kubernetes 事件（Events）记录了所有 API 操作，出问题第一时间看：

```bash
# 查看 namespace 下所有异常事件（最近 30 分钟）
kubectl get events -n <namespace> --sort-by='.lastTimestamp' \
  | grep -v Normal | tail -30

# 查看某个 Pod 相关的所有事件
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name>

# 集群级别事件（节点相关）
kubectl get events --all-namespaces \
  --field-selector involvedObject.kind=Node
```

---

## 总结：运维三板斧，流程要记牢

**集群升级核心口诀：** 控制平面先升，节点分批 drain uncordon，失败先查 etcd 再重启组件。

**节点管理核心口诀：** cordon 临时隔离，drain 正式维护，uncordon 恢复调度，污点和标签配合用。

**故障排查核心口诀：** 一看 get pods 状态，二看 describe 事件，三看 logs 日志，四看 exec 进容器实地诊断。

> 收藏了不等于会了，建议搭个测试集群走一遍 drain/uncordon 这套流程。
