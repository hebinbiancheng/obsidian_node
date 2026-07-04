---
title: K8s PVC 绑定 PV 全过程详解
date: 2026-07-02
tags: [kubernetes, k8s, storage, pvc, pv, storageclass, csi]
category: 知识资料
source: https://mp.weixin.qq.com/s/efB1vlHvk4uJvOrHyswC9A
---

# Kubernetes 存储：PVC 绑定 PV 全过程

> 原文: [微信文章](https://mp.weixin.qq.com/s/efB1vlHvk4uJvOrHyswC9A)

---

## 整体核心流程

```
用户创建 PVC → 集群检索可用存储 → 有现成 PV 直接绑定 / 无则自动新建 → 双向绑定成功 → 容器挂载使用
```

---

## 4 个核心角色

| 角色 | 职责 | 类比 |
|------|------|------|
| **PVC** | 用户提需求（容量、读写权限） | 存储申请单 |
| **PV** | 集群中真实的存储资源 | 真实磁盘 |
| **StorageClass** | 存储配置规则（类型、区域、绑定方式） | 存储模板 |
| **CSI** | 对接底层存储的插件 | 存储对接工具 |

---

## 6 步绑定全过程

### 步骤 1：用户提交存储申请

用户创建 PVC（YAML），集群记录并标记为 **Pending**。

> 此时只有申请，没有真实磁盘。

### 步骤 2：集群控制器自动巡查

K8s 存储控制器持续监控 PVC，发现新申请后：

- 遍历集群找符合要求的空闲 PV
- 找到 → 准备绑定
- 找不到 → 触发自动建盘
- 完全匹配不上 → 保持 Pending

### 步骤 3：智能分流（静态 vs 动态）

#### 场景 A：静态供给（用现成盘）

满足条件即可直接分配：
- 磁盘容量够大
- 读写权限一致
- 存储类型匹配
- 磁盘空闲（未被占用）

#### 场景 B：动态供给（自动建新盘）

无合适 PV + 配置了 StorageClass → 触发自动建盘。

### 步骤 4：自动创建真实磁盘

```
集群根据 StorageClass → 调用 CSI 插件 → 底层创建磁盘 → 自动生成 PV
```

> 绑定分配 = 集群控制器干活；创建磁盘 = 存储插件干活。两者互不干涉。

### 步骤 5：双向绑定，锁死资源

最终校验通过后：
- PVC 绑定指定 PV
- PV 反向绑定 PVC

> 磁盘从此只属于这个 PVC，不会被抢占。

### 步骤 6：状态就绪

| 资源 | 旧状态 | 新状态 |
|------|--------|--------|
| PVC | Pending | **Bound** |
| PV | Available | **Bound** |

---

## 延迟绑定模式（生产必备）

### 普通模式问题

创建 PVC 就立刻建盘绑定 → 容器调度节点可能与磁盘不在同一可用区 → 挂载失败。

### 延迟绑定（`WaitForFirstConsumer`）

```
创建 PVC → 等待容器调度确定节点 → 根据节点区域创建磁盘 → 再绑定
```

**完美解决**：
- 磁盘和节点不在同一可用区
- 跨区域挂载失败、权限异常

---

## 总流程图

```
创建 PVC
  ↓
集群控制器检测
  ↓
有合适空闲 PV？
  ├── 有 → 直接双向绑定
  └── 无 → 自动建盘
       ↓
  CSI 创建真实磁盘
       ↓
  自动生成 PV
       ↓
  PVC ↔ PV 双向绑定
       ↓
  状态就绪，容器挂载
```

---

## 常见故障排查

### 1. PVC 一直 Pending

- 没有合适空闲 PV，又没配置 StorageClass
- CSI 插件异常，建盘失败
- StorageClass 配置错误
- 延迟绑定模式下，节点与磁盘区域不匹配

### 2. 有 PV 但绑不上

- 容量不够 / 读写权限不匹配
- 存储类型不一致
- PV 有专属筛选规则

### 3. PV 存在但无法使用

- 已被其他 PVC 占用
- 状态异常（释放失败、创建失败）
- 区域策略限制

---

## 排查思路

顺着 `申请 → 匹配 → 建盘 → 绑定 → 就绪` 链路梳理，绝大多数挂载异常都能快速定位。

---

## 生产实战：PV/PVC/StorageClass 配置与排坑

> 补充来源: [微信文章](https://mp.weixin.qq.com/s/znAwPQnxF_tXEP8xMwXfjg)

### PV 生命周期

| 状态 | 含义 |
|------|------|
| Available | 空闲待用 |
| Bound | 已被 PVC 绑定 |
| Released | PVC 已删，PV 待清理 |
| Failed | 回收失败 |

**回收策略**：

| 策略 | 行为 | 注意 |
|------|------|------|
| Retain | 保留数据，手动清理 | 生产推荐 |
| Delete | PVC 删除时联动删除 PV 和底层存储 | 云环境常用 |
| Recycle | ~~rm -rf 清理~~ | ⚠️ v1.20+ 已废弃 |

### accessModes 三种模式

| 模式 | 含义 | 场景 |
|------|------|------|
| ReadWriteOnce (RWO) | 单节点读写 | 云盘、Local PV |
| ReadOnlyMany (ROX) | 多节点只读 | 共享配置 |
| ReadWriteMany (RWX) | 多节点读写 | NFS、CephFS |

---

### 静态供给实战

**创建 PV**：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-static-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.1.100
    path: /data/nfs-share
```

**创建 PVC**：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-static-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi          # 必须 ≤ PV 容量
```

**Pod 挂载**：

```yaml
volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-static-pvc
containers:
  - volumeMounts:
      - name: storage
        mountPath: /data
```

---

### 动态供给实战

**云环境 StorageClass（AWS EBS）**：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer   # 避免跨可用区挂载失败
allowVolumeExpansion: true                 # 允许在线扩容
```

**自建 NFS 动态供给**：

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=192.168.1.100 \
  --set nfs.path=/data/nfs-k8s
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  pathPattern: "{.PVC.name}"
  onDelete: delete
```

**Local SSD（数据库高性能场景）**：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-ssd
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer   # 必须！

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-node1
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-ssd
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:                           # 必须指定节点
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node1
```

> NVMe SSD Local Volume 单盘可提供 50 万+ IOPS。但节点故障时数据不可用。

---

### PVC 排障三板斧

**第一步：看事件**

```bash
kubectl describe pvc <name> -n <ns>
```

**第二步：检查 StorageClass**

```bash
kubectl get sc                    # 确认 storageClassName 存在
```

**第三步：检查动态供给器**

```bash
kubectl get pods -n kube-system | grep -E "provisioner|csi"
kubectl logs -n kube-system <provisioner-pod>
```

**隐藏坑：容量不足**

```bash
kubectl get resourcequota -n <ns>   # ResourceQuota 限制？
```

---

### PVC Terminating 卡住

**原因 1：还有 Pod 在使用**

```bash
kubectl get pods --all-namespaces -o json | \
  jq '.items[] | select(.spec.volumes[].persistentVolumeClaim.claimName=="your-pvc")'
```

**原因 2：Finalizers 卡住**

```bash
# ⚠️ 谨慎！确保无业务使用
kubectl patch pvc <name> -p '{"metadata":{"finalizers":null}}' --type=merge
```

---

### StorageClass 迁移

推荐工具 [pvmigrate](https://github.com/utkuozdemir/pvmigrate)：

```bash
pvmigrate --source-sc default --dest-sc fast-ssd
```

流程：验证 SC → 找 PVC → 创建新 PVC → 停 Pod → rsync 数据 → 切换关联 → 恢复 Pod。

> 局限：只支持 StatefulSet 和 Deployment，建议维护窗口操作。

---

## 相关笔记

- [[Kubernetes 学习]]
- [[K8s Pod 调度流程面试题]]
