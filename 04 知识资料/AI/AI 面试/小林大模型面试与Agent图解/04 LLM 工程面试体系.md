---
title: LLM 工程面试体系
type: knowledge
status: evergreen
source_type: 本地剪藏与链接解析
created: 2026-06-24
updated: 2026-06-24
tags:
  - AI
  - LLM
  - Transformer
  - 部署
  - 评测
source:
  - "https://xiaolinnote.com/ai/llm/llm_info.html"
  - "https://xiaolinnote.com/ai/llm/what_is_llm.html"
  - "https://xiaolinnote.com/ai/llm/transformer_architecture.html"
  - "https://xiaolinnote.com/ai/llm/mha_mqa_gqa_flash_attention.html"
  - "https://xiaolinnote.com/ai/llm/position_encoding.html"
  - "https://xiaolinnote.com/ai/llm/tokenizer.html"
  - "https://xiaolinnote.com/ai/llm/llm_training.html"
  - "https://xiaolinnote.com/ai/llm/scaling_law_emergence.html"
  - "https://xiaolinnote.com/ai/llm/finetuning.html"
  - "https://xiaolinnote.com/ai/llm/lora.html"
  - "https://xiaolinnote.com/ai/llm/post_training.html"
  - "https://xiaolinnote.com/ai/llm/dpo_vs_ppo.html"
  - "https://xiaolinnote.com/ai/llm/decoding_strategies.html"
  - "https://xiaolinnote.com/ai/llm/temperature_top_p_top_k.html"
  - "https://xiaolinnote.com/ai/llm/kv_cache_prompt_caching.html"
  - "https://xiaolinnote.com/ai/llm/quantization.html"
  - "https://xiaolinnote.com/ai/llm/prompt_engineering.html"
  - "https://xiaolinnote.com/ai/llm/cot.html"
  - "https://xiaolinnote.com/ai/llm/hallucination.html"
  - "https://xiaolinnote.com/ai/llm/moe.html"
  - "https://xiaolinnote.com/ai/llm/deployment_frameworks.html"
  - "https://xiaolinnote.com/ai/llm/evaluation_metrics.html"
  - "https://xiaolinnote.com/ai/llm/model_selection.html"
aliases: []
---

# LLM 工程面试体系

## 一句话结论

LLM 工程面试覆盖从模型结构到训练、推理、部署、评测和选型的全链路；应用工程师不必成为训练专家，但必须能解释关键机制如何影响成本、延迟和效果。

## 基础原理

- Transformer 通过 Attention 建模序列依赖，Encoder/Decoder 适配不同任务形态。
- MHA、MQA、GQA、Flash Attention 都围绕注意力计算的效率、显存和吞吐优化。
- RoPE、ALiBi 等位置编码决定模型如何感知 token 顺序和长上下文。
- Tokenizer 影响分词粒度、上下文长度、费用和多语言表现。

## 训练与对齐

- 预训练学习通用语言和世界知识，SFT 学习指令跟随。
- LoRA/QLoRA 通过低秩适配降低微调成本。
- RLHF、DPO、GRPO、拒绝采样等属于后训练/偏好对齐方法。

## 推理与部署

- 解码策略包括贪心、Beam Search、采样、Top-P、Top-K、温度控制。
- KV Cache 和 Prompt Caching 直接影响长上下文和多轮对话成本。
- 量化在显存、速度和精度之间取舍，INT8/INT4/AWQ/GPTQ 适合不同部署约束。
- vLLM、TGI、SGLang、llama.cpp 的选择取决于吞吐、延迟、硬件、模型类型和工程生态。

## 评测与选型

- Benchmark 只能做初筛，业务评测集才决定能否上线。
- 模型选型要同时看效果、延迟、上下文、工具调用能力、成本、稳定性和合规。

## 来源链接

- [大模型工程面试题介绍](https://xiaolinnote.com/ai/llm/llm_info.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [1. 什么是大语言模型？和传统 NLP 模型有什么区别？](https://xiaolinnote.com/ai/llm/what_is_llm.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [2. 讲讲 Transformer 架构基本原理？Encoder 和 Decoder 是什么？](https://xiaolinnote.com/ai/llm/transformer_architecture.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [3. 多头注意力（MHA）有哪些局限？MQA、GQA、Flash Attention 怎么解决？](https://xiaolinnote.com/ai/llm/mha_mqa_gqa_flash_attention.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [4. 大模型的位置编码是干什么用的？sin/cos、RoPE、ALiBi 有什么区别？](https://xiaolinnote.com/ai/llm/position_encoding.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [5. 什么是大模型项目的分词器？原理是什么？](https://xiaolinnote.com/ai/llm/tokenizer.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [6. 大模型是怎么训练出来的？](https://xiaolinnote.com/ai/llm/llm_training.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [7. 什么是 Scaling Law？大模型的「涌现能力」是怎么回事？](https://xiaolinnote.com/ai/llm/scaling_law_emergence.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [8. 大模型微调的方案有哪些？](https://xiaolinnote.com/ai/llm/finetuning.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [9. 请讲一下 LoRA 技术，除了减少参数量，它还有哪些优点？](https://xiaolinnote.com/ai/llm/lora.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [10. SFT 之后还有哪些 Post-Training？RLHF、DPO、GRPO、拒绝采样什么关系？](https://xiaolinnote.com/ai/llm/post_training.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [11. 大模型的 DPO 和 PPO 的区别是什么？](https://xiaolinnote.com/ai/llm/dpo_vs_ppo.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [12. 大模型生成文本时的解码策略有哪些？贪心、Beam Search、采样分别什么时候用？](https://xiaolinnote.com/ai/llm/decoding_strategies.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [13. 大模型的参数：温度值、Top-P、Top-K 分别是什么？各个场景下的最佳设置是什么？](https://xiaolinnote.com/ai/llm/temperature_top_p_top_k.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [14. KV Cache 是什么？Prompt Caching 的原理是什么？](https://xiaolinnote.com/ai/llm/kv_cache_prompt_caching.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [15. 大模型量化是什么？INT8/INT4/AWQ/GPTQ 怎么选？](https://xiaolinnote.com/ai/llm/quantization.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [16. 如何写好 Prompt？分享下 Prompt 工程实践经验？](https://xiaolinnote.com/ai/llm/prompt_engineering.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [17. 什么是 CoT？为啥效果好？它有什么缺点或局限性？](https://xiaolinnote.com/ai/llm/cot.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [18. 大模型为什么会出现幻觉？怎么缓解？](https://xiaolinnote.com/ai/llm/hallucination.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [19. MoE 混合专家模型是什么？DeepSeek V3、Qwen 为什么用 MoE？](https://xiaolinnote.com/ai/llm/moe.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [20. 大模型部署有哪些主流方案？vLLM、TGI、llama.cpp、SGLang 实际项目里怎么选？](https://xiaolinnote.com/ai/llm/deployment_frameworks.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [21. 大模型能力评测指标有哪些？](https://xiaolinnote.com/ai/llm/evaluation_metrics.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
- [22. 对比使用过哪些主流大模型？你们项目中最终选用了哪个模型？为什么？](https://xiaolinnote.com/ai/llm/model_selection.html)：围绕 Transformer、Attention 优化、训练/微调、推理生成、缓存、量化、幻觉、MoE、部署和评测展开。
