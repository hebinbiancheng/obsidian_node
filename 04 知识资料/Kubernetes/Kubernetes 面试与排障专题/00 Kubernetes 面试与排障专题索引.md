---
title: Kubernetes 面试与排障专题索引
type: knowledge
status: evergreen
source_type: 微信公众号合集
created: 2026-06-24
updated: 2026-06-24
tags:
  - Kubernetes
  - 容器
  - 排障
  - 面试
source:
  - "https://mp.weixin.qq.com/s?__biz=MzYzOTc5Nzg5MQ==&mid=2247484211&idx=1&sn=b68d29a1ea4d07f0a32d313fd725434b&chksm=f1cd7c44068fa390b11c9017fc13530547d0fe584638484cdb2bcc89006d379b37910fd4b79d&mpshare=1&scene=24&srcid=06174MRTmy4zy5t85P3il7DF&sharer_shareinfo=b32d114f9496d943303418a55455d782&sharer_shareinfo_first=b32d114f9496d943303418a55455d782#rd"
  - "https://mp.weixin.qq.com/s?__biz=Mzk2NDA2MTE5MA==&mid=2247483825&idx=1&sn=2d49d496507098eba6ef75b9ef8847fa&chksm=c54688be0b2ac285d35a4ad8d15653db3eaaef83fca8b9101a41131077b1441e7f62e5d0df5b&mpshare=1&scene=24&srcid=0618oXhzAl6Jrtx79qVvsIQs&sharer_shareinfo=0c18ce937673862328c0e76e96fbecca&sharer_shareinfo_first=0c18ce937673862328c0e76e96fbecca#rd"
  - "https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247486318&idx=1&sn=9354b9da91461fb8b015d8d9cd1bd0e0&chksm=c36f5720cb852fa547c6acc9485be583327bab7244c870435775847ca255ceb48b6aac59a47a&mpshare=1&scene=24&srcid=0618AsERVhSNQu7mrRLC7G15&sharer_shareinfo=9bb81a72b1a60a8c875188e0ab071d57&sharer_shareinfo_first=9bb81a72b1a60a8c875188e0ab071d57#rd"
  - "https://mp.weixin.qq.com/s?__biz=Mzg4NzgzOTQ4MQ==&mid=2247484459&idx=1&sn=6f852042c495c5ddbbabc046ac75442d&chksm=cee6b43b0e18f63d183606d4dcf00bf00cf037c601fa77a346515e5d87179f336d3d3c3c13a4&mpshare=1&scene=24&srcid=0618vD0GyPgxPTpIX8R5wyn9&sharer_shareinfo=3f8dd55f029c21b112011addaa219fd0&sharer_shareinfo_first=3f8dd55f029c21b112011addaa219fd0#rd"
  - "https://mp.weixin.qq.com/s?__biz=Mzk2NDA2MTE5MA==&mid=2247483791&idx=1&sn=bfb8b8ff88ca2f8c6bd2b66da1f7e473&chksm=c5362be121dd47914d3891d3582092ff49e4561a21775df4842d016f4acae9f519e9a8e8c064&mpshare=1&scene=24&srcid=0620SIAnQDuz1yKbHanKXLlw&sharer_shareinfo=b0f37e5608c7ae03f168d86eca5ccdf1&sharer_shareinfo_first=b0f37e5608c7ae03f168d86eca5ccdf1#rd"
  - "https://mp.weixin.qq.com/s?__biz=MzI2MTMwMTkxMQ==&mid=2247491507&idx=1&sn=b62ab8d12a530c146acb01372809752a&chksm=eb2f5e885a6017481ccd210da61c2818caac296c229065629fd8887fc4e9ba91b73b4d0a0238&mpshare=1&scene=24&srcid=0620iYbLsDirryFMlQZSewN5&sharer_shareinfo=9dfae358f1f45f2f7b92f6dddf3f8733&sharer_shareinfo_first=9dfae358f1f45f2f7b92f6dddf3f8733#rd"
  - "https://mp.weixin.qq.com/s?__biz=MzU3NzA0MjUwNQ==&mid=2247484017&idx=1&sn=6c858b5740f02200ba7345fa1167e186&chksm=fcc04f8f2a68eaf04ee19d861f8281f637aa3ee71898bc2d310a923bb8bb1323b543e20dbea8&mpshare=1&scene=24&srcid=0621E8QuoQOiWWmNWd1TtZL2&sharer_shareinfo=dcf3420cd217abf652d3d9daa283108e&sharer_shareinfo_first=dcf3420cd217abf652d3d9daa283108e#rd"
  - "https://mp.weixin.qq.com/s?__biz=Mzk2NDA2MTE5MA==&mid=2247483729&idx=1&sn=98a5bb4679f1df3007e0c4164f28d130&chksm=c56dd3afabc4eaec98dc78883ca401bba51034b6e4f0da7eba214126895b3444f7b59d5c2046&mpshare=1&scene=24&srcid=0622cp6dW1eGrgN9TCsqB9Bt&sharer_shareinfo=2b5a51ff015f2af98f4476f17ae16d1d&sharer_shareinfo_first=2b5a51ff015f2af98dc78883ca401bba51034b6e4f0da7eba214126895b3444f7b59d5c2046#rd"
aliases: []
---

# Kubernetes 面试与排障专题索引

## 学习路径

1. [[01 容器与 Kubernetes 基础架构]]：先理解容器、Namespace、Cgroups、控制平面和节点组件。
2. [[02 Pod 创建与声明式控制流程]]：掌握从 kubectl apply 到 Pod Running 的完整链路。
3. [[03 CoreDNS 与服务发现]]：理解 Service、DNS 名称和 CoreDNS 插件如何协同。
4. [[04 Kubernetes 排障与面试题]]：按集群、节点、Pod、网络、存储分层排查。
5. [[05 kubectl 常用命令速查]]：沉淀常用命令和适用场景。
6. [[06 原文解析要点与图片索引]]：保留来源与图片索引。

## 核心地图

- 容器基础：Namespace 负责隔离视图，Cgroups 负责资源限制，镜像负责应用交付。
- K8s 思想：用户提交期望状态，控制器持续让现实状态逼近期望状态。
- 控制面：API Server 是唯一入口，etcd 是事实来源，Scheduler 做调度，Controller Manager 做状态修复。
- 节点侧：kubelet 管 Pod 生命周期，Runtime 真正创建容器，kube-proxy/CoreDNS 支撑服务访问。
- 排障方法：先判断范围，再定位层级，最后执行命令并验证恢复。

## 相关笔记

- [[04 知识资料/Kubernetes/结构化笔记/00 Kubernetes 知识库索引|Kubernetes 知识库索引]]
- [[04 知识资料/Kubernetes/结构化笔记/01 Kubernetes 核心架构与组件|Kubernetes 核心架构与组件]]
- [[04 知识资料/Kubernetes/结构化笔记/05 Kubernetes 面试问答精粹|Kubernetes 面试问答精粹]]
