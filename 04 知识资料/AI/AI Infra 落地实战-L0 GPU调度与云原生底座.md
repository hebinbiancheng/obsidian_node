---
title: AI Infra 落地实战 - L0 GPU调度与云原生底座
date: 2026-07-05
tags: [AI, Infra, GPU调度, K8s, Volcano, Kueue, DRA, 存储, 网络]
category: 知识资料
source: https://mp.weixin.qq.com/s/ZG9h3V76jtqEZLmCPIo0qw
author: Knock
---

# AI Infra 落地实战 · L0 基础资源层

> 系列文章 [[AI Infra 落地实战]] 第 1 篇 · GPU 调度与云原生底座
> 算力调度、存储分层、网络低延迟 — 地基不稳，上面八层都是空中楼阁。

---
### 一、开篇场景：一个真实问题

某公司咬牙买了 8 张 H100，总价小几百万，推理服务上线后却发现 GPU 利用率只有 30%。月账单 $12 万，CTO 在周会上拍桌子：「两周内给我提到 70%，否则项目砍掉。」

团队一查，问题比想象中更糟：训练任务和推理任务抢同一批 GPU，调度全靠手动 kubectl，谁嗓门大谁先跑；推理服务凌晨没流量但 GPU 照常占着，空跑烧钱；模型权重散落在各个节点的本地磁盘，版本对不上导致推理结果飘移；运维半夜被告警叫起来手动扩容，团队疲惫不堪。

这不是个例。2026 年的行业调研数据显示，企业 GPU 集群平均利用率仅在 30%–50% 之间，大量算力在空转。H100 每小时 $2–4，B200 on-demand 更是 $6.03/hr——每一台空跑的 GPU 都在烧钱。问题的根源不在 GPU 本身，而在底座：调度、存储、网络这三件事没做好。这就是 L0 基础资源层要解决的核心命题。

### 二、这一层到底解决什么

AI 系统跑在哪里、算力怎么调度、存储怎么分层、网络怎么低延迟——L0 是整个 AI Infra 的地基。

具体来说，L0 解决四个核心问题：

**算力供给：** GPU 是稀缺且昂贵的资源。2026 年 NVIDIA B200 Blackwell 单卡 192GB HBM3e、8.0 TB/s 显存带宽、9,000 TFLOPS FP4 算力，单卡 TDP 1,000W，on-demand 价格 $6/hr 起。GB200 NVL72 机架更是 72 张 B200 组成的超级节点，总算力超过 13 PFLOPS。买回来只是第一步，怎么把每一张卡的算力榨干，才是 L0 的核心价值。

**调度效率：** 训练任务是「gang scheduling」——一组 Pod 必须同时启动，差一个就全部等待。推理任务需要按延迟 SLA 分池。默认 Kubernetes 调度器为无状态微服务设计，对 AI 工作负载几乎完全不对路。2026 年 Kubernetes Dynamic Resource Allocation（DRA）已正式 GA，取代了静态 Device Plugin，成为 GPU 资源分配的新标准。但 DRA 只是基础，上面还需要 Volcano、Kueue 等批处理调度器才能实现公平调度和抢占。

**存储分层：** 模型权重动辄几十 GB，训练数据 TB 起步。全放 NVMe 太贵，全放对象存储太慢。L0 需要构建热（NVMe 本地缓存）→温（分布式文件系统如 JuiceFS）→冷（对象存储归档）的三级存储体系，在成本和性能之间找到平衡点。

**网络低延迟：** 分布式训练时 GPU 间梯度同步产生海量通信流量。用标准以太网，网络成为瓶颈，多卡性能甚至不如单卡。RDMA/InfiniBand 绕过内核直接内存到内存传输，GPU 间通信延迟压到 1ms 以内。NVLink 5.0 在 B200 上提供 1.8 TB/s 双向带宽，节点内 GPU 间通信几乎无瓶颈。

L0 不稳，上面 L1 模型推理层、L2 数据知识层、L4 Agent 编排层全都是空中楼阁。

### 三、核心架构图

![架构图](assets/ai-infra-l0-1.png)

L0 架构自下而上分为五层，每层解决一个维度的资源供给问题：

