---
type: knowledge
status: evergreen
created: "2026-06-17"
updated: "2026-06-17"
tags:
  - 知识库
  - Kubernetes
  - 部署
  - kubeadm
  - 运维
aliases:
  - Kubernetes 集群部署实战
confidence: high
---

# 06 Kubernetes 集群部署实战

> 基于 kubeadm 部署 Kubernetes v1.36 集群的关键步骤、核心依赖关系和排障指南。适用于学习和实验环境。

## 一句话结论

kubeadm 部署 Kubernetes 集群的核心在于理解四个依赖关系：containerd cgroup 驱动与 kubelet 匹配、Pod 网段与 CNI 插件一致、镜像仓库可达、以及 token 有效期管理。单 master 架构仅适合学习，生产环境需多 master 高可用。

## 部署目标拓扑

| 节点 | IP | 配置 | 角色 |
|------|-----|------|------|
| k8s-master | 10.0.0.21 | 2C4G | control-plane |
| k8s-node1 | 10.0.0.22 | 2C2G | worker |
| k8s-node2 | 10.0.0.23 | 2C2G | worker |

## 部署流程总览

```
系统初始化 → 安装 containerd → 配置 containerd → 安装 kubeadm/kubelet/kubectl
    → kubeadm init → 配置 kubeconfig → worker join → 安装 CNI → 验证集群
```

## 关键步骤

### 1. 系统初始化（所有节点）

| 步骤 | 命令/操作 | 原因 |
|------|-----------|------|
| 关闭防火墙 | `systemctl disable --now firewalld` | 避免网络规则干扰（实验环境） |
| 关闭 SELinux | `setenforce 0` + 修改配置 | 避免权限限制 |
| 关闭 Swap | `swapoff -a` | Kubernetes 强制要求 |
| 加载内核模块 | `overlay`、`br_netfilter` | 容器网络所需 |
| 开启转发 | `net.ipv4.ip_forward=1` | Pod 间通信需要 |
| 桥接流量 | `bridge-nf-call-iptables=1` | 让 iptables 处理桥接流量 |
| 时间同步 | chrony → ntp1.aliyun.com | 证书和令牌依赖准确时间 |

### 2. 安装并配置 containerd

```bash
# 安装 Docker CE（附带 containerd）
yum install -y docker-ce

# 生成默认配置
containerd config default > /etc/containerd/config.toml

# 关键配置修改
sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
sed -i 's#registry.k8s.io#registry.aliyuncs.com/google_containers#g' /etc/containerd/config.toml
```

> ⚠️ **核心依赖 1**：`SystemdCgroup = true` 必须与 kubelet 的 cgroup 驱动一致，否则 kubelet 无法管理容器。

### 3. 安装 Kubernetes 组件

```bash
# 配置阿里云 yum 源
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.36/rpm/
enabled=1
gpgcheck=0
EOF

# 安装指定版本
yum install -y kubectl-1.36.1 kubelet-1.36.1 kubeadm-1.36.1
systemctl enable kubelet
```

| 组件 | 作用 |
|------|------|
| `kubeadm` | 集群初始化与节点加入 |
| `kubelet` | 节点守护进程，管理 Pod |
| `kubectl` | 命令行客户端 |

### 4. 初始化集群（Master 节点）

```bash
kubeadm init \
  --kubernetes-version=v1.36.1 \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=10.0.0.21 \
  --service-cidr=10.224.0.0/16 \
  --image-repository=registry.aliyuncs.com/google_containers
```

| 参数 | 含义 | 注意事项 |
|------|------|----------|
| `--pod-network-cidr` | Pod 网络地址段 | ⚠️ 必须与 CNI 插件配置一致 |
| `--service-cidr` | Service 虚拟 IP 段 | 不能与现有网络冲突 |
| `--apiserver-advertise-address` | API Server 通告地址 | 通常为 master 节点 IP |
| `--image-repository` | 镜像仓库 | 国内环境需指定国内镜像源 |

### 5. 配置 kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 6. Worker 节点加入

```bash
# 在 worker 节点执行（token 来自 init 输出）
kubeadm join 10.0.0.21:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# token 过期时重新生成
kubeadm token create --print-join-command
```

> ⚠️ **核心依赖 2**：`kubeadm join` token 有有效期（默认 24 小时），过期后需重新生成。

