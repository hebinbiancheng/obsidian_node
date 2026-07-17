---
title: AI Infra 落地实战 - L1 模型与推理层
date: 2026-07-05
tags: [AI, Infra, 推理引擎, 模型网关, 量化, KV Cache, PagedAttention, LiteLLM, vLLM]
category: 知识资料
source: https://mp.weixin.qq.com/s/mTlyFY7AOOiUU_IMESFt-A
author: Knock
---

# AI Infra 落地实战 · L1 模型与推理层

> 系列文章 [[AI Infra 落地实战]] 第 2 篇 · 模型网关、路由与推理引擎
> 网关是入口，路由是大脑，推理引擎是肌肉 — 三者合起来是 AI Infra 的神经中枢。

---
### 一、开篇场景：一个真实问题

某电商公司的智能客服上线三个月，CTO 收到一份诡异的报告：用户问同一个问题，有时 1 秒返回，有时 30 秒超时；月账单 $8 万，但没人能说清钱花在哪了。是 GPT-4o 太贵？是 Embedding 调用太多？是重试把费用翻倍了？财务要拆账到每个业务线，开发两手一摊：「我只调了 OpenAI 的接口，别的真不知道。」

更尴尬的是，大促当晚 OpenAI 限流，客服全线瘫痪 40 分钟。团队连夜接了 Anthropic 做备用，但代码里硬编码了 model=「gpt-4o」，切换靠改配置重启服务，中间又丢了 20 分钟。事后复盘，有人提出自建 vLLM 跑开源模型降本，可没人说得清 7B 和 72B 该怎么分工、量化会不会掉精度、KV Cache 会不会撑爆显存。

这不是个例。2026 年，几乎所有把 LLM 推上生产线的团队都会撞上同一堵墙：模型调用散落各处、成本无法归因、故障无法切换、性能无法榨干。这堵墙，就是 L1 模型与推理层要拆掉的。

### 二、这一层到底解决什么

L1 解决四个字的问题：**调、选、跑、省。**

**调：** 模型从哪来、怎么统一调用。一个企业可能同时用 OpenAI、Anthropic、Azure、自建 vLLM、还有国产模型的 API。如果每个业务自己直连，密钥散落、计费混乱、故障时各自为战。网关（Gateway）把所有模型收拢成一个入口，对外暴露统一的 OpenAI 兼容接口，对内做认证、限流、计费、审计。LiteLLM 在 2026 年已支持 100+ 模型供应商，P95 延迟仅 8ms（1k RPS 下），成为自建网关的事实标准。

**选：** 怎么为每个请求挑最合适的模型。简单问答用 7B 就够，复杂推理得上 72B 或前沿闭源模型。智能路由按任务类型、上下文长度、延迟 SLA、成本预算动态决策，能把整体 Token 成本砍掉 40%–60%。Fallback 链保证主模型挂了自动切备用，熔断器防止雪崩。

**跑：** 怎么把 GPU 性能榨干。推理引擎（vLLM、SGLang、TensorRT-LLM、LMDeploy）是这一层的技术内核。PagedAttention 消除显存碎片、连续批处理提升吞吐、Prefix Caching 复用前缀计算、Prefill-Decode 分离把算力与带宽各自最优化——这些技术决定了同样一张 H100 能服务多少并发用户。

**省：** 怎么把成本打下来。量化（INT8/INT4/AWQ/GPTQ/FP8/NVFP4）用精度换速度和显存，INT4 能把 70B 模型从 4 张 A100 压到 1 张 RTX 4090；KV Cache 复用让相同 system prompt 不重复计算，多轮对话命中率可达 75%–90%；缓存让重复查询成本直降 95%。L1 是 AI Infra 的神经中枢——网关统一入口，路由智能选模型，推理引擎榨干 GPU，缓存和量化把成本打下来。

### 三、核心架构图

![架构图](assets/ai-infra-l1-1.png)

上图是 L1 的完整数据流，从左到右四层串联：

**入口层：** 业务方（客服 Agent、代码助手、RAG 管道）统一走网关的 OpenAI 兼容 API，带上虚拟密钥（Virtual Key），密钥里编码了租户、应用、预算上限。网关不信任任何直连模型的请求。

