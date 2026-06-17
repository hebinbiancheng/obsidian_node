---
type: knowledge
status: evergreen
created: "2026-06-12"
updated: "2026-06-12"
tags:
  - 知识库
  - Kubernetes
  - 容器
  - 运维
aliases:
  - Kubernetes 1.36 单 Master 双 Node 集群部署手册
---
---
title: Kubernetes 1.36 单 Master 双 Node 集群部署手册
source: https://mp.weixin.qq.com/s/XbnFi3A2t1CE91fwY-ZtLQ
author: 李永斌
source_type: 微信文章
created: 2026-06-11
published: 2026-06-11
tags:
  - Kubernetes
  - K8S
  - RockyLinux
  - kubeadm
  - containerd
  - Calico
  - 运维
status: 已整理
---

# Kubernetes 1.36 单 Master 双 Node 集群部署手册

> 原文：[领导让部署一套Kubernetes最新集群，我咔咔咔给搞定（1.36无坑版）](https://mp.weixin.qq.com/s/XbnFi3A2t1CE91fwY-ZtLQ)  
> 作者：李永斌  
> 整理时间：2026-06-11

## 一句话总结

这篇文章是一份基于 `kubeadm` 部署 Kubernetes `v1.36.1` 的实操教程，目标环境是 **1 个 master + 2 个 worker node**，系统为 Rocky Linux 9.5，容器运行时使用 containerd，网络插件使用 Calico。整体适合学习、测试和实验环境；生产环境应改为多 master 高可用架构。

## 部署目标

| 节点 | IP | 操作系统 | 配置 | 角色 |
|---|---|---|---|---|
| k8s-master | `10.0.0.21` | Rocky Linux 9.5 | 2C4G | control-plane |
| k8s-node1 | `10.0.0.22` | Rocky Linux 9.5 | 2C2G | worker |
| k8s-node2 | `10.0.0.23` | Rocky Linux 9.5 | 2C2G | worker |

## 总体流程

1. 系统初始化配置；
2. 安装并配置 containerd；
3. 安装 `kubeadm`、`kubelet`、`kubectl`；
4. 在 master 节点执行 `kubeadm init`；
5. 配置 kubeconfig；
6. worker 节点加入集群；
7. 安装 Calico 网络插件；
8. 给 worker 节点打标签；
9. 验证集群状态。

## 1. 系统初始化配置

以下操作通常需要在所有节点执行。

### 1.1 关闭防火墙和 SELinux

```bash
systemctl disable --now firewalld
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
setenforce 0
```

> 注意：生产环境不建议简单粗暴关闭安全能力，应结合企业安全策略做规则放行和基线加固。实验环境可以先关闭以减少部署干扰。

### 1.2 修改主机名

```bash
# master
hostnamectl set-hostname k8s-master

# node1
hostnamectl set-hostname k8s-node1

# node2
hostnamectl set-hostname k8s-node2
```

### 1.3 配置 hosts

```bash
cat >> /etc/hosts <<EOF
10.0.0.21 k8s-master
10.0.0.22 k8s-node1
10.0.0.23 k8s-node2
EOF
```

### 1.4 时间同步

```bash
yum install -y chrony
sed -ri 's/^pool.*/#&/' /etc/chrony.conf
cat >> /etc/chrony.conf << EOF
pool ntp1.aliyun.com iburst
EOF
systemctl restart chronyd
chronyc sources

ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'Asia/Shanghai' > /etc/timezone
```

### 1.5 关闭 Swap

Kubernetes 默认要求关闭 swap。

```bash
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

### 1.6 内核模块与转发参数

```bash
cat << EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat >> /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl -p /etc/sysctl.d/kubernetes.conf
sysctl --system
lsmod | grep br_netfilter
```

关键点：

- `br_netfilter`：让桥接流量进入 iptables / ip6tables；
- `net.ipv4.ip_forward=1`：允许 IPv4 转发；
- `overlay`：容器网络常用内核模块。

### 1.7 配置免密登录（可选）

用于批量管理节点。

```bash
ssh-keygen
ssh-copy-id -i ~/.ssh/id_rsa.pub root@10.0.0.21
ssh-copy-id -i ~/.ssh/id_rsa.pub root@10.0.0.22
ssh-copy-id -i ~/.ssh/id_rsa.pub root@10.0.0.23
```

### 1.8 安装 IPVS 转发支持

Kubernetes Service 代理模式可使用 iptables 或 IPVS。文章建议加载 IPVS 相关模块。

```bash
yum install -y conntrack ipvsadm libseccomp
mkdir /etc/sysconfig/modules/

cat > /etc/sysconfig/modules/ipvs.modules << EOF
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF

chmod +x /etc/sysconfig/modules/ipvs.modules
/etc/sysconfig/modules/ipvs.modules
lsmod | grep -e ip_vs -e nf_conntrack
```

## 2. 安装 Docker 与 containerd

文章中先安装 Docker CE，同时利用其附带安装的 containerd。Kubernetes 1.24 之后不再使用 dockershim，真正作为容器运行时的是 containerd。

```bash
wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

yum -y install yum-utils
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

yum install -y docker-ce
systemctl enable --now docker

docker --version
containerd --version
systemctl status docker containerd
```

> 也可以直接安装 containerd，而不是安装完整 Docker。文章采用 Docker CE 路径，是为了方便获得 containerd 和相关工具链。

## 3. 配置 containerd

```bash
containerd config default > /etc/containerd/config.toml

# 使用 systemd cgroup 驱动
sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml

# 替换 sandbox image 镜像源
sed -i "s#registry.k8s.io#registry.aliyuncs.com/google_containers#g" /etc/containerd/config.toml
```

安装 `crictl`：

```bash
VERSION="v1.36.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```

配置 `crictl` 连接 containerd：

```bash
cat >> /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

systemctl enable --now containerd
systemctl status containerd
```

关键注意点：

- `SystemdCgroup = true` 要与 kubelet 的 cgroup 驱动保持一致；
- 国内环境需要处理镜像源访问问题；
- `crictl` 是排查 CRI 运行时问题的重要工具。

## 4. 安装 Kubernetes 组件

### 4.1 配置 Kubernetes yum 源

所有节点执行：

```bash
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.36/rpm/
enabled=1
gpgcheck=0
EOF

yum list kubelet --showduplicates | sort -r | grep 1.36
```

### 4.2 安装 kubeadm / kubelet / kubectl

```bash
yum install -y kubectl-1.36.1 kubelet-1.36.1 kubeadm-1.36.1
systemctl enable kubelet
```

组件说明：

| 组件 | 作用 |
|---|---|
| `kubeadm` | 集群初始化和节点加入工具 |
| `kubelet` | 每个节点上的核心守护进程，负责管理 Pod 生命周期 |
| `kubectl` | Kubernetes 命令行客户端 |

## 5. 初始化 Kubernetes 集群

在 master 节点执行：

```bash
kubeadm init --kubernetes-version=v1.36.1 \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=10.0.0.21 \
  --service-cidr=10.224.0.0/16 \
  --image-repository=registry.aliyuncs.com/google_containers
```

参数说明：

| 参数 | 含义 |
|---|---|
| `--kubernetes-version` | Kubernetes 版本，应与安装的组件版本一致 |
| `--apiserver-advertise-address` | API Server 对外通告地址，通常为 master 节点 IP |
| `--pod-network-cidr` | Pod 网络地址段，要与 CNI 插件配置一致 |
| `--service-cidr` | Service 虚拟 IP 地址段 |
| `--image-repository` | 指定镜像仓库，解决国内拉取镜像困难问题 |

初始化成功后会输出：

```text
Your Kubernetes control-plane has initialized successfully!
```

并给出两类重要命令：

1. 配置 kubeconfig；
2. worker 节点加入集群的 `kubeadm join` 命令。

### 5.1 初始化失败后的重置（可选）

```bash
kubeadm reset -f
rm -fr ~/.kube/ /etc/kubernetes/* /var/lib/etcd/*
```

> 注意：这是破坏性操作，会清理集群配置和 etcd 数据。生产环境不要随意执行。

## 6. 配置 kubectl 环境变量

在 master 节点执行：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

也可以配置环境变量：

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

持久化：

```bash
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```

## 7. Worker 节点加入集群

在 worker 节点执行 master 初始化输出的 `kubeadm join` 命令，例如：

```bash
kubeadm join 10.0.0.21:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

如果 token 过期，可以在 master 节点重新生成：

```bash
kubeadm token list
kubeadm token create --print-join-command
```

加入后查看节点：

```bash
kubectl get node
```

此时节点可能是 `NotReady`，通常是因为还没有安装网络插件。

## 8. 安装 Calico 网络插件

文章推荐使用 Calico。

### 8.1 下载 Calico YAML

```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/calico.yaml
```

### 8.2 修改 Pod 网段

确保 `CALICO_IPV4POOL_CIDR` 与 `kubeadm init` 中的 `--pod-network-cidr` 一致。

```yaml
- name: CALICO_IPV4POOL_CIDR
  value: "192.168.0.0/16"
```

### 8.3 部署 Calico

```bash
kubectl apply -f calico.yaml
```

## 9. 给 worker 节点打标签（可选）

```bash
kubectl label nodes k8s-node1 node-role.kubernetes.io/worker=worker
kubectl label nodes k8s-node2 node-role.kubernetes.io/worker=worker
```

## 10. 验证集群

### 10.1 查看集群信息

```bash
kubectl cluster-info
```

预期能看到 control plane 和 CoreDNS 信息。

### 10.2 查看节点状态

```bash
kubectl get node
```

预期状态：

```text
NAME         STATUS   ROLES           VERSION
k8s-master   Ready    control-plane   v1.36.1
k8s-node1    Ready    worker          v1.36.1
k8s-node2    Ready    worker          v1.36.1
```

### 10.3 查看所有 Pod

```bash
kubectl get pods -A
```

重点检查：

- `kube-system` 下的组件是否 Running；
- CoreDNS 是否 Running；
- Calico 相关 Pod 是否 Running；
- 节点是否全部 Ready。

## 排障检查清单

如果节点一直 `NotReady`，优先检查：

- containerd 是否运行：`systemctl status containerd`；
- kubelet 是否正常：`systemctl status kubelet`；
- kubelet 日志：`journalctl -u kubelet -f`；
- CNI 是否安装：`kubectl get pods -A | grep calico`；
- Pod 网段是否与 Calico 配置一致；
- swap 是否关闭：`free -h` / `swapon --show`；
- 内核参数是否生效：`sysctl net.ipv4.ip_forward`；
- 节点时间是否同步；
- 镜像是否能正常拉取。

## 生产环境注意事项

文章中的架构是 `1 master + 2 node`，更适合学习和测试。生产环境至少需要考虑：

- 多 master 高可用；
- etcd 高可用和定期备份；
- API Server 前置负载均衡；
- 节点、Pod、Service 网段规划；
- 镜像仓库内网化；
- 证书续期策略；
- 集群升级策略；
- 监控、日志、告警；
- 网络策略和安全基线；
- 备份恢复和灾备演练。

## 我的理解

这篇教程的价值在于给出了一条完整的 kubeadm 部署链路：从系统初始化、运行时配置，到集群初始化、网络插件安装和最终验证。真正部署时，不要只机械复制命令，重点要理解几个依赖关系：

1. `containerd` 的 cgroup 驱动要与 kubelet 匹配；
2. `--pod-network-cidr` 要与 Calico 的 Pod 网段一致；
3. 节点 `NotReady` 通常与 CNI、kubelet、containerd、镜像拉取有关；
4. `kubeadm join` token 有有效期，过期后需要重新生成；
5. 单 master 不是生产高可用方案。

## 后续行动项

- [ ] 在实验环境按本文步骤部署一套 1 master + 2 node 集群。
- [ ] 将部署过程中的报错记录到 [[04 知识资料/Kubernetes/Kubernetes 问题.md]]。
- [ ] 将常用验证命令补充到 [[04 知识资料/Kubernetes/Kubernetes 学习.md]]。
- [ ] 后续单独整理一篇 Kubernetes 高可用部署方案。
- [ ] 建立 Kubernetes 部署前检查清单和部署后验收清单。

## 相关笔记

- [[04 知识资料/Kubernetes/Kubernetes 学习.md]]
- [[04 知识资料/Kubernetes/Kubernetes 问题.md]]
- [[04 知识资料/Kubernetes/Kubernetes 答案.md]]
- [[04 知识资料/运维/结构化笔记/01 运维技术栈官网导航|01 运维技术栈官网导航]]

## 关联笔记

- [[04 知识资料/知识库总索引|知识库总索引]]
- [[04 知识资料/Kubernetes/Kubernetes 知识库索引|Kubernetes 知识库索引]]