### 7. 安装 Calico 网络插件

```bash
# 下载 Calico 清单
wget https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/calico.yaml

# 确保 CALICO_IPV4POOL_CIDR 与 --pod-network-cidr 一致
# 默认值：192.168.0.0/16

kubectl apply -f calico.yaml
```

> ⚠️ **核心依赖 3**：`CALICO_IPV4POOL_CIDR` 必须与 `kubeadm init --pod-network-cidr` 一致，否则 Pod 网络不通。

### 8. 验证集群

```bash
kubectl get node          # 所有节点应为 Ready
kubectl get pods -A       # 所有系统 Pod 应为 Running
kubectl cluster-info      # 查看控制平面和 CoreDNS
```

## 核心依赖关系图

```
containerd (SystemdCgroup=true)  ←→  kubelet (cgroup 驱动一致)
         ↓
kubeadm init (--pod-network-cidr)  ←→  Calico (CALICO_IPV4POOL_CIDR)
         ↓
kubeadm join (token)  ←→  token 有效期 (24h)
         ↓
镜像仓库 (--image-repository)  ←→  网络可达性
```

## 排障检查清单

节点 `NotReady` 时按以下顺序排查：

| 优先级 | 检查项 | 命令 |
|--------|--------|------|
| 1 | containerd 状态 | `systemctl status containerd` |
| 2 | kubelet 状态 | `systemctl status kubelet` |
| 3 | kubelet 日志 | `journalctl -u kubelet -f` |
| 4 | CNI 是否安装 | `kubectl get pods -A \| grep calico` |
| 5 | Pod 网段一致性 | 对比 init 参数和 Calico 配置 |
| 6 | Swap 是否关闭 | `free -h` / `swapon --show` |
| 7 | 内核转发参数 | `sysctl net.ipv4.ip_forward` |
| 8 | 时间同步 | `chronyc sources` |
| 9 | 镜像拉取 | `crictl pull <image>` |

### 初始化失败重置

```bash
kubeadm reset -f
rm -fr ~/.kube/ /etc/kubernetes/* /var/lib/etcd/*
```

> ⚠️ 破坏性操作，会清理集群配置和 etcd 数据。

## 生产环境注意事项

实验环境（1 master + 2 node）的不足与改进：

| 不足 | 生产改进 |
|------|----------|
| 单 master | 多 master 高可用 + API Server 前置负载均衡 |
| 单 etcd | etcd 集群（3/5 节点）+ 定期备份 |
| 公网镜像 | 内网镜像仓库（Harbor） |
| 无监控 | Prometheus + Grafana + AlertManager |
| 无日志 | EFK/Loki 日志系统 |
| 无安全策略 | NetworkPolicy + RBAC + PodSecurity |

## 常见问题

### Q1：kubeadm init 失败后如何重试？

```bash
kubeadm reset -f                    # 清理集群配置
rm -fr ~/.kube/ /etc/kubernetes/*   # 清理 kubeconfig 和配置
rm -fr /var/lib/etcd/*              # 清理 etcd 数据
# 修复问题后重新 kubeadm init
```

### Q2：节点加入后一直 NotReady 怎么办？

90% 的情况是 CNI 网络插件未安装或配置不正确。先检查 `kubectl get pods -n kube-system | grep calico`（或其他 CNI），确保所有 CNI Pod 都是 Running。然后检查 `--pod-network-cidr` 与 CNI 配置是否一致。

### Q3：containerd 的 cgroup 驱动为什么要和 kubelet 一致？

kubelet 通过 CRI 接口与 containerd 交互来管理容器的资源限制。如果两者使用不同的 cgroup 驱动（一个用 systemd，一个用 cgroupfs），会导致资源统计不一致，甚至容器无法正常启动。Kubernetes 官方推荐使用 systemd 作为 cgroup 驱动。

## 关联笔记

- [[04 知识资料/Kubernetes/结构化笔记/00 Kubernetes 知识库索引|Kubernetes 知识库索引]]
- [[04 知识资料/Kubernetes/结构化笔记/01 Kubernetes 核心架构与组件|01 核心架构与组件]]
- [[04 知识资料/Kubernetes/结构化笔记/03 Kubernetes 节点管理|03 节点管理]]
