---
title: kubectl 常用命令速查
type: knowledge
status: evergreen
source_type: 微信公众号合集
created: 2026-06-24
updated: 2026-06-24
tags:
  - Kubernetes
  - kubectl
  - 命令
source:
  - "https://mp.weixin.qq.com/s?__biz=MzI2MTMwMTkxMQ==&mid=2247491507&idx=1&sn=b62ab8d12a530c146acb01372809752a&chksm=eb2f5e885a6017481ccd210da61c2818caac296c229065629fd8887fc4e9ba91b73b4d0a0238&mpshare=1&scene=24&srcid=0620iYbLsDirryFMlQZSewN5&sharer_shareinfo=9dfae358f1f45f2f7b92f6dddf3f8733&sharer_shareinfo_first=9dfae358f1f45f2f7b92f6dddf3f8733#rd"
aliases: []
---

# kubectl 常用命令速查

## 集群信息

```bash
kubectl get nodes
kubectl get nodes -o wide
kubectl cluster-info
```

## Pod

```bash
kubectl get pods
kubectl get pods -A
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs -f <pod>
kubectl logs <pod> -c <container>
kubectl exec -it <pod> -- bash
```

## Deployment

```bash
kubectl get deploy
kubectl describe deploy <name>
kubectl create deployment nginx --image=nginx:latest
kubectl scale deployment nginx --replicas=5
kubectl set image deployment/nginx nginx=nginx:1.27
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout undo deployment/nginx
```

## Service 与网络

```bash
kubectl get svc
kubectl describe svc <service>
kubectl get endpoints
kubectl get endpointslice
kubectl run -it --rm debug --image=busybox -- sh
nslookup <service>
```

## 排障建议

- `get` 看状态，`describe` 看事件，`logs` 看应用输出，`exec` 进容器验证环境。
- 控制面异常时，不要只依赖 kubectl，必要时登录节点查看 systemd、容器运行时和静态 Pod。