**计算层（GPU/CPU）：** 最底层的物理资源。2026 年主流算力卡包括 NVIDIA H100（80GB HBM3, 3.35 TB/s）、H200（141GB HBM3e）、B200（192GB HBM3e, 8.0 TB/s, 原生 FP4 支持）以及 GB200 超级芯片（Grace CPU + Blackwell GPU, 统一内存架构最高 13.4TB）。B200 相比 H100 训练吞吐量约 2 倍提升，显存带宽 2.4 倍——192GB 显存让 70B 模型 FP16 单卡即可装载，不再需要张量并行切分。国产侧有昇腾 910C、摩尔线程 S5000 等替代方案。

**编排层（K8s/Ray/Slurm）：** 计算资源的管理者。Kubernetes + Volcano 是推理服务的首选组合，DRA 已 GA，GPU 调度从「能用」走向「好用」；Ray Cluster 适合训练与推理统一场景，Python 原生 Actor 模型，Ray 2.55 版本在 KubeRay 管理下可实现 GPU 利用率提升 50%–70%；Slurm 在纯 HPC 训练集群中多租户公平调度仍然最强。

**存储层（对象/块/文件）：** 对象存储（S3/OSS）做冷数据底座，JuiceFS 做 POSIX 兼容的分布式文件系统层——韩国搜索引擎巨头 NAVER 在对比 Alluxio 后选择了 JuiceFS，BioMap 用 JuiceFS 将模型存储成本降低 90%。热数据放 NVMe 本地缓存，模型权重在推理节点本地预加载。

**网络层（RDMA/IB/VPC）：** 节点内用 NVLink（B200 上 1.8 TB/s），节点间用 InfiniBand HDR（200 Gb/s/链路）或 RoCE v2（RDMA over Converged Ethernet）。NVIDIA Spectrum-X 以太网方案在超大规模下可将 RoCE 性能逼近 InfiniBand，成本更低。NCCL 库自动选择最优通信路径。

**安全层（密钥/隔离）：** MIG 物理隔离、网络策略、密钥管理（Vault/KMS），保障多租户环境下的资源隔离与数据安全。

五层环环相扣，任何一层成为短板都会拖垮整个系统的效率。

### 四、关键技术拆解

![技术拆解](assets/ai-infra-l0-2.png)

L0 的核心技术可以拆解为四大模块：GPU 虚拟化、弹性伸缩、存储分层、RDMA 高速互联。下面逐一展开。

#### 4.1 GPU 虚拟化：MIG、时间分片与 MPS

GPU 是单体资源，但不同工作负载需要的算力粒度不同。一个 7B 模型推理可能只需要 10GB 显存，而 B200 有 192GB——不切分就是浪费。GPU 虚拟化有三种主流方案：

**MIG（Multi-Instance GPU）——物理硬切分。** NVIDIA 在 Hopper、Blackwell、Rubin 架构上支持 MIG，可将单张 GPU 最多切分为 7 个完全隔离的实例，每个实例有独立的计算核心、显存和缓存，提供确定性 QoS。以 GB200 为例，管理员可以创建 2 个 93GB 实例、4 个 46GB 实例或 7 个 23GB 实例。MIG 的最大优势是硬件级隔离——一个实例的负载不会影响另一个实例的性能，非常适合多团队共享 GPU。白天开 7 个 MIG 实例跑低吞吐推理，晚上重配为 1 个大实例跑深度学习训练，资源利用率最大化。Kubernetes 中通过 NVIDIA GPU Operator（v1.9.0+）和 MIG Manager 管理，支持 single 和 mixed 两种策略。

**时间分片（Time-Slicing）——软件虚拟化。** 多个 Pod 共享同一张 GPU，按时间片轮流使用。实现简单（NVIDIA Device Plugin 内置支持），但无 QoS 保障——一个 Pod 的显存带宽暴涨会饿死其他 Pod。适合开发测试环境或对延迟不敏感的批量推理。

**MPS（Multi-Process Service）——共享计算。** 多个进程共享 GPU 的计算资源但各自有独立地址空间。比时间分片性能更好（真正并行而非轮流），但隔离性不如 MIG，一个进程崩溃可能影响其他进程。