**网关层（Model Gateway）：** 这是请求的第一站，依次穿过五道关卡——认证鉴权（谁在调）、智能路由（调哪个模型）、限流配额（Token 预算够不够）、缓存命中（是不是重复问题）、Fallback 编排（挂了切谁）。LiteLLM Proxy 是这层的典型实现，支持基于延迟、成本、权重的多种路由策略，并内置熔断与冷却机制。

**推理引擎层：** 网关把请求转发给具体的推理后端。2026 年的主流选择是 vLLM（通用云原生首选）、SGLang（多轮/Agent 场景的 RadixAttention 王者）、TensorRT-LLM（NVIDIA 极致性能天花板）、LMDeploy（国产模型与 INT4 量化强项）。每个引擎背后是 GPU 集群，跑着量化后的模型权重，用 PagedAttention 管理 KV Cache，用连续批处理提升吞吐。

**加速层：** 横切在引擎之下的是一系列优化技术——Prefix Caching、Speculative Decoding、Prefill-Decode 分离（Disaggregated Serving）、KV Cache 多级分层（HBM→DRAM→NVMe）。NVIDIA 在 2026 年推出的 Dynamo 框架把 KV Cache 划成 G1–G4 四级，与 OpenAI、Anthropic 的商用 Prefix Cache 思路一致。

整个链路的设计原则是：**业务方不感知模型部署细节，网关不感知 GPU 物理拓扑，引擎不感知上游路由策略——各层解耦，独立演进。**

### 四、关键技术拆解

![技术拆解](assets/ai-infra-l1-2.png)

L1 的核心技术可以拆成五块：智能路由、自动 Fallback、KV Cache 复用、量化、PagedAttention。下面逐个讲透，并给出可落地的伪代码。

#### 4.1 智能路由：把对的任务给对的模型

路由不是简单的 if-else。2026 年成熟的做法是三段式：静态规则打底（任务类型→默认模型）、LLM 路由器精分（用一个 <1B 的小模型在 10ms 内判断复杂度）、置信度兜底（便宜模型低置信度时自动升级到强模型）。一个客服请求，如果只是查订单状态，7B 量化模型足矣；如果用户在抱怨物流并要求赔偿，路由器识别出情感复杂度，直接走 72B 或 GPT-4o。APIScout 2026 年的实测数据显示，这种路由能把成本降低 40%–60%，而分类器增加的延迟仅约 20ms，对用户几乎无感。

```python
def route_model(request, fallback_chain):
    # 三段式路由：静态规则 + 复杂度评估 + SLA 约束
    model = select_model(
        task=request.task_type,
        context_len=request.context_length,
        latency_sla=request.max_latency_ms,
        cost_budget=request.remaining_budget)
    try:
        resp = call_model(model, request, timeout=model.sla)
        record_success(model)          # 供冷却机制统计
        return resp
    except (Timeout, RateLimit, ModelError) as e:
        mark_failure(model, error=e)   # 失败计数 +1
        if cooldown_triggered(model):  # 超阈值则临时摘除
            router.remove_deployment(model, ttl=60)
        if fallback_chain:
            return route_model(request, fallback_chain[1:])
        raise CircuitBreaker("所有备用模型不可用，降级到缓存回答")
```

三个关键设计：`select_model` 综合任务、上下文、SLA、预算四维决策；失败后递归调用 `fallback_chain[1:]` 形成天然降级链；`cooldown_triggered` 实现熔断——LiteLLM 的做法是同一 deployment 在时间窗口内失败超阈值就临时移出轮转。

#### 4.2 自动 Fallback 与熔断：防止雪崩

Fallback 最容易踩的坑是「备用也挂了」。大促当晚 OpenAI 限流，你切到 Azure，Azure 也限流，再切到自建 vLLM，自建集群因为突然涌入 3 倍流量也 OOM 了——这就是雪崩。正确的做法是三层防御：Fallback 链要有足够的多样性（闭源 + 自建 + 缓存兜底）；每层都要有独立的熔断器；最底层必须是「零依赖」的降级方案。

