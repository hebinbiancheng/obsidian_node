---
title: kubeadm 集群证书更新实战：从 PKI 原理、HA 架构到生产级自动化治理
date: 2026-07-03
tags: [kubernetes, kubeadm, 证书, pki, tls, etcd, ha, 运维自动化, prometheus]
category: 知识资料
source: https://mp.weixin.qq.com/s/iw0JxW_yqYWCR1ZXdCMxrg
---

# kubeadm 集群证书更新实战

> 原文: [微信文章](https://mp.weixin.qq.com/s/iw0JxW_yqYWCR1ZXdCMxrg)

---

## 一、核心结论

证书更新从来不是一条命令的问题。`kubeadm certs renew all` 只完成了"重新签发证书"，真正难题是让整个控制面有序切换到新凭据。

**五个关键问题**：
1. 是否理解 kubeadm 控制面的 PKI 拓扑
2. 续签后为什么 `kubectl` 仍可能失败
3. 多控制面集群如何逐节点、逐组件、可回滚地滚动更新
4. 是否有提前预警、自动备份、失败熔断和事后审计能力
5. 是否从"运维动作"升级成"平台治理能力"

---

## 二、kubeadm PKI 拓扑

```
Cluster Root CA
├── apiserver.crt / apiserver.key
├── apiserver-kubelet-client.crt / key
├── controller-manager.conf         (内嵌 client cert)
├── scheduler.conf                  (内嵌 client cert)
├── admin.conf                      (内嵌 client cert)
├── kubelet.conf                    (部分场景会被轮换/重建)
├── front-proxy-ca.crt / key
│   └── front-proxy-client.crt / key
└── etcd-ca.crt / key
    ├── etcd/server.crt / key
    ├── etcd/peer.crt / key
    ├── etcd/healthcheck-client.crt / key
    └── apiserver-etcd-client.crt / key

ServiceAccount KeyPair
├── sa.key
└── sa.pub
```

**三个容易混淆的点**：
- `/etc/kubernetes/pki/*.crt/*.key` 是原始证书与私钥文件
- `/etc/kubernetes/*.conf` 是 kubeconfig，内嵌 base64 客户端证书
- `sa.key/sa.pub` 是 ServiceAccount Token 签名密钥，不属于 X.509 证书

**所以续签后 kubectl 仍异常 — 根因是 kubeconfig 里的嵌入式证书没更新。**

---

## 三、续签后为什么要重启

控制面组件以静态 Pod 方式运行（`/etc/kubernetes/manifests/`）。进程启动时读取证书到内存，宿主机文件被原地替换后**不会自动热更新**。

正确链路：`生成新证书 → 更新 kubeconfig → 重启组件 → 重新读取 → 健康检查`

---

## 四、单控制面 vs 多控制面策略

| | 单控制面 | 多控制面 HA |
|------|:--:|:--:|
| 复杂度 | 低 | 高 |
| 风险窗口 | 维护期间控制面不可用 | 逐节点滚动，控制面始终可用 |
| 核心策略 | 短时维护窗口 | 严格滚动纪律 |

---

## 五、标准生产更新 Runbook

### 5.1 更新前检查
```bash
kubeadm certs check-expiration
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide
etcdctl endpoint health --cluster
```

### 5.2 备份
- `/etc/kubernetes/pki`
- `/etc/kubernetes/*.conf`
- `/etc/kubernetes/manifests`
- etcd 快照

```bash
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  --endpoints=https://127.0.0.1:2379 \
  snapshot save /backup/etcd-snapshot-$(date +%Y%m%d%H%M%S).db
```

### 5.3 续签
```bash
kubeadm certs renew all

# 或按组件续签
kubeadm certs renew apiserver
kubeadm certs renew apiserver-etcd-client
kubeadm certs renew front-proxy-client
```

### 5.4 刷新 kubeconfig
```bash
kubeadm init phase kubeconfig admin
kubeadm init phase kubeconfig controller-manager
kubeadm init phase kubeconfig scheduler
kubeadm init phase kubeconfig kubelet --node-name "$(hostname)"
```

### 5.5 按依赖顺序重启
```
etcd → kube-apiserver → kube-controller-manager → kube-scheduler → kubelet
```

### 5.6 验证（四层）
1. **文件层**：证书到期时间是否已刷新
2. **进程层**：静态 Pod 是否 Running
3. **接口层**：`https://127.0.0.1:6443/healthz` 是否可用
4. **集群层**：`kubectl get nodes/pods` 是否正常

---

## 六、HA 集群关键要点

1. **从 LB 摘流** → 执行更新 → 健康验证 → 重新加入 LB
2. **绝不能并发更新两个控制面节点** — 会同时失去控制面多数可用性和 etcd 仲裁安全边界
3. **一次只动一台**，当前节点完全恢复后才处理下一台
4. stacked etcd 场景要特别注意 peer 证书一致性

---

## 七、平台化架构升级

从"脚本"升级到"平台能力"：

```
Prometheus / Alertmanager
       |
       v
Certificate Governance Controller
       |
       +--> Inventory & Expiry Scanner
       +--> Renewal Orchestrator
       +--> Backup Executor
       +--> LB Drain / Recover Adapter
       +--> Validation & Rollback Engine
       +--> Audit Event Publisher
```

**关键能力**：
- 证书扫描器：周期性扫描，输出统一指标
- 更新编排器：判断窗口、触发逐节点续签、管理批次
- 备份执行器：文件级备份 + etcd 快照
- 验证与回滚引擎：健康检查 + 超时自动中止 + 证书恢复

---

## 八、Go 控制器样例

采用"控制器编排 + 节点执行器"架构：

```yaml
apiVersion: platform.example.io/v1alpha1
kind: CertificateRotation
spec:
  clusterName: prod-k8s-01
  renewBeforeDays: 45
  maxUnavailableControlPlanes: 1
  targetNodes:
    - master-1
    - master-2
    - master-3
  strategy:
    type: RollingUpdate
    requireLoadBalancerDrain: true
    requireEtcdSnapshot: true
```

Controller 核心逻辑：单飞锁 → 选下一个节点 → 健康检查 → 创建 NodeRotationTask → 等待 → 下一台。

---

## 九、Prometheus 指标与告警

### 指标
- `kubeadm_cert_expiry_days{cert_name,node}` — 证书剩余天数
- `kubeadm_cert_rotation_success_total{cluster,node}`
- `kubeadm_cert_rotation_failure_total{cluster,node}`
- `kubeadm_cert_rotation_duration_seconds{cluster,node}`

### 告警规则
- `KubeadmCertificateExpiringSoon`：剩余 < 30 天，severity: warning
- `KubeadmCertificateExpired`：剩余 ≤ 0，severity: critical

---

## 十、常见坑位

| 坑 | 根因 | 修复 |
|------|------|------|
| 证书更新后 kubectl 仍报错 | admin.conf 没更新 | `kubeadm init phase kubeconfig admin` |
| apiserver 起不来 | etcd 客户端证书链不匹配 / apiserver 先于 etcd 恢复 | 检查 apiserver-etcd-client.crt，先恢复 etcd |
| 只更新了一台控制面 | HA 集群未逐节点完整执行 | 立即补齐剩余节点 |
| 误删原始证书 | 无备份 | 用备份恢复 pki 和 kubeconfig，必要时恢复 etcd 快照 |
| 外部 etcd 被忽略 | external etcd 不在同一运维域 | Kubernetes 控制面证书 + 外部 etcd 证书分开治理 |

---

## 十一、回滚策略

三层回滚能力：
1. **文件级回滚**：恢复 `pki` + `*.conf` + 静态 Pod Manifest → 重启 kubelet
2. **etcd 数据级回滚**：恢复变更前 etcd 快照
3. **批次级熔断**：HA 集群逐台更新时某台失败 → 停止后续节点 → 人工处置

---

## 十二、最小执行命令清单

```bash
# 1. 查看证书到期时间
kubeadm certs check-expiration

# 2. 备份
cp -R /etc/kubernetes /backup/kubernetes-$(date +%Y%m%d%H%M%S)

# 3. 续签
kubeadm certs renew all

# 4. 刷新 kubeconfig
kubeadm init phase kubeconfig admin
kubeadm init phase kubeconfig controller-manager
kubeadm init phase kubeconfig scheduler
kubeadm init phase kubeconfig kubelet --node-name "$(hostname)"

# 5. 重启 kubelet
systemctl restart kubelet

# 6. 验证
KUBECONFIG=/etc/kubernetes/admin.conf kubectl get nodes
curl -k https://127.0.0.1:6443/healthz
```

---

## 最终建议

从"更新证书"升级到"治理证书"：

| 阶段 | 做法 |
|------|------|
| 脚本 | 解决一次问题 |
| Runbook | 固化流程 |
| 控制器 + 平台 | 解决长期治理 |

核心思路：让证书治理成为一项**稳定、低风险、可审计、可演进**的平台能力。

---

## 插图

![01](images/kubeadm_cert_01.jpg)

![02](images/kubeadm_cert_02.jpg)

## 相关笔记

- [[K8s控制面故障应急响应复盘]]
- [[K8s Deployment 实战指南]]
- [[K8s PVC 绑定 PV 全过程]]