选型建议：生产环境多租户用 MIG（硬件隔离 + QoS 保障），开发测试用时间分片（简单够用），性能敏感但可接受弱隔离的场景用 MPS。

#### 4.2 弹性伸缩：按队列深度与 GPU 利用率自动扩缩容

推理服务的流量有明显波峰波谷。白天高峰需要 16 个推理副本，凌晨低谷 2 个就够。手动扩缩容反应慢、易出错，自动弹性伸缩是必需品。

弹性伸缩的核心逻辑是双指标驱动：请求队列深度（反映需求积压）和 GPU 利用率（反映资源饱和度）。队列深但 GPU 利用率低，说明单卡处理能力不足，需要扩容；队列浅且 GPU 利用率低，说明资源过剩，可以缩容。还要加入空跑检测——GPU 被占用但没有实际计算，这种情况立即缩容到最小副本数。

```python
def auto_scale_gpu(queue_depth, gpu_util):
    if queue_depth > 100 and gpu_util > 0.85:
        replicas = scale_up(min=2, max=16, step=2)
        notify("GPU 扩容 → %d 副本" % replicas)
    elif queue_depth < 10 and gpu_util < 0.3:
        scale_down(step=1, min=2)
    if idle_minutes > 5:  # 空跑检测
        scale_to(min_replicas)
    record_metric("queue_depth", queue_depth)
    record_metric("gpu_util", gpu_util)
```

Kubernetes 生态中，KEDA 支持基于自定义指标（如 Prometheus 中的队列深度）的 HPA 扩缩容。Cluster Autoscaler 负责节点层面扩缩容，关键参数 `scale-down-unneeded-time` 建议推理场景设 10 分钟、训练场景设 30 分钟——太短会频繁扩缩导致抖动，太长则空跑烧钱。合理的弹性伸缩策略可降低 20%–40% 的云账单。

#### 4.3 多租户公平调度：Volcano 与 Kueue

多个团队共享 GPU 集群时，「谁先来谁先用」的 FIFO 策略会导致大团队垄断资源、小团队永远排队。公平调度的目标是按权重分配 GPU 配额，高优先级任务可抢占低优先级任务。

```python
def schedule_gpu_task(jobs):
    for tenant, weight in fair_share_weights.items():
        tenant_jobs = [j for j in jobs if j.tenant == tenant]
        quota = total_gpus * weight
        allocate(tenant_jobs[:quota], priority="fair")
    preempt(low_priority_running,
            if=high_priority_pending)
    log_schedule_decision(jobs, fair_share_weights)
```

Volcano 是 CNCF 孵化的批处理调度器，核心能力是 gang scheduling——一组 Pod 要么全部调度成功，要么全部等待，避免训练任务部分启动后卡死。Kueue 是 Kubernetes 原生的队列管理组件，配合 Volcano 使用，支持多团队公平共享、优先级抢占和配额管理。2026 年 Kueue + Volcano + DRA 已成为 K8s 上 AI 工作负载调度的标准组合，合理配置的调度器可将集群利用率从默认的 40% 提升到 75%。

#### 4.4 存储分层与 RDMA 高速互联

**存储三级分层：** 热——NVMe SSD 本地缓存，模型权重预加载到推理节点本地，读取延迟 <1ms；温——JuiceFS 分布式文件系统，S3 后端 + POSIX 接口 + Kubernetes CSI Driver，支持 ReadWriteMany 多节点并发访问，NAVER 和 BioMap 的实践证明其在大规模 AI 场景下兼具性能与成本优势；冷——对象存储归档，训练数据和历史版本长期保存。Alluxio 定位为数据编排与热数据缓存层，可跨存储加速，但在 S3/FUSE 兼容性上不如 JuiceFS，更适合作为缓存中间层而非主存储。