LiteLLM 在 2026 年支持三类 Fallback：通用 Fallback（任意错误）、上下文窗口 Fallback（prompt 太长自动换大窗口模型）、内容策略 Fallback（触发安全策略时换模型）。配合指数退避重试，对 RateLimitError 默认重试 5 次。

#### 4.3 Token 预算与配额：钱花在哪，一目了然

```python
def check_token_budget(user_id, app_id, estimated_tokens):
    spent = redis.incrby(
        f"tokens:{app_id}:{user_id}:{today()}", 0)
    daily_limit = config[app_id].daily_token_budget
    if spent + estimated_tokens > daily_limit:
        return Downgrade(model="qwen-7b",
                         reason="quota_exceeded")
    return Allow()
```

核心思想是「软降级」而非「硬拒绝」：超预算不报错，而是把模型从贵的降到便宜的。LiteLLM 支持 per-user / per-team / per-project / per-model 四级预算，并在接近上限时触发告警。

#### 4.4 KV Cache 复用：相同前缀不重复算

KV Cache 是 LLM 推理中最贵的资源。一个 70B 模型跑 128K 上下文，单条请求的 KV Cache 就要约 40GB——比模型权重本身还大。传统推理引擎为每条请求预留最大长度的连续显存，60%–80% 都浪费在碎片和超额预留上。

**PagedAttention**（vLLM 的核心创新）借鉴操作系统的分页机制：把 KV Cache 切成固定大小的「页」，用页表管理，按需分配。这把显存浪费压到接近零，同样一张卡能服务的并发请求数翻好几倍。

**Prefix Caching** 进一步复用相同前缀的计算。所有请求共享同一个 system prompt？只需算一次。SGLang 的 RadixAttention 用基数树管理前缀缓存，Few-shot 场景命中率 85%–95%，多轮对话 75%–90%。2026 年 2 月，SGLang 联合 NVIDIA 在 GB300 NVL72 上跑 DeepSeek R1，相对 H200 取得 25 倍性能提升。

```python
def serve_request(request, kv_cache):
    prefix_hash = hash(request.system_prompt
                       + request.few_shot)
    cached = kv_cache.get(prefix_hash)
    if cached:
        # 命中：只算新增 token，复用历史 KV
        result = model.forward(
            input_ids=request.new_tokens,
            past_key_values=cached)
    else:
        # 未命中：全量计算并写入缓存
        result = model.forward(request.full_input)
        kv_cache.set(prefix_hash,
                     result.past_key_values, ttl=300)
```

#### 4.5 量化：用精度换速度和显存

量化是降本最立竿见影的手段。FP16 的 70B 模型要 140GB，INT4 只要 35GB——一张 RTX 4090（24GB）配合 AWQ INT4 就能跑起来，而原本需要 4 张 A100。LMDeploy 的 TurboMind 引擎在 INT4 下相对 FP16 有 2.4 倍加速。

**2026 年量化格局：**

| 方案 | 适用场景 | 说明 |
|------|----------|------|
| **FP8** | 生产首选 | Hopper/Blackwell 原生支持，质量损失极小 |
| **NVFP4** | Blackwell 专属 | TensorRT-LLM + ModelOpt，MoE 吞吐再上台阶 |
| **AWQ** | GPU 服务首选 4-bit | 激活感知权重量化，vLLM/SGLang 首选 |
| **GPTQ** | 社区方案 | 基于 OBDB 算法，生态成熟 |
| **GGUF** | CPU/Mac/边缘 | llama.cpp 专属 |
| **INT4/INT8** | 通用 | 4-bit 在数学、代码、推理任务质量损失 1%–8% |

**一句话原则：** Blackwell 选 FP8/NVFP4，显存吃紧选 AWQ INT4，CPU/边缘选 GGUF，永远用量化后的 Golden Set 对比验证精度损失再上线。

### 五、主流开源框架对比

![框架对比](assets/ai-infra-l1-3.png)

2026 年的推理引擎格局已经从「vLLM 一家独大」演变为「四强分治」。

