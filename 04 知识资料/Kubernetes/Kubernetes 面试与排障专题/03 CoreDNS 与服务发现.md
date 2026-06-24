---
title: CoreDNS 与 Kubernetes 服务发现
type: knowledge
status: evergreen
source_type: 微信公众号合集
created: 2026-06-24
updated: 2026-06-24
tags:
  - Kubernetes
  - CoreDNS
  - 服务发现
source:
  - "https://mp.weixin.qq.com/s?__biz=Mzg4NzgzOTQ4MQ==&mid=2247484459&idx=1&sn=6f852042c495c5ddbbabc046ac75442d&chksm=cee6b43b0e18f63d183606d4dcf00bf00cf037c601fa77a346515e5d87179f336d3d3c3c13a4&mpshare=1&scene=24&srcid=0618vD0GyPgxPTpIX8R5wyn9&sharer_shareinfo=3f8dd55f029c21b112011addaa219fd0&sharer_shareinfo_first=3f8dd55f029c21b112011addaa219fd0#rd"
aliases: []
---

# CoreDNS 与 Kubernetes 服务发现

## 一句话结论

CoreDNS 把 Kubernetes 的 Service 发现能力暴露为标准 DNS 查询，让应用通过稳定服务名访问动态变化的 Pod 后端。

## 为什么需要 CoreDNS

- Pod IP 会随重建、扩缩容和调度变化。
- 应用不应该硬编码 Pod IP。
- Service 提供稳定虚拟入口，CoreDNS 提供稳定名称解析。

## 标准访问形式

- 同命名空间：`service-name`。
- 跨命名空间：`service-name.namespace`。
- 完整域名：`service-name.namespace.svc.cluster.local`。

## 工作链路

1. 应用访问服务名。
2. Pod 内 DNS 配置把查询转发给 CoreDNS。
3. CoreDNS 的 kubernetes 插件查询 Service/EndpointSlice 信息。
4. CoreDNS 返回 ClusterIP 或无头服务的后端 Pod 记录。
5. 应用通过返回地址访问目标服务。

## 排障要点

- 检查 CoreDNS Pod 是否 Running。
- 检查 Service 和 EndpointSlice 是否存在后端。
- 在业务 Pod 内用 `nslookup` 或 `dig` 验证解析。
- 检查 NetworkPolicy、CNI、kube-proxy 或 Service 配置是否阻断访问。
