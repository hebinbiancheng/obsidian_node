---
title: kubectl 常用管理命令速查
date: 2026-07-02
tags: [kubernetes, kubectl, 命令速查, cli]
category: 知识资料
source: https://mp.weixin.qq.com/s/L9kM8KcMWbzca50Gk8Yc_A
---

# kubectl 常用管理命令速查

> 原文: [微信文章](https://mp.weixin.qq.com/s/L9kM8KcMWbzca50Gk8Yc_A)

---

## 1. YAML 管理

```bash
kubectl create -f xxx.yaml          # 创建资源
kubectl apply -f xxx.yaml           # 更新资源
kubectl delete -f xxx.yaml          # 删除资源

# 获取创建清单（dry-run 不真实创建）
kubectl create namespace test --dry-run=client -o yaml
```

---

## 2. 资源管理

### 查看资源

```bash
kubectl get pod                              # 所有 pod
kubectl get pod podname                      # 指定 pod
kubectl get pod -o wide                      # 含调度信息
kubectl get pod -o yaml                      # YAML 格式
kubectl get pod podname -o yaml --export     # 过滤 status 的 YAML
kubectl get pod -v=9                         # 交互流程详情
kubectl get pod --show-labels                # 含标签
kubectl get pod -l app                       # 按标签过滤
kubectl get pod -w                           # 实时监控
kubectl get rc -o wide                       # RC 含选择器
kubectl get ev                               # 集群事件
```

### JSONPath 查询

```bash
# 获取指定属性
kubectl get node -l kubernetes.io/hostname=k8s-master \
  -o jsonpath='{.items[0].metadata.name}'

# 循环遍历
kubectl get node -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

### 修改资源

```bash
kubectl label pod nginx run=web --overwrite   # 修改标签
kubectl edit pod nginx                        # 在线编辑
```

### 删除资源

```bash
kubectl delete pod nginx
kubectl delete pod nginx --force --grace-period=0   # 强制删除
```

### 容器操作

```bash
kubectl cp filename podname:dir          # 文件→容器
kubectl cp podname:dir filename          # 容器→本地
kubectl exec -it busybox -- /bin/sh      # 进入容器
kubectl exec -it busybox -c busybox2 -- /bin/sh   # 指定容器
```

### 扩缩容与滚动更新

```bash
kubectl scale rc myweb --replicas=3
kubectl rolling-update myweb2 -f nginx-rc.yaml --update-period=1s
kubectl rolling-update myweb myweb2 --rollback
```

### 连接远程 API Server

```bash
kubectl -s 10.0.0.81:6443 get pod
```

---

## 3. 日志查询

```bash
kubectl logs podname -f                              # 实时日志
kubectl logs podname -c containername                # 指定容器
kubectl logs -l app=test -f --tail=-1                # 标签过滤，全量日志
```

---

## 4. 名称空间管理

```bash
kubectl create namespace mynamespace           # 创建
kubectl get namespace                          # 查看
kubectl delete namespace mynamespace           # 删除（含所有资源）
kubectl get all --all-namespaces               # 所有 ns 的资源
kubectl get all --namespace mynamespace        # 指定 ns
```

---

## 5. 集群管理

### 节点操作

```bash
kubectl get componentstatus             # master 健康状态
kubectl get all                         # 所有资源
kubectl get node                        # 节点信息
kubectl cordon nodename                 # 取消调度（添加污点）
kubectl uncordon nodename               # 恢复调度
kubectl drain nodename --delete-local-data --force --ignore-daemonsets  # 驱逐 Pod
kubectl delete nodename                 # 删除节点
```

---

## 6. API 查询

```bash
kubectl explain pod                          # 资源编写帮助
kubectl explain pod.spec.containers          # 子字段帮助
kubectl explain pod --api-version=xxx        # 指定 API 版本
kubectl api-versions                         # 支持的 API 版本
kubectl api-resources                        # 支持的 CRD 资源
kubectl explain --api-version=autoscaling/v2beta1 hpa

# 检查 namespace 残留资源
kubectl api-resources -o name --verbs=list --namespaced \
  | xargs -n 1 kubectl get --show-kind --ignore-not-found -n NAMESPACE
```

---

## 7. 标准输入

```bash
cat test.yaml | kubectl apply -f -            # 文件输入
cat <<EOF | kubectl apply -f -                # 命令行输入
...
EOF

# sed 修改后更新
kubectl get deploy test -o yaml | sed 's#xxx#xxx#g' | kubectl replace -f -
```

---

## 8. 流量转发

```bash
kubectl port-forward pod/podname 8080:80     # 本地 8080 → Pod 80
```

---

## 9. 补丁命令 (patch)

### Shell patch

```bash
kubectl patch deploy test -p "$(cat xxx.yaml)"
kubectl patch deploy test -p '{"spec":{"replicas":1}}'
kubectl patch deploy test \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"test","image":"nginx:alpine"}]}}}}'
```

### JSON patch (op/path/value)

```bash
# 添加
kubectl patch deploy test --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/image","value":"nginx:latest"}]'

# 替换
kubectl patch deploy test --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"nginx:latest"}]'

# 删除
kubectl patch deploy test --type='json' \
  -p='[{"op":"remove","path":"/spec/template/spec/containers/0/resources"}]'
```

---

## 10. krew 插件管理

> 官网: https://krew.sigs.k8s.io

```bash
kubectl krew version           # 版本
kubectl krew update            # 更新插件列表
kubectl krew search            # 搜索插件
kubectl krew install access-matrix   # 安装
kubectl krew list              # 已安装列表
kubectl krew uninstall access-matrix # 卸载
```

安装：

```bash
wget https://github.com/kubernetes-sigs/krew/releases/download/v0.4.2/krew-linux_amd64.tar.gz
tar xf krew-linux_amd64.tar.gz
./krew-linux_amd64 install krew
echo 'export PATH="$HOME/.krew/bin/:$PATH"' >> ~/.bashrc
```

---

## 相关笔记

- [[Kubernetes 学习]]
- [[Kubernetes 问题]]
- [[K8s Pod 调度流程面试题]]
- [[etcd 如何保存 Kubernetes 状态]]
