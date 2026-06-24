---
title: Pod 创建与声明式控制流程
type: knowledge
status: evergreen
source_type: 微信公众号合集
created: 2026-06-24
updated: 2026-06-24
tags:
  - Kubernetes
  - Pod
  - 声明式API
source:
  - "https://mp.weixin.qq.com/s?__biz=Mzk2NDA2MTE5MA==&mid=2247483825&idx=1&sn=2d49d496507098eba6ef75b9ef8847fa&chksm=c54688be0b2ac285d35a4ad8d15653db3eaaef83fca8b9101a41131077b1441e7f62e5d0df5b&mpshare=1&scene=24&srcid=0618oXhzAl6Jrtx79qVvsIQs&sharer_shareinfo=0c18ce937673862328c0e76e96fbecca&sharer_shareinfo_first=0c18ce937673862328c0e76e96fbecca#rd"
  - "https://mp.weixin.qq.com/s?__biz=MzU3NzA0MjUwNQ==&mid=2247484017&idx=1&sn=6c858b5740f02200ba7345fa1167e186&chksm=fcc04f8f2a68eaf04ee19d861f8281f637aa3ee71898bc2d310a923bb8bb1323b543e20dbea8&mpshare=1&scene=24&srcid=0621E8QuoQOiWWmNWd1TtZL2&sharer_shareinfo=dcf3420cd217abf652d3d9daa283108e&sharer_shareinfo_first=dcf3420cd217abf652d3d9daa283108e#rd"
  - "https://mp.weixin.qq.com/s?__biz=Mzk2NDA2MTE5MA==&mid=2247483729&idx=1&sn=98a5bb4679f1df3007e0c4164f28d130&chksm=c56dd3afabc4eaec98dc78883ca401bba51034b6e4f0da7eba214126895b3444f7b59d5c2046&mpshare=1&scene=24&srcid=0622cp6dW1eGrgN9TCsqB9Bt&sharer_shareinfo=2b5a51ff015f2af98f4476f17ae16d1d&sharer_shareinfo_first=2b5a51ff015f2af98dc78883ca401bba51034b6e4f0da7eba214126895b3444f7b59d5c2046#rd"
aliases: []
---

# Pod 创建与声明式控制流程

## 一句话结论

`kubectl apply` 并不是直接“命令集群创建容器”，而是把期望状态写入 API Server 和 etcd，再由控制器、调度器、kubelet 分工完成实际运行。

## 主流程

1. 用户执行 `kubectl apply -f nginx.yaml`。
2. kubectl 校验参数并携带认证信息请求 kube-apiserver。
3. API Server 完成认证、授权、准入控制和对象校验。
4. API Server 将 Deployment 等资源对象写入 etcd。
5. Deployment Controller 监听到期望状态，创建 ReplicaSet。
6. ReplicaSet Controller 发现副本不足，创建未绑定节点的 Pod。
7. Scheduler 监听到无 `nodeName` 的 Pod，经过过滤和打分选择目标 Node。
8. kubelet 监听到本节点 Pod，准备目录、Secret、ConfigMap 和镜像。
9. kubelet 通过 CRI 调用 containerd 等 Runtime 创建 pause 容器和业务容器。
10. CNI 为 Pod 配置网络，kubelet 持续上报 Pod 状态。

## 声明式控制

- 用户提交的是“期望状态”，不是一次性命令脚本。
- 控制器循环比较期望状态和实际状态，发现偏差就修复。
- 这种机制让 Kubernetes 具备自愈、扩缩容和滚动更新能力。
