---
title: K8s Service 类型与 Ingress 面试题
type: knowledge
status: evergreen
source_type: 微信公众号
created: 2026-06-09
tags:
  - K8s
  - Service
  - Ingress
  - 云原生
  - 面试
source: https://mp.weixin.qq.com/s/jrDopNuj47XrC_0BEQ8QBA
confidence: high
---

# K8s Service 类型与 Ingress 面试题

> 原文：[微信文章](https://mp.weixin.qq.com/s/jrDopNuj47XrC_0BEQ8QBA) · 2026-06-09
> 原始资料：`^[raw/articles/wechat-k8s-service-ingress-2026.html]`

---

## 一句话总结

K8s Service 四种类型 + Ingress 七层路由——三题覆盖服务暴露、底层关系、域名路由的核心考点。

---

## 一、Service 四种类型

| 类型 | 访问范围 | 适用场景 |
|------|---------|---------|
| **ClusterIP** | 仅集群内部 | 微服务间通信 |
| **NodePort** | 节点 IP:端口 | 测试环境、简单外部访问 |
| **LoadBalancer** | 云厂商 LB 公网 | 生产环境公网服务 |
| **ExternalName** | 集群内→外部域名映射 | 集群服务指向外部 DNS |

> 生产最常见：ClusterIP + Ingress 或 LoadBalancer。

---

## 二、ClusterIP / NodePort / LoadBalancer 区别

**底层关系**：

```
LoadBalancer → NodePort → ClusterIP
```

- NodePort 底层依赖 ClusterIP
- LoadBalancer 底层转发到 NodePort
- ClusterIP 是最基础的 Service 类型

| 维度 | ClusterIP | NodePort | LoadBalancer |
|------|-----------|----------|--------------|
| **暴露范围** | 集群内 | 节点端口 | 公网 |
| **访问方式** | 集群 DNS/IP | 节点IP:30000-32767 | 云 LB 公网 IP |
| **适用** | 内部通信 | 测试 | 生产 |

---

## 三、Ingress

**本质**：K8s 的七层反向代理网关——统一管理 HTTP/HTTPS 流量入口。

| 功能 | 示例 |
|------|------|
| 域名路由 | `www.test.com/api` → service-a |
| 路径转发 | `www.test.com/web` → service-b |
| HTTPS/TLS | 证书管理 |
| 负载均衡 | 多副本分发 |
| 统一入口 | 一个公网 IP 代理多个 Service |

> Ingress 需要 Ingress Controller（ingress-nginx / Traefik）才能生效。

**生产推荐**：`Ingress + ClusterIP Service`，而非直接 NodePort 对外暴露。
