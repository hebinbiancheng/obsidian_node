---
title: K8s面试每日3题 Day18 - Pod故障排查
date: 2026-07-04
tags: [Kubernetes, 面试, 运维, Pod, 排障]
category: 知识资料
source: https://mp.weixin.qq.com/s/4GqjMsLtRfLXon6zfCPxTA
---

# K8s 面试每日 3 题 | Day 18 — Pod 故障排查

> 原文: [微信文章](https://mp.weixin.qq.com/s/4GqjMsLtRfLXon6zfCPxTA)

---

[IMG]
专注云原生 · 运维 · 架构 · 实战
温馨提示
🔍
关注公众号「云原生运维之道」，让运维不再困难
🧠
于文末加入「技术交流讨论群」，可在线答疑解惑
🗂️
关注公众号回复领取资料包，即可获取全套面试题
【K8s 面经实录】
[IMG]
01
问：Pod 一直 CrashLoopBackOff 如何排查？
答
：Pod 一直 CrashLoopBackOff，说明容器启动后反复异常退出，Kubernetes 不断重启它，并进入退避重试。
排查思路一般是：
先看 Pod 事件，确认退出原因：
kubectl describe pod <pod-name> -n <namespace>
重点看 Events、Exit Code、Reason、Probe 失败信息。
再看应用日志：
kubectl logs <pod-name> -n <namespace>
如果容器已经重启过，要看上一次日志：
kubectl logs <pod-name> -n <namespace> --previous
常见原因主要有：
启动命令或启动参数错误
配置文件、环境变量缺失
依赖服务连接失败，比如数据库、Redis、注册中心
Liveness Probe 配置不合理
内存不足导致 OOMKilled
挂载目录权限不足
应用本身启动报错
生产环境中我一般会优先看
describe
和
logs --previous
，因为 CrashLoopBackOff 很多时候当前日志已经被新容器覆盖，上一次失败日志更关键。
面试官考察点
：
是否真正排查过 Pod 启动失败问题
是否知道
describe
、
logs
、
logs --previous
的使用
是否能区分应用问题、配置问题、资源问题和探针问题
是否理解 CrashLoopBackOff 是结果，不是根因
02
问：Pod 处于 Pending 状态如何定位问题？
答
：Pod 处于 Pending，说明 Pod 已创建，但还没有成功运行，通常是卡在调度或资源准备阶段。
定位时我一般先看事件：
kubectl describe pod <pod-name> -n <namespace>
重点看 Events 中的报错，常见原因有：
节点 CPU、内存资源不足，导致
FailedScheduling
PVC 没有绑定成功，导致 Pod 等待存储
节点有 Taint，但 Pod 没有对应 Toleration
nodeSelector、nodeAffinity 条件不匹配
镜像拉取慢或镜像仓库访问异常
集群节点 NotReady，无法调度
如果是资源问题，还会继续看节点资源：
kubectl top node
kubectl describe node <node-name>
如果是 PVC 问题，会继续看：
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>
面试官考察点
：
是否理解 Pending 通常和调度、资源、存储有关
是否会通过 Events 定位
FailedScheduling
是否理解 request、Taint、Affinity、PVC 对调度的影响
是否具备生产环境调度类问题排查经验
03
问：Pod 频繁 OOMKilled 的原因有哪些？
答
：Pod 出现 OOMKilled，说明容器使用的内存超过了 limit，被 Linux 内核强制杀死。
常见原因包括：
应用本身内存泄漏
JVM 参数配置不合理
limit 设置过小
程序瞬时内存暴涨
缓存、线程、连接数过多
大量数据一次性加载到内存
节点本身内存不足
排查时我一般会先看：
kubectl describe pod <pod-name>
重点看：
Reason: OOMKilled
然后再结合：
kubectl top pod
查看 Pod 实际内存使用情况。
如果是 Java 应用，还需要重点检查：Xms、Xmx、容器 limit 是否匹配。
生产环境中比较常见的问题是：
limit 配置太小
JVM 最大堆超过容器 limit
导致容器被频繁杀死。
面试官考察点
：
是否理解 OOMKilled 的本质是内存超限
是否知道 Kubernetes limit 与 Linux OOM 的关系
是否真正处理过 Java、Redis 等内存问题
是否具备基础资源调优能力
什么？三道题不过瘾？猫猫特意为大家准备了完整的面试题和各大厂面经，都是PDF版，排版清晰，打印出来直接背，助你轻松拿offer。
有简历相关需求的也可私聊，可免费在线帮看。
点击这篇文章：
爆肝整理300+运维面试题 | 含2026年多个大厂面经，祝你轻松拿offer！
或者关注后，后台私信：面试题。即可领取完整版资料。
➡️
如何
加入技术群：关注后，后台发送 '进群'
[IMG]

---

## 三题速查

| # | 问题 | 关键命令 | 常见原因 |
|---|------|----------|----------|
| 1 | Pod CrashLoopBackOff | `describe` + `logs --previous` | 启动命令错、依赖服务不通、OOM、探针配置 |
| 2 | Pod Pending | `describe` + `get events` | 资源不足、PVC未绑定、Taint、节点NotReady |
| 3 | Pod OOMKilled | `describe` + `top pod` | 内存泄漏、limit太小、JVM堆超大、瞬时暴涨 |

---

## 相关笔记

- [[AI Agent 面试题与答案]] — 含 K8s 相关面试题（Q35）