| 框架 | 类型 | 核心能力 | 适用场景 | 2026 状态 |
|------|------|----------|----------|-----------|
| **vLLM** | 推理引擎 | PagedAttention、连续批处理、最广生态 | 通用云原生推理首选 | 月度发布，CUDA 13.0，2-5x 优于通用方案 |
| **SGLang** | 推理引擎 | RadixAttention 前缀缓存、结构化生成 | 多轮对话、Agent 工作流 | GB300 上 25x 提升，agentic 吞吐可达 5x |
| **TensorRT-LLM** | 推理引擎 | 算子融合/编译、NVFP4、极致 GPU 优化 | NVIDIA 环境极限性能 | KV Cache Connector API、Disaggregated Serving |
| **LMDeploy** | 推理引擎 | TurboMind(C++)、INT4 量化强、部署一体化 | 国产模型、显存受限 | v0.13.0+，支持 Qwen3/DeepSeek-V3.2/GLM-5 |
| **TGI** | 推理引擎 | HuggingFace 生态、简单部署 | HF 模型快速上线 | 新模型支持略慢于 vLLM |
| **LiteLLM** | 网关 | 100+ 模型统一接口、路由/Fallback/计费 | 多供应商统一管理 | 8ms P95@1k RPS，MCP Gateway，四级预算 |

**选型建议：** 自建通用推理用 vLLM；多轮/Agent 场景用 SGLang；NVIDIA 集群极限性能用 TensorRT-LLM；国产模型或 INT4 量化用 LMDeploy；多供应商统一管理用 LiteLLM。实践中常见组合是 **LiteLLM（网关）+ vLLM/SGLang（自建后端）+ 闭源 API（兜底）**。

> ⚠️ 安全提醒：2026 年 4 月 LMDeploy 爆出 CVE-2026-33626（SSRF），推理服务器的管理 API 和多模态输入处理是新的攻击前沿。选型时务必把「是否有安全补丁机制」纳入考量。

### 六、企业生产实施方案

![生产实施](assets/ai-infra-l1-4.png)

#### 阶段一：自建网关，收拢入口

所有业务方不再直连 OpenAI/Anthropic，而是调用网关的虚拟密钥。网关配置三套路由策略：**成本优先**（简单任务走自建 7B）、**质量优先**（复杂任务走 72B 或闭源前沿）、**延迟优先**（实时场景走当前延迟最低的 deployment）。

```yaml
router_settings:
  routing_strategy: "latency-based-routing"
  num_retries: 3
  timeout: 30
  context_window_fallbacks:
    - "gpt-4o": ["claude-3-opus", "qwen-72b"]
  cooldown_time: 60
litellm_settings:
  max_budget: 100000        # 月度预算 $100k
  budget_duration: "30d"
  cache_responses: true
  cache_kwargs: {ttl: 3600, type: "redis"}
```

#### 阶段二：大小模型路由 + 量化部署

把流量按复杂度分流：70% 简单查询走 INT4 量化的 7B 模型（自建 vLLM/SGLang），20% 中等复杂度走 32B（FP8），10% 复杂推理走 72B 或闭源 API。量化部署前必须做 Golden Set 验证（500–1000 条标注样本，精度损失 <2% 才允许灰度）。

```python
def canary_deploy_quantized(model_id, quant_type, traffic=0.05):
    golden = load_golden_set(model_id)       # 标注样本
    quant_model = quantize(model_id, quant_type)
    loss = evaluate(quant_model, golden)     # 精度损失评估
    if loss > 0.02:                          # 阈值 2%
        raise QualityGate(f"量化损失 {loss:.1%} 超阈值，拒绝上线")
    # 灰度：5% 流量先跑，监控 P99 延迟与错误率
    router.add_deployment(quant_model, weight=traffic)
    for window in monitor(duration="1h"):
        if window.error_rate > 0.01 or window.p99 > sla:
            router.rollback(quant_model); break
        if window.stable:
            router.scale_traffic(quant_model, to=1.0)  # 全量
```

#### 阶段三：KV Cache 复用 + Prefill-Decode 分离

对于多轮对话和 RAG 场景，开启 Prefix Caching 收益巨大。对于高吞吐场景，上 Prefill-Decode 分离（Disaggregated Serving）：Prefill 是计算密集型（GEMM），Decode 是带宽密集型（GEMV），分离后各用最优硬件，可获 54% 吞吐提升、64% P90 TTFT 下降。