**RDMA 高速互联：** 分布式训练中 GPU 间梯度同步通信量巨大，标准 TCP/Ethernet 要经过内核协议栈，延迟高、吞吐低。RDMA 绕过操作系统内核，实现 GPU 显存到显存的直接传输。InfiniBand HDR 单链路 200 Gb/s，NVIDIA Spectrum-X 以太网方案在 RoCE v2 上实现接近 IB 的性能，成本更低。节点内 NVLink 5.0 在 B200 上提供 1.8 TB/s 双向带宽，是 PCIe 的数十倍。NCCL 库自动感知网络拓扑，选择最优通信路径，正确设置 `NCCL_SOCKET_IFNAME` 和 `NCCL_IB_DISABLE` 等环境变量至关重要，否则多节点通信性能可能差 10–100 倍。微软在 Azure 上部署的 4,608 张 GPU NVLink 集群实现了 92.1 exaFLOPS FP4 推理算力，印证了高速互联在超大规模下的关键价值。

### 五、主流开源框架对比

![框架对比](assets/ai-infra-l0-3.png)

L0 涉及计算、编排、存储三大领域的框架选型。以下是 2026 年主流开源方案的深度对比：

| 框架 | 类型 | 定位 | 适用场景 | 2026 特点与版本 |
|------|------|------|----------|----------------|
| **Kubernetes + Volcano** | 编排调度 | 容器调度 + 批处理 | 通用 AI 平台，推理为主 | DRA 已 GA，GPU 插件成熟；Volcano gang scheduling 稳定；66% 企业在 K8s 上跑 GenAI 推理 |
| **Ray Cluster** | 分布式计算 | 训练 + 推理统一 | Python 原生 AI 工作负载 | Ray 2.55 稳定版；KubeRay 与 DRA 集成；Actor 模型，GPU 利用率提升 50%–70% |
| **Slurm** | HPC 调度 | 纯训练集群 | 大规模学术/工业训练 | 多租户公平调度最强；checkpoint 恢复成熟；非容器化，K8s 生态外运行 |
| **Kueue** | K8s 队列管理 | 多团队共享 GPU | 配合 Volcano 使用 | Kubernetes 原生，CRD 定义；支持优先级抢占和配额；与 DRA 协同 |
| **JuiceFS** | 分布式文件系统 | 模型权重共享 | POSIX + S3 后端 | CSI Driver 支持 PV；ReadWriteMany 多节点并发；NAVER/BioMap 生产验证，存储成本降低 90% |
| **Alluxio** | 数据编排 | 热数据缓存 | 跨存储加速 | 缓存层定位，非主存储；S3/FUSE 兼容性弱于 JuiceFS；适合已有存储系统加速 |

**选型建议矩阵：**

- **推理服务为主：** Kubernetes + Volcano + Kueue。DRA GA 后 GPU 调度已标准化，Volcano gang scheduling 保障推理 Pod 组完整启动，Kueue 管理多团队队列。
- **训练为主：** Ray Cluster（Python 原生团队）或 Slurm（HPC 传统团队）。Ray 在数据预处理 + 训练 + 评估端到端流程上更统一；Slurm 在超大规模训练和 checkpoint 管理上更成熟。
- **存储方案：** JuiceFS + S3 做主存储，NVMe 本地缓存做热层，Alluxio 可选做跨存储加速。避免直接用 s3fs 挂载对象存储——性能和兼容性都差。
- **网络方案：** 节点内 NVLink（B200 上 1.8 TB/s），节点间 InfiniBand HDR（大规模训练）或 RoCE v2 + Spectrum-X（成本敏感场景）。

### 六、企业生产实施方案

![生产实施](assets/ai-infra-l0-4.png)

将 L0 技术落地到企业生产环境，需要围绕推理和训练两条主线设计实施方案。以下是一个中型 AI 平台（8–32 张 B200/H100 GPU）的参考架构。

#### 6.1 推理 GPU 弹性伸缩：SLA 分池 + 自动扩缩容

推理服务不能所有请求混在一起跑。按 SLA 要求分池：高优池服务实时对话（P99 < 3s），普通池服务批量处理（无严格延迟要求）。高优池使用 MIG 物理隔离的 GPU 实例，保障 QoS；普通池使用时间分片共享 GPU，最大化利用率。

