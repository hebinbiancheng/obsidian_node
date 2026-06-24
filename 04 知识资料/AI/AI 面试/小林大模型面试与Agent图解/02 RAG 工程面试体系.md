---
title: RAG 工程面试体系
type: knowledge
status: evergreen
source_type: 本地剪藏与链接解析
created: 2026-06-24
updated: 2026-06-24
tags:
  - AI
  - RAG
  - 向量检索
  - 面试
source:
  - "https://xiaolinnote.com/agent/rag/rag.html"
  - "https://xiaolinnote.com/agent/rag/graphrag-lightrag.html"
  - "https://xiaolinnote.com/ai/rag/rag_info.html"
  - "https://xiaolinnote.com/ai/rag/1_whatisrag.html"
  - "https://xiaolinnote.com/ai/rag/2_rag_problems.html"
  - "https://xiaolinnote.com/ai/rag/3_rag_vs_finetune.html"
  - "https://xiaolinnote.com/ai/rag/4_chunking.html"
  - "https://xiaolinnote.com/ai/rag/5_semantic_cuts.html"
  - "https://xiaolinnote.com/ai/rag/6_embedding.html"
  - "https://xiaolinnote.com/ai/rag/7_embedding_algos.html"
  - "https://xiaolinnote.com/ai/rag/8_vectordb.html"
  - "https://xiaolinnote.com/ai/rag/9_vectordb_practice.html"
  - "https://xiaolinnote.com/ai/rag/10_online_workflow.html"
  - "https://xiaolinnote.com/ai/rag/11_retrieval_types.html"
  - "https://xiaolinnote.com/ai/rag/12_query_rewrite.html"
  - "https://xiaolinnote.com/ai/rag/13_multi_retrieval.html"
  - "https://xiaolinnote.com/ai/rag/14_retrieval_opt.html"
  - "https://xiaolinnote.com/ai/rag/15_advanced_paradigms.html"
  - "https://xiaolinnote.com/ai/rag/16_graph_db.html"
  - "https://xiaolinnote.com/ai/rag/17_hallucination.html"
  - "https://xiaolinnote.com/ai/rag/18_evaluation.html"
  - "https://xiaolinnote.com/ai/rag/19_dynamic_update.html"
  - "https://xiaolinnote.com/ai/rag/20_hardest_parts.html"
aliases: []
---

# RAG 工程面试体系

## 一句话结论

RAG 的核心价值是把外部知识可靠接入 LLM；真正的工程难点不在“向量检索”四个字，而在知识入库、召回质量、上下文构造、评估、权限和持续更新。

## 标准链路

1. 文档采集与清洗。
2. Chunking：按语义、标题、段落、代码块或业务实体切分。
3. Embedding：把文本映射为向量，并选择适合领域的模型。
4. 索引存储：向量库、关键词索引、元数据过滤。
5. 查询改写：补全意图、拆分问题、生成多路查询。
6. 多路召回：向量检索、BM25、图检索、规则召回组合。
7. 重排与过滤：rerank、权限过滤、时效性过滤。
8. 上下文拼接与生成：控制引用、长度、去重和冲突处理。
9. 评估与反馈：命中率、答案正确率、引用准确率、延迟和成本。

## 关键面试点

- 微调解决模型行为和风格，RAG 解决知识更新和可追溯，两者不是互斥关系。
- Chunk 太小容易丢上下文，太大影响召回精准度和 Token 成本。
- Embedding 选择要看语言、领域、维度、速度、成本和评测集表现。
- RAG 幻觉治理要靠引用、上下文约束、答案校验、拒答策略和人工反馈。
- GraphRAG/LightRAG 适合实体关系复杂、跨文档推理明显的场景，但成本和构建复杂度更高。

## 来源链接

- [RAG 是什么？RAG 检索增强面试题万字图解](https://xiaolinnote.com/agent/rag/rag.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [GraphRAG 和 LightRAG 详解：原理、对比与选型](https://xiaolinnote.com/agent/rag/graphrag-lightrag.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [RAG 面试题介绍](https://xiaolinnote.com/ai/rag/rag_info.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [1. 什么是 RAG？详细描述一个完整 RAG 系统的详细工作流程？](https://xiaolinnote.com/ai/rag/1_whatisrag.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [2. 大模型的 RAG 主要用来解决什么问题？](https://xiaolinnote.com/ai/rag/2_rag_problems.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [3. 相比直接微调 LLM，RAG 解决了什么问题？微调和 RAG 各自的优劣势是什么？](https://xiaolinnote.com/ai/rag/3_rag_vs_finetune.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [4. RAG 中的文档是怎么存的？粒度是多大？详细说说文档切割（Chunking）策略？](https://xiaolinnote.com/ai/rag/4_chunking.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [5. 怎么规避语义被切割掉的问题？](https://xiaolinnote.com/ai/rag/5_semantic_cuts.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [6. 在 RAG 中 Embedding 究竟是什么？如何选择和评估一个 Embedding 模型？](https://xiaolinnote.com/ai/rag/6_embedding.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [7. Embedding 有哪几种算法你了解过吗？](https://xiaolinnote.com/ai/rag/7_embedding_algos.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [8. 什么是向量数据库？有没有做过向量数据库的对比选型？](https://xiaolinnote.com/ai/rag/8_vectordb.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [9. 讲讲你用的向量数据库？数据量级是多大？性能如何？遇到过性能瓶颈吗？](https://xiaolinnote.com/ai/rag/9_vectordb_practice.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [10. 你使用 RAG 给大模型一个输入，系统是怎样的工作流程？](https://xiaolinnote.com/ai/rag/10_online_workflow.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [11. 请你介绍一下向量检索和关键词检索的区别？](https://xiaolinnote.com/ai/rag/11_retrieval_types.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [12. 如何润色用户的 Query（Query Rewrite）？目的是什么？](https://xiaolinnote.com/ai/rag/12_query_rewrite.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [13. 什么是多路召回？具体怎么做？](https://xiaolinnote.com/ai/rag/13_multi_retrieval.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [14. RAG 检索优化策略有哪些？](https://xiaolinnote.com/ai/rag/14_retrieval_opt.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [15. 了解哪些更复杂的 RAG 范式？](https://xiaolinnote.com/ai/rag/15_advanced_paradigms.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [16. 在什么场景下，你会选择使用图数据库来增强传统的向量检索？](https://xiaolinnote.com/ai/rag/16_graph_db.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [17. 如何规避 RAG 系统中大模型的幻觉？](https://xiaolinnote.com/ai/rag/17_hallucination.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [18. 怎么量化你的 RAG 效果？](https://xiaolinnote.com/ai/rag/18_evaluation.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [19. RAG 知识库如何实现动态与持续更新？](https://xiaolinnote.com/ai/rag/19_dynamic_update.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
- [20. 在实际落地中，你觉得 RAG 最难的地方是哪里？](https://xiaolinnote.com/ai/rag/20_hardest_parts.html)：围绕 RAG 工作流、切分、Embedding、向量库、混合检索、Query Rewrite、评估、幻觉规避和持续更新展开。