#### 阶段四：可观测与成本看板

每个请求必须能追溯到 app + user + model + token_cost，P50/P95/P99 延迟、Fallback 触发率、缓存命中率、量化精度漂移都要可视化。

### 七、指标与验收标准

![指标看板](assets/ai-infra-l1-5.png)

| 指标 | 目标值 | 衡量方式 |
|------|--------|----------|
| P99 延迟（简单任务） | <3s | Trace 端到端统计 |
| P99 延迟（复杂任务） | <8s | Trace 端到端统计 |
| Fallback 触发率 | <5% | 网关日志统计 |
| 量化精度损失 | <2% | Golden Set 对比 |
| KV Cache 命中率 | >30%（多轮 >75%） | 缓存命中统计 |
| Token 成本/请求 | 可归因到 app + user | 成本看板 |
| 网关可用性 | ≥99.9% | 健康检查 + 熔断统计 |
| 首 Token 延迟(TTFT) | <500ms | 引擎侧计时 |

验收建议：压测（真实流量回放验证 P99）、混沌测试（主动杀主模型验证 Fallback SLA）、成本审计（随机抽 100 条请求验证成本归因）。

### 八、常见坑与最佳实践

1. **Fallback 雪崩**：备用模型和主模型跑在同一云厂商。实践：Fallback 链跨厂商、跨形态（闭源 API + 自建 + 缓存兜底），最底层零依赖降级，配熔断器 60 秒冷却。
2. **量化后质量暴跌**：直接 AWQ INT4 上线没评测。实践：Golden Set 验证，精度损失 <2% 才灰度，数学/代码/推理密集型慎用 INT4，优先 FP8。
3. **KV Cache 内存爆炸**：Prefix Caching 没设上限。实践：设容量上限 + LRU 淘汰，`--mem-fraction-static 0.9` 限制静态显存占比。
4. **只用一个大模型扛所有任务**：所有请求走 GPT-4o。实践：按任务复杂度路由，简单走 7B（成本 1/10），LLM 路由器增加 ~20ms 延迟省 40%–60% 成本。
5. **流式输出中断无处理**：网络抖动断线，已生成内容全丢。实践：网关层断线重连 + 部分结果保留，SSE 重连时从最后收到的 token 续传。
6. **推理服务暴露管理 API**：管理端口和多模态输入处理是攻击面。实践：管理 API 绑内网、加鉴权，输入做 SSRF 过滤，及时打安全补丁。

### 九、和上下游层的关系

- **上游 L0（基础资源层）**：推理引擎跑在 L0 的 GPU 上，模型权重存在 L0 的存储里，RDMA 网络支撑 Prefill-Decode 分离时的 KV Cache 传输。
- **下游 L3（Prompt 与上下文层）和 L4（编排与 Agent 层）**：L3 把组装好的上下文发给 L1 推理，L4 的 Agent 通过网关调用模型。L1 把模型变成统一可调用的 API，L3/L4 不需要关心模型部署细节。

### 十、总结

| 维度 | 要点 |
|------|------|
| 解决什么 | 模型统一调用、智能选择、推理加速、成本控制 |
| 核心技术 | 智能路由、Fallback 熔断、KV Cache 复用、量化、PagedAttention、Prefill-Decode 分离 |
| 首选框架 | vLLM（通用推理）、SGLang（多轮/Agent）、LiteLLM（网关）、TensorRT-LLM（极致性能） |
| 关键指标 | P99<3s、Fallback<5%、量化损失<2%、KV 命中率>30% |
| 最大坑 | Fallback 雪崩、量化质量暴跌、KV Cache 爆显存、只用一个大模型 |
| 一句话 | L1 是神经中枢，路由省的不只是延迟，还有真金白银 |

---

*作者：Knock | 约 5500 字*

---

## 相关笔记

- [[AI Infra 落地实战]] — 系列索引
- [[AI Infra 落地实战-L0 GPU调度与云原生底座]] — 第 1 篇 · GPU 调度与云原生底座