扩缩容策略采用双指标驱动（队列深度 + GPU 利用率），配合 KEDA 从 Prometheus 拉取自定义指标触发 HPA。Cluster Autoscaler 负责节点层扩缩，`scale-down-unneeded-time` 设为 10 分钟。凌晨低谷自动缩到最小副本数，空跑超过 5 分钟立即缩容。

关键配置要点：推理 Pod 必须设置 GPU 资源 request 和 limit 一致（避免超卖），MIG 策略选 single（管理简单）或 mixed（灵活切分），DCGM Exporter 采集 GPU 利用率指标供 Prometheus 抓取。

#### 6.2 训练任务队列：Volcano 公平调度 + 抢占

多团队共享训练集群，核心是公平调度和抢占机制。每个团队按权重分配 GPU 配额（如 A 团队 40%、B 团队 30%、C 团队 30%）。高优先级训练任务提交后，Volcano 自动抢占低优先级任务的 GPU，被抢占任务 checkpoint 后释放资源。

训练任务必须配置 checkpoint 策略——按步数间隔定期保存，被抢占后从最近 checkpoint 恢复，避免从头训练浪费算力。Ray Train 和 PyTorch FSDP 都支持自动 checkpoint，配合 JuiceFS 将 checkpoint 写到分布式存储，任意节点可恢复。

#### 6.3 模型权重版本化与 P2P 分发

模型权重管理是容易被忽视但极其关键的环节。权重散落在本地磁盘会导致版本混乱、推理结果不一致。企业级方案是统一的 Artifact Registry + P2P 分发：

```python
def deploy_model_weights(model_id, version, target_nodes):
    weights = registry.fetch(model_id, version,
                             verify_hash=True)
    for node in target_nodes:
        distribute(weights, node, method="p2p")
    wait_until(health_check(target_nodes), timeout=120)
    notify("模型 %s v%s 已就绪" % (model_id, version))
    update_serving_routes(model_id, version, target_nodes)
```

模型权重存储在 Artifact Registry（如 HuggingFace Hub 自建实例或自研 Registry），每个版本带 hash 校验确保完整性。分发到推理节点时用 P2P 协议（如 Dragonfly 或 Kraken），多个节点并行拉取，避免单点带宽瓶颈。权重预加载到 NVMe 本地缓存，推理启动时直接从本地读取，避免每次冷启动都从远端拉取。70B 模型 FP16 约 140GB，B200 单卡 192GB 显存可完整装载；在 NVMe 上预加载后，模型就绪时间可控制在 60 秒以内。

#### 6.4 网络与监控配置

训练集群节点间配置 InfiniBand 或 RoCE v2，NCCL 环境变量必须正确设置。监控层面，DCGM Exporter 采集 GPU 利用率、显存使用、温度、功耗等指标，Prometheus + Grafana 做可视化看板。关键告警：GPU 利用率持续 <30% 超过 10 分钟（空跑告警）、GPU 温度 >85°C（散热告警）、NVLink 带宽利用率异常（互联故障告警）。

### 七、指标与验收标准

L0 基础资源层的验收不能靠感觉，必须有量化指标。以下是 2026 年企业级 AI 平台的验收标准：

| 指标 | 目标值 | 衡量方式 |
|------|--------|----------|
| GPU 利用率 | ≥70% | Prometheus + DCGM Exporter 采集 |
| GPU 空跑率 | <5% | 无请求但占用 GPU 的时间占比 |
| 模型权重分发延迟 | <60s | 从 Registry 拉取到推理就绪 |
| 存储 IOPS（推理） | >10K IOPS | fio 压测验证 |
| RDMA 网络延迟 | <1ms | GPU 间 RDMA ping |
| 资源回收时间 | <3min | 训练任务结束到 GPU 释放 |
| 集群整体利用率 | ≥75% | 调度器优化后 vs 默认 40% |
| 弹性扩容响应时间 | <2min | 从指标触发到新 Pod Ready |

GPU 利用率 ≥70% 是核心指标——行业平均 30%–50%，做到 70% 意味着算力 ROI 接近翻倍。空跑率 <5% 靠空闲检测 + 自动缩容保障。集群整体利用率从默认 40% 提升到 75%，是 Volcano + Kueue + DRA 调度优化的直接效果。

