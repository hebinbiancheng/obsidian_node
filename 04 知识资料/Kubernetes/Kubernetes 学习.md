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
  - Kubernetes 学习
---
[kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/)
## kubernetes组件
kubernetes集群由控制平面和一个或多个工作节点组成：
- 控制平面组件
	- kube-api服务器
	- etcd
	- kube-调度程序
	- kube-控制器管理器
- node组件
	- kubelet
	- container runtime
## kubernetes对象
### kubernetes对象管理
创建配置文件中定义的对象：

```shell
kubectl create -f nginx.yaml
```

删除两个配置文件中定义的对象：
```shell
kubectl delete -f nginx.yaml -f redis.yaml
```
通过覆盖活动配置来更新配置文件中定义的对象：
```shell
kubectl replace -f nginx.yaml
```
处理configs目录中的所有对象配置文件，创建并更新活跃对象。可以首先使用diff子命令查看将要进行的更改，如何在进行应用：
```shell
kubectl diff -f configs/
kubectl apply -f configs/
```
递归处理目录：
```sh
kubectl diff -R -f configs/
kubectl apply -R -f configs/
```
### 对象名称和ID
<font color="#9bbb59">名称</font>：集群中每个对象都有一个名称来表示在同类资源中的唯一性
<font color="#9bbb59">UID</font>：每个kubernetes对象也有一个UID来表示在整个集群中的唯一性

> [!NOTE]
> 当对象所代表的是一个物理实体（例如代表一台物理主机的 Node）时， 如果在 Node 对象未被删除并重建的条件下，重新创建了同名的物理主机， 则 Kubernetes 会将新的主机看作是老的主机，这可能会带来某种不一致性。

### 标签和选择算符
标签：是附加到kubernetes对象上的键值对，<font color="#ff0000">标签不支持唯一性</font>
```json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```
标签使我们能够按照指定的任何维度对资源进行切片和切块
```shell
kubectl apply -f examples/guestbook/all-in-one/guestbook-all-in-one.yaml
# 显示的Pod的`app`、`tier`和`role`标签的值
kubectl get pods -Lapp -Ltier -Lrole
```
更新标签：
```shell
kubectl label pods -l app=nginx tier=fe
```
### 名字空间
- 初始名字空间
	- default：kubernetes包含这个名称空间，以便无需创建新的，即可开始使用新集群
	- kube-node-lease：该控件包含用于各个节点关联的lease（租约）对象，节点租约允许kubelet发送心跳，由此控制面能检测到节点故障
	- kube-public：所有的客户端（包括未经身份验证的客户端）都可以读取该名称空间，该名称空间主要预留为集群使用，以便某些资源需要在整个集群中可见可读
	- kube-system：用于kubernetes系统创建的对象
查看名称空间
```shell
kubectl get namespace
```
为请求设置名称空间
```shell
kubectl run nginx --image=nginx --namespace=<名字空间名称>
kubectl get pods --namespace=<名字空间名称>
```
设置名称空间偏好
```shell
kubectl config set-context --current --namespace=<名字空间名称>
# 验证
kubectl config view --minify | grep namespace:
```
#### 名称空间和DNS

