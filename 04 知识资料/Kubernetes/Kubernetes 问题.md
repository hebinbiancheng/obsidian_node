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
  - Kubernetes 问题
---
1. 假设集群有 2 个 node 节点，其中一个有 pod，另一个则没有，那么新的 pod 会被调度到哪个节点上？[kubernetes答案](kubernetes答案.md)

2. 应用程序通过容器的形式运行，如果 OOM（Out-of-Memory）了，是容器重启还是所在的 Pod 被重建？

3. 应用程序配置如环境变量或者 `ConfigMap` 可以不重建 Pod 实现动态更新吗？

4. pod 被创建后是稳定的吗，即使用户不进行任何操作？

5. 使用 `ClusterIP` 类型的 `Service` 能保证 TCP 流量的负载均衡吗？

6. 应用日志要怎么采集，会不会有丢失的情况？

7. 某个 HTTP Server Pod 的 `livenessProbe` 正常是否就一定没问题？

8. 应用程序如何扩展以应对流量波动的情况？

9. 当你执行 `kubectl exec -it <pod> -- bash` 之后是登录到了 pod 里吗？

10. 如果 Pod 里的容器反复退出并重启，如何排查？

## 关联笔记

- [[04 知识资料/知识库总索引|知识库总索引]]
- [[04 知识资料/Kubernetes/Kubernetes 知识库索引|Kubernetes 知识库索引]]