### 八、常见坑与最佳实践

L0 落地过程中有五个高频踩坑点，每一个都直接关联成本或性能：

1. **GPU 空跑烧钱：** 推理服务凌晨没流量但 Pod 一直占着 GPU，8 张 H100 空跑一晚就是几百美元。实践：空闲 5 分钟自动缩容到最小副本数，配合 KEDA + Cluster Autoscaler。`scale-down-unneeded-time` 推理场景设 10 分钟。这一条做好就能降低 20%–40% 账单。

2. **存储 I/O 瓶颈拖慢推理：** 模型权重放在对象存储，每次推理 Pod 启动都要从 S3 拉几十 GB，冷启动 5 分钟以上。实践：模型权重预加载到 NVMe 本地缓存，JuiceFS 做分布式文件系统层支持多节点共享，P2P 分发加速拉取。权重分发延迟控制在 60s 以内。

3. **网络成为延迟瓶颈：** 分布式训练用标准以太网，梯度同步通信延迟高，8 卡训练性能可能只有 2 卡的 3 倍（理想是 7 倍）。实践：节点间用 InfiniBand 或 RoCE v2，正确设置 `NCCL_SOCKET_IFNAME` 和 `NCCL_IB_DISABLE` 环境变量。不设置的后果——多节点通信性能可能差 10–100 倍。

4. **训练任务排队等 GPU：** 默认 FIFO 调度，大团队垄断资源，小团队排队数天。实践：Volcano 公平调度 + 优先级抢占，按团队权重分配配额。高优任务自动抢占低优任务，被抢占任务 checkpoint 后恢复。

5. **模型权重散落本地磁盘：** 各节点各自管理权重，版本对不上导致 A/B 测试结果不可信。实践：统一 Artifact Registry + 版本化 + hash 校验，P2P 分发到推理节点。权重即代码，版本可追溯、可回滚。

### 九、和上下游层的关系

L0 是 AI Infra 的最底层，没有上游。它的下游关系清晰而紧密：
- **L1 模型与推理层：** L1 的推理引擎（vLLM/TGI/SGLang）直接跑在 L0 的 GPU 上，L0 的调度效率决定 L1 的延迟和成本。GPU 利用率从 30% 提到 70%，L1 的单位推理成本直接减半。
- **L2 数据与知识层：** 向量数据库需要存储资源，L0 的存储分层性能影响 L2 检索速度。JuiceFS 的 IOPS 和延迟直接决定向量检索的 P99。
- **L5 沙箱与工具层：** Agent 的代码执行沙箱需要计算资源，L0 的 CPU/GPU 调度为 L5 提供底层支撑。
- **L6 记忆与存储层：** 长期记忆存储依赖 L0 的对象存储和分布式文件系统。

L0 的调度效率和存储性能会向上传导，影响每一层的体验。下一篇文章我们讲 L1 模型与推理层——GPU 到了，怎么在上面把模型跑得又快又省。

### 十、总结：一张表带走

| 维度 | 要点 |
|------|------|
| 解决什么 | 算力调度、存储分层、网络低延迟 |
| 核心技术 | MIG GPU 虚拟化、弹性伸缩、Volcano 公平调度、RDMA/NVLink 互联、JuiceFS 存储分层 |
| 首选框架 | K8s+Volcano+Kueue（推理）、Ray 2.55（训练）、JuiceFS+S3（存储） |
| 2026 硬件 | B200 192GB HBM3e / 8.0 TB/s / 9,000 TFLOPS FP4；GB200 NVL72 72 卡超级节点 |
| 关键指标 | GPU 利用率≥70%、空跑率<5%、分发<60s、集群利用率≥75% |
| 最大坑 | GPU 空跑烧钱、I/O 瓶颈、网络延迟、权重散落 |
| 一句话 | L0 是地基，利用率就是钱 |

*作者：Knock | 约 4800 字*

---

---

## 相关笔记

- [[AI Infra 落地实战]] — 系列索引
- [[AI Infra 落地实战-L1 模型与推理层]] — 第 2 篇 · 模型网关与推理引擎