> [!警告]
> 
> 通过创建与[公共顶级域名](https://data.iana.org/TLD/tlds-alpha-by-domain.txt)同名的名字空间， 这些名字空间中的服务可以拥有与公共 DNS 记录重叠的、较短的 DNS 名称。 所有名字空间中的负载在执行 DNS 查找时， 如果查找的名称没有[尾部句点](https://datatracker.ietf.org/doc/html/rfc1034#page-8)， 就会被重定向到这些服务上，因此呈现出比公共 DNS 更高的优先序。
> 
> 为了缓解这类问题，需要将创建名字空间的权限授予可信的用户。 如果需要，你可以额外部署第三方的安全控制机制， 例如以[准入 Webhook](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/extensible-admission-controllers/) 的形式，阻止用户创建与公共 [TLD](https://data.iana.org/TLD/tlds-alpha-by-domain.txt) 同名的名字空间。


> [!NOTE] 注
> 并非所有的对象都在名称空间中

查看哪些 Kubernetes 资源在名字空间中，哪些不在名字空间中：
```shell
# 位于名字空间中的资源
kubectl api-resources --namespaced=true

# 不在名字空间中的资源
kubectl api-resources --namespaced=false
```

### 注解
注解：注解中的元数据可以是很小，可以很大，可以是结构化的，也可以是非结构化的，能够包含标签不允许的字符
```json
"metadata": {
  "annotations": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

> [!说明]
> 
> Map 中的键和值必须是字符串。 换句话说，你不能使用数字、布尔值、列表或其他类型的键或值。
### 字段选择符
字段选择符：允许根据一个或者多个资源字段的值筛选kubernetes对象
例如：筛选status.phase字段值为R嬣的所有pod：
```shell
kubectl get pods --field-selector status.phase=Running
```
#### 支持的字段
不同的 Kubernetes 资源类型支持不同的字段选择算符。 所有资源类型都支持 `metadata.name` 和 `metadata.namespace` 字段。
#### 支持的操作符
中使用 `=`、`==` 和 `!=`（`=` 和 `==` 的意义是相同的）操作符
#### 链式选择符
字段选择算符可以通过使用逗号分隔的列表组成一个选择链。
```shell
kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always
```

### Finalizers

> [!NOTE]
> Finalizer 是带有命名空间的键，告诉 Kubernetes 等到特定的条件被满足后， 再完全删除被标记为删除的资源。 Finalizer 提醒[控制器](https://kubernetes.io/zh-cn/docs/concepts/architecture/controller/)清理被删除的对象拥有的资源。

#### Finalizers如何工作：
当你使用清单文件创建资源时，你可以在 `metadata.finalizers` 字段指定 Finalizers。 当你试图删除该资源时，处理删除请求的 API 服务器会注意到 `finalizers` 字段中的值， 并进行以下操作：

- 修改对象，将你开始执行删除的时间添加到 `metadata.deletionTimestamp` 字段。
- 禁止对象被删除，直到其 `metadata.finalizers` 字段内的所有项被删除。
- 返回 `202` 状态码（HTTP "Accepted"）。

管理 finalizer 的控制器注意到对象上发生的更新操作，对象的 `metadata.deletionTimestamp` 被设置，意味着已经请求删除该对象。然后，控制器会试图满足资源的 Finalizers 的条件。 每当一个 Finalizer 的条件被满足时，控制器就会从资源的 `finalizers` 字段中删除该键。 当 `finalizers` 字段为空时，`deletionTimestamp` 字段被设置的对象会被自动删除。 你也可以使用 Finalizers 来阻止删除未被管理的资源。

> [!注]
> - 当你 `DELETE` 一个对象时，Kubernetes 为该对象增加删除时间戳，然后立即开始限制 对这个正处于待删除状态的对象的 `.metadata.finalizers` 字段进行修改。 你可以删除现有的 finalizers （从 `finalizers` 列表删除条目），但你不能添加新的 finalizer。 对象的 `deletionTimestamp` 被设置后也不能修改。
> - 删除请求已被发出之后，你无法复活该对象。唯一的方法是删除它并创建一个新的相似对象。

####  属主引用、标签和 Finalizers
[属主引用](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/owners-dependents/) 描述了 Kubernetes 中对象之间的关系，但它们作用不同。 当一个[控制器](https://kubernetes.io/zh-cn/docs/concepts/architecture/controller/) 管理类似于 Pod 的对象时，它使用标签来跟踪相关对象组的变化。 例如，当 [Job](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/job/) 创建一个或多个 Pod 时， Job 控制器会给这些 Pod 应用上标签，并跟踪集群中的具有相同标签的 Pod 的变化。

Job 控制器还为这些 Pod 添加了“属主引用”，指向创建 Pod 的 Job。 如果你在这些 Pod 运行的时候删除了 Job， Kubernetes 会使用属主引用（而不是标签）来确定集群中哪些 Pod 需要清理。

# kubernetes架构
kubernetes集群由一个控制平面和一组用于运行容器化应用的工作节点（node）组成。每个集群至少需要一个工作系欸但来运行pod。
控制平面组件：
- kube-apiserver：API 服务器是 Kubernetes [控制平面](https://kubernetes.io/zh-cn/docs/reference/glossary/?all=true#term-control-plane)的组件， 该组件负责公开了 Kubernetes API，负责处理接受请求的工作。 API 服务器是 Kubernetes 控制平面的前端。
- etcd
- kube-scheduler：是[控制平面](https://kubernetes.io/zh-cn/docs/reference/glossary/?all=true#term-control-plane)的组件， 负责监视新创建的、未指定运行[节点（node）](https://kubernetes.io/zh-cn/docs/concepts/architecture/nodes/)的 [Pods](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/)， 并选择节点来让 Pod 在上面运行。
- kube-controller-manager：是[控制平面](https://kubernetes.io/zh-cn/docs/reference/glossary/?all=true#term-control-plane)的组件， 负责运行[控制器](https://kubernetes.io/zh-cn/docs/concepts/architecture/controller/)进程。
	- - Node 控制器：负责在节点出现故障时进行通知和响应
	- Job 控制器：监测代表一次性任务的 Job 对象，然后创建 Pod 来运行这些任务直至完成
	- EndpointSlice 控制器：填充 EndpointSlice 对象（以提供 Service 和 Pod 之间的链接）。
	- ServiceAccount 控制器：为新的命名空间创建默认的 ServiceAccount。
- cloud-controller-manager：一个 Kubernetes [控制平面](https://kubernetes.io/zh-cn/docs/reference/glossary/?all=true#term-control-plane)组件， 嵌入了特定于云平台的控制逻辑。允许将你的集群连接到云提供商的 API 之上， 并将与该云平台交互的组件同与你的集群交互的组件分离开来。
节点组件：
- kubelet：会在集群中每个[节点（node）](https://kubernetes.io/zh-cn/docs/concepts/architecture/nodes/)上运行。 它保证[容器（containers）](https://kubernetes.io/zh-cn/docs/concepts/containers/)都运行在 [Pod](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/) 中。
- kube-proxy（可选）：[kube-proxy](https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kube-proxy/) 是集群中每个[节点（node）](https://kubernetes.io/zh-cn/docs/concepts/architecture/nodes/)上所运行的网络代理， 实现 Kubernetes [服务（Service）](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/) 概念的一部分。
- 容器运行时：这个基础组件使 Kubernetes 能够有效运行容器。 它负责管理 Kubernetes 环境中容器的执行和生命周期。
- 插件（Addons）：插件使用 Kubernetes 资源（[DaemonSet](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/daemonset/)、 [Deployment](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/) 等）实现集群功能。 因为这些插件提供集群级别的功能，插件中命名空间域的资源属于 `kube-system` 命名空间。
	- DNS
	- Web界面
	- 容器资源监控
	- 集群层面日志
	- 网络插件
## 节点
### 管理
向API服务器上添加节点的方式主要有两种：
- 节点上的kubelet向控制面执行自注册
- 手动添加一个node对象

> [!NOTE] 说明
> Kubernetes 会一直保存着非法节点对应的对象，并持续检查该节点是否已经变得健康。
> 必须显式地删除该 Node 对象以停止健康检查操作。

### 节点名称唯一性
节点名称用来表示node对象，没有两个node可以使用相同的名称
### 节点自注册
当kubelet标志 --register-node 为true（默认）事，会尝试向api服务注册自己
对于自注册模式，kubelet 使用下列参数启动：
- `--kubeconfig` - 用于向 API 服务器执行身份认证所用的凭据的路径。
- `--cloud-provider` - 与某[云驱动](https://kubernetes.io/zh-cn/docs/reference/glossary/?all=true#term-cloud-provider) 进行通信以读取与自身相关的元数据的方式。
- `--register-node` - 自动向 API 服务器注册。
- `--register-with-taints` - 使用所给的[污点](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/taint-and-toleration/)列表 （逗号分隔的 `<key>=<value>:<effect>`）注册节点。当 `register-node` 为 false 时无效。

- `--node-ip` - 可选的以英文逗号隔开的节点 IP 地址列表。你只能为每个地址簇指定一个地址。 例如在单协议栈 IPv4 集群中，需要将此值设置为 kubelet 应使用的节点 IPv4 地址。 参阅[配置 IPv4/IPv6 双协议栈](https://kubernetes.io/zh-cn/docs/concepts/services-networking/dual-stack/#configure-ipv4-ipv6-dual-stack)了解运行双协议栈集群的详情。
    
    如果你未提供这个参数，kubelet 将使用节点默认的 IPv4 地址（如果有）； 如果节点没有 IPv4 地址，则 kubelet 使用节点的默认 IPv6 地址。
    

- `--node-labels` - 在集群中注册节点时要添加的[标签](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/)。 （参见 [NodeRestriction 准入控制插件](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#noderestriction)所实施的标签限制）。
- `--node-status-update-frequency` - 指定 kubelet 向 API 服务器发送其节点状态的频率。

### 节点控制器
节点控制器在节点的生命周期中扮演多个角色：
- 当节点注册时为他分配一个CIDR区段（开启cidr分配）
- 保持节点控制器内的节点列表与云服务所提供的可用机器列表同步
- 监控节点的健康状况（默认每5秒检查一次节点状态）

### 逐出速率限制
节点控制器把逐出速率（--node-eviction-rate）在每秒0.1（默认）个，表示每10s至多从一个节点驱逐pod
当一个可用区域（Availability Zone）中的节点变为不健康时，节点的驱逐行为将发生改变。
- 如果不健康节点的比例超过 `--unhealthy-zone-threshold`（默认为 0.55）， 驱逐速率将会降低。
- 如果集群较小（意即小于等于 `--large-cluster-size-threshold` 个节点 - 默认为 50）， 驱逐操作将会停止。
- 否则驱逐速率将降为每秒 `--secondary-node-eviction-rate` 个（默认为 0.01）。

### 资源容量跟踪
node对象会跟踪节点上资源的容量（如可用内存和cpu数量）
kubernetes调度器保证节点上有足够的资源供其上的所有pod使用，他会检查节点上所有容器的请求的综合不会超过节点的容量

## 节点与控制器之间的通信
### 节点到控制器

> [!NOTE]
> kubernetes采用的是中心辐射型api模式：所有从节点（或运行于其上的pod）发出的API调用都终止于API服务器。API服务器被配置在一个安全的HTTPS端口（通常为443）上监听远程连接请求，并启用一个或多个形式的客户端身份认证机制。

### 控制器到节点
从控制器到节点有两个主要的通信路径：
- 从API服务器到集群中每个节点上运行的kubelet进程
- 从API服务器通过他的代理功能连接到任何节点，pod或者服务
#### API服务器到kubelet
这种链接终止于kubelet的HTTPS末端，默认情况下，API服务器不检查kubelet的服务证书，所以这种连接容易收到中间人的攻击，是不安全的
#### API服务器到节点，pod和服务
默认是纯HTTP方式，因此既没有认证，也灭有加密，可以运行在HTTPS连接上，不过也不会雁阵给HTTPS末端提供的证书，也不会提供客户端证书，因此，虽然连接是加密的，仍无法提供任何完整性保证

#### ssh隧道
kubernetes支持使用ssh隧道来保护从控制器到节点的通信路径，在这种配置下，API服务器建立一个到集群中各节点的ssh隧道，并通过这个隧道传输所有到kubelet，节点，pod或服务的请求。
#### konnectivity服务
作为ssh隧道的替代方案，提供tcp的代理，以便支持从控制器到集群的通信

## 控制器
在kubernetes中，控制器通过监控集群内的公共状态，并致力于将当前状态转为期望的状态
### 控制器模式
一个控制器至少追踪一种类型的kubernetes资源，这些对象有一个代表期望状态的spec字段。该资源的控制器负责确保器当前状态接近期望状态
#### <font color="#ffff00">通过API服务器来控制</font>
job控制器是一个kubernetes内置的控制器的例子，内置控制器通过和集群API服务器交互来管理状态
job是一种kubernetes资源，运行一个或者多个pod，来执行一个任务然后停止

在集群中，当job控制器拿到新任务时，会保证一组node节点上的kubelet可以运行正确数量的pod来完成工作。

> [!NOTE]
> job控制器不会自己运行任何pod或者容器，job控制器是通知api服务器来创建或者移除pod

#### 直接控制
相比于job控制器，有些控制器需要对集群外的一些东西进行修改
例如：如果你使用一个控制回路来保证集群中有足够的节点，没那么控制器就许哟啊当前集群外的一些服务在需要时创建新的节点


> [!NOTE] 说明：
>可以有多个控制器来创建或者更新相同类型的对象。 在后台，Kubernetes 控制器确保它们只关心与其控制资源相关联的资源。
>例如，你可以创建 Deployment 和 Job；它们都可以创建 Pod。 Job 控制器不会删除 Deployment 所创建的 Pod，因为有信息 （[标签](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/)）让控制器可以区分这些 Pod。


## 租约

> [!NOTE] 租约
> 租约提供了一种机制来锁定共享资源并协调集合成员之间的活动。
> 在kubernetes中，租约表示为：coordination.k8s.io中的Lease对象


> [!NOTE] 节点心跳
> kubernetes使用Lease API将kubelet节点心跳传递到kubernetes API服务器。对于每个node，在kube-node-lease名称空间中都有一个具有匹配名称的lease对象。
>  在此基础上，每个 kubelet 心跳都是对该 `Lease` 对象的 **update** 请求，更新该 Lease 的 `spec.renewTime` 字段。 Kubernetes 控制平面使用此字段的时间戳来确定此 `Node` 的可用性。


> [!NOTE] 领导者选举
> Kubernetes 也使用 Lease 确保在任何给定时间某个组件只有一个实例在运行。 这在高可用配置中由 `kube-controller-manager` 和 `kube-scheduler` 等控制平面组件进行使用， 这些组件只应有一个实例激活运行，而其他实例待机。

## 垃圾收集

> [!NOTE] 垃圾收集
> 垃圾收集时kubernetes用于清理集群资源的各种机制的同城，允许系统清理如下资源
> 1. 终止的pod
> 2. 已完成的job
> 3. 不再存在属主引用的对象
> 4. 未使用的容器和容器镜像
> 5. 动态制备的，storageClass回收策略为delete的pv卷
> 6. 阻滞或者过期的certificateSigningRequest（CSR）
> 7. 节点租约对象


> [!NOTE] 级联删除
> kubernetes会检查并删除那些不再拥有属主引用的对象
> 例如：删除了ReplicaSet之后留下的pod，当删除某个对象时，可以Jon告知kubernetes是否去自动删除该对象的依赖对象
> 级联删除：
> - 前台级联删除
> - 后台级联删除


> [!NOTE] 前台级联删除
> 在前台级联删除中，正在被你删除的属主对象首先进入 **deletion in progress** 状态。 在这种状态下，针对属主对象会发生以下事情：
> - Kubernetes API 服务器将某对象的 `metadata.deletionTimestamp` 字段设置为该对象被标记为要删除的时间点。
> - Kubernetes API 服务器也会将 `metadata.finalizers` 字段设置为 `foregroundDeletion`。
> - 在删除过程完成之前，通过 Kubernetes API 仍然可以看到该对象。
>
当属主对象进入**删除进行中**状态后，控制器会删除其已知的依赖对象。 在删除所有已知的依赖对象后，控制器会删除属主对象。 这时，通过 Kubernetes API 就无法再看到该对象。


> [!NOTE] 后台级联删除
> 在后台级联删除过程中，Kubernetes 服务器立即删除属主对象， 而垃圾收集控制器（无论是自定义的还是默认的）在后台清理所有依赖对象。 如果存在 Finalizers，它会确保所有必要的清理任务完成后对象才被删除。 <font color="#ffff00">默认情况下，</font>Kubernetes 使用后台级联删除方案，除非你手动设置了要使用前台删除， 或者选择遗弃依赖对象。

## 关联笔记

- [[04 知识资料/知识库总索引|知识库总索引]]
- [[04 知识资料/Kubernetes/Kubernetes 知识库索引|Kubernetes 知识库索引]]

