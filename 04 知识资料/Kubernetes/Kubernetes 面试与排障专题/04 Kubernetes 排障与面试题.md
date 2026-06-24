---
title: Kubernetes 排障与面试题
type: knowledge
status: evergreen
source_type: 微信公众号合集
created: 2026-06-24
updated: 2026-06-24
tags:
  - Kubernetes
  - 排障
  - 面试
source:
  - "https://mp.weixin.qq.com/s?__biz=MzYzOTc5Nzg5MQ==&mid=2247484211&idx=1&sn=b68d29a1ea4d07f0a32d313fd725434b&chksm=f1cd7c44068fa390b11c9017fc13530547d0fe584638484cdb2bcc89006d379b37910fd4b79d&mpshare=1&scene=24&srcid=06174MRTmy4zy5t85P3il7DF&sharer_shareinfo=b32d114f9496d943303418a55455d782&sharer_shareinfo_first=b32d114f9496d943303418a55455d782#rd"
  - "https://mp.weixin.qq.com/s?__biz=Mzk2NDA2MTE5MA==&mid=2247483791&idx=1&sn=bfb8b8ff88ca2f8c6bd2b66da1f7e473&chksm=c5362be121dd47914d3891d3582092ff49e4561a21775df4842d016f4acae9f519e9a8e8c064&mpshare=1&scene=24&srcid=0620SIAnQDuz1yKbHanKXLlw&sharer_shareinfo=b0f37e5608c7ae03f168d86eca5ccdf1&sharer_shareinfo_first=b0f37e5608c7ae03f168d86eca5ccdf1#rd"
aliases: []
---

# Kubernetes 排障与面试题

## 一句话结论

K8s 排障要先判断故障范围，再按控制面、节点、网络、Pod、应用逐层收敛，避免一上来只盯日志。

## 通用排障路径

1. 判断范围：集群级、节点级、Pod 级、应用级。
2. 看控制面：API Server、etcd、Controller Manager、Scheduler。
3. 看节点：kubelet、Runtime、磁盘、CPU、内存、网络。
4. 看 Pod：状态、事件、日志、探针、资源限制、镜像拉取。
5. 看网络：Service、Endpoint、CoreDNS、CNI、NetworkPolicy。
6. 修复后验证：确认状态恢复、错误率下降、监控稳定。

## 高频问题

### ConfigMap 挂载为什么默认只读

ConfigMap 是外部配置源，Pod 只应读取，不应在容器内直接修改。允许容器改写会导致集群对象与容器文件不一致，也会破坏多 Pod 共享配置和更新机制。需要写入时，应复制到 `emptyDir` 等可写目录。

### kube-apiserver 无法启动怎么排查

- 查看静态 Pod 或系统服务状态。
- 检查日志、启动参数、证书、端口占用和磁盘空间。
- 检查 etcd 健康、时间同步和 `/etc/kubernetes/manifests/` 配置。
- 注意 kube-apiserver 是控制面入口，故障会导致整个集群写入和查询异常。

### etcd 空间爆满会怎样

- 无法创建或更新资源，`kubectl apply` 失败。
- Controller 无法写入状态，集群可能表现为卡死。
- 处理方向包括告警确认、备份、压缩、碎片整理、扩容和根因排查。

## 图片资料

![[assets/k8s-05.png]]
![[assets/k8s-06.png]]
![[assets/k8s-07.png]]
![[assets/k8s-08.png]]
