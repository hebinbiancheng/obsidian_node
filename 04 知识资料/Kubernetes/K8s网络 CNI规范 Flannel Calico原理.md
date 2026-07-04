---
title: K8s网络终极拆解：CNI规范 + Flannel/Calico原理
date: 2026-07-02
tags: [kubernetes, 网络, cni, flannel, calico, cilium, 运维]
category: 知识资料
source: https://mp.weixin.qq.com/s/sjPPzUrKsTAJ_VZNtYUGMA
---

# K8s 网络终极拆解：CNI 规范 + Flannel/Calico 原理

> 原文: [微信文章](https://mp.weixin.qq.com/s/sjPPzUrKsTAJ_VZNtYUGMA)

---

## 一、Pod 通信的三层路径

### 1. 同 Pod 内容器通信

共享同一个网络命名空间，直接 `localhost` 访问。无需配置。

### 2. 同节点 Pod 间通信

两端 Veth 插在同一个 CNI 网桥（如 `cni0`），同网段二层转发，无需网关。

### 3. 跨节点 Pod 间通信

```
Pod1(节点A) → 默认路由 → 节点A网桥
→ 查路由：目标Pod IP 下一跳=节点B
→ VXLAN/IPIP 隧道封装（取决于CNI方案）
→ 节点B → Pod2
```

> ⚠️ 扔掉 `docker0` 的概念！现代集群中路由和网桥完全由 CNI 插件管理，每个节点分配独立子网（如 /24）。

### 纠正：Service IP 不走路由

访问 ClusterIP 时，目标 MAC 是网桥的 MAC，到达后被宿主机 **iptables DNAT** 规则截获，重定向到后端 Pod IP。**Service IP 只是 iptables 的钩子，不在网络层路由。**

---

## 二、CNI 规范

CNI = 容器网络接口，只定义了两个操作：**ADD**（加入网络）和 **DEL**（退出网络）。

### 核心组成

| 组件 | 说明 |
|------|------|
| CNI Plugin | 可执行文件（bridge、calico），创建 Veth、分配 IP、设置路由 |
| IPAM Plugin | IP 地址管理（host-local、dhcp、calico-ipam） |
| 网络配置 | JSON 格式，告诉运行时调用哪个插件 |
| 网络配置列表 | 允许容器同时接入多个网络 |

### kubelet 工作流程

```
创建 Pod → 读 /etc/cni/net.d/ 配置
→ 调用 /opt/cni/bin/ 可执行文件 → ADD 操作
→ 删除 Pod → DEL 操作 → 回收 IP
```

> Kubernetes 不再使用 Docker 的 `--net=bridge`。`kubenet` 已弃用，生产不要碰。

### Deployment 与 CNI 协同流程

```
kubectl apply -f deployment.yaml
→ K8s 调度选定 Node
→ kubelet 创建 Pod 网络命名空间
→ CNI 插件自动介入（创建 Veth、分配 IP、配置路由）
→ Pod 启动，eth0 已就绪
```

> YAML 说"我要一个盒子"，kubelet 和 CNI 负责接网线、分地址。对应用完全透明。

---

## 三、四大网络方案原理与误区纠正

### 1. Flannel

| 模式 | 原理 | 适用 |
|------|------|------|
| **VXLAN**（主流） | 内核 VXLAN 封装，flanneld 只管理子网和路由，**不参与数据转发** | 跨子网 |
| **host-gw** | 直接路由，无隧道，需二层连通 | 同二层 |
| UDP | ~~用户态转发~~ | ⚠️ 已淘汰 |

> ❌ 误区：flannel0 网桥 + docker0。现代 Flannel 用 `flannel.1` VXLAN 接口，内核直接封装。

### 2. Calico

**纯三层 BGP，无网桥**：

- 每个节点运行虚拟路由器，BGP 交换路由
- 每个 Pod 得 /32 地址，路由表直指节点 IP
- 数据转发由内核路由完成，**无隧道、无封包解包**，性能接近物理网络
- 支持 IPIP（跨子网隧道）和 VXLAN 模式
- Network Policy 基于 iptables

> ❌ 误区：Calico 必须手动配置 BGP。同二层下自动构建全网状路由，大规模集群才需 BGP Route Reflector。

### 3. 直接路由 + 动态路由协议

节点路由表添加静态路由：`目标 Pod 子网 → 目标节点 IP`。性能无限接近线速，但要求同二层。

现在用 **FRRouting (FRR)** 或 **BIRD** 运行动态路由协议（BGP/OSPF）。

> ❌ 不要再提 Quagga/Zebra，已停止维护。

### 4. Open vSwitch / OVN

OVS 是强大的虚拟交换机，但在 K8s 中更多用 **OVN** 作为控制器自动管理。手动 `ovs-vsctl` 建隧道的内容已过时。

---

## 四、方案选择

| 方案 | 原理 | 优点 | 适用 | 常见坑 |
|------|------|------|------|--------|
| **Flannel VXLAN** | 内核 VXLAN 隧道 | 简单、稳定、低资源 | 中小集群 (<100 节点) | 误认为 flanneld 参与转发 |
| **Calico BGP** | 三层路由 + BGP | 高性能、Network Policy | 中大集群、需流量控制 | 跨子网需配 IPIP/VXLAN |
| **直接路由 + FRR** | 动态路由协议 | 最低延迟 | 同二层数据中心 | 新增节点需更新路由 |
| **Cilium (eBPF)** | 内核 eBPF | 极高性能、L7 策略 | 金融、安全要求高 | 需 Linux 4.18+ |

### 推荐

- **入门**：Flannel VXLAN
- **生产**：Calico（理解 BGP vs IPIP 区别）
- **前沿**：Cilium（网络 + 安全 + 可观测性三合一）

---

## 相关笔记

- [[K8s Gateway API 完全指南]]
- [[K8s PVC 绑定 PV 全过程]]
- [[etcd 如何保存 Kubernetes 状态]]
