---
title: 容器与 Kubernetes 基础架构
type: knowledge
status: evergreen
source_type: 微信公众号合集
created: 2026-06-24
updated: 2026-06-24
tags:
  - Kubernetes
  - 容器
  - Namespace
  - Cgroups
source:
  - "https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247486318&idx=1&sn=9354b9da91461fb8b015d8d9cd1bd0e0&chksm=c36f5720cb852fa547c6acc9485be583327bab7244c870435775847ca255ceb48b6aac59a47a&mpshare=1&scene=24&srcid=0618AsERVhSNQu7mrRLC7G15&sharer_shareinfo=9bb81a72b1a60a8c875188e0ab071d57&sharer_shareinfo_first=9bb81a72b1a60a8c875188e0ab071d57#rd"
  - "https://mp.weixin.qq.com/s?__biz=MzU3NzA0MjUwNQ==&mid=2247484017&idx=1&sn=6c858b5740f02200ba7345fa1167e186&chksm=fcc04f8f2a68eaf04ee19d861f8281f637aa3ee71898bc2d310a923bb8bb1323b543e20dbea8&mpshare=1&scene=24&srcid=0621E8QuoQOiWWmNWd1TtZL2&sharer_shareinfo=dcf3420cd217abf652d3d9daa283108e&sharer_shareinfo_first=dcf3420cd217abf652d3d9daa283108e#rd"
  - "https://mp.weixin.qq.com/s?__biz=Mzk2NDA2MTE5MA==&mid=2247483729&idx=1&sn=98a5bb4679f1df3007e0c4164f28d130&chksm=c56dd3afabc4eaec98dc78883ca401bba51034b6e4f0da7eba214126895b3444f7b59d5c2046&mpshare=1&scene=24&srcid=0622cp6dW1eGrgN9TCsqB9Bt&sharer_shareinfo=2b5a51ff015f2af98f4476f17ae16d1d&sharer_shareinfo_first=2b5a51ff015f2af98dc78883ca401bba51034b6e4f0da7eba214126895b3444f7b59d5c2046#rd"
aliases: []
---

# 容器与 Kubernetes 基础架构

## 一句话结论

容器是基于 Linux 内核能力实现的进程隔离与资源限制，Kubernetes 则把容器编排成“声明期望状态、持续自动修复”的控制系统。

## 容器是什么

- 容器把应用、依赖、运行时和配置打包为可迁移单元。
- 容器不是虚拟机，它共享宿主机内核，启动快、资源开销低。
- 隔离性来自 Namespace，资源治理来自 Cgroups。

## Namespace

- PID Namespace：让容器只看到自己的进程树。
- Mount Namespace：让容器拥有独立文件系统视图。
- Network Namespace：隔离网卡、路由表、端口和网络栈。
- UTS、IPC、User、Cgroup Namespace 分别处理主机名、进程通信、用户映射和资源视图。

## Cgroups

- 限制 CPU、内存、IO 等资源使用。
- 支撑 Kubernetes requests/limits 的底层资源约束。
- 排障时要关注 OOMKilled、CPU throttling、磁盘 IO 饱和等症状。

## Kubernetes 控制面

- API Server：统一入口，负责认证、授权、准入控制、校验和 Watch 通知。
- etcd：保存所有集群状态，是唯一事实来源。
- Scheduler：为未绑定节点的 Pod 选择 Node。
- Controller Manager：不断对比期望状态与现实状态，执行补偿动作。

## 节点组件

- kubelet：节点代理，负责 Pod 生命周期管理并回报状态。
- Container Runtime：如 containerd，负责拉镜像和创建容器。
- kube-proxy：维护 Service 访问规则。
- CoreDNS：提供集群内部服务名解析。

## 图片资料

![[assets/k8s-01.jpg]]
![[assets/k8s-02.png]]
![[assets/k8s-03.gif]]
![[assets/k8s-04.png]]
