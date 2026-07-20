---
title: AI Agent 架构面试题 - RAG 篇
date: 2026-07-04
tags: [AI, RAG, 面试, 检索增强生成, Embedding, 向量数据库, 幻觉]
category: 知识资料
source: https://mp.weixin.qq.com/s/XWBa1X-iLt7pOc-w4CLgvA
---

# AI Agent 架构面试题 - RAG 篇

> 原文：[微信文章](https://mp.weixin.qq.com/s/XWBa1X-iLt7pOc-w4CLgvA)
> 本文是 [[AI Agent 面试题与答案]] 的 RAG 专题拆分（共10题 + 5个高频追问）。

---

## 1. RAG知识库搭建流程（文档接入到检索）

### 核心流程分为四个阶段：

**阶段一：文档接入与预处理**
- **多源接入**：支持PDF、Word、Markdown、TXT、HTML、数据库、API等
- **格式解析**：使用PyMuPDF(pdf)、python-docx(docx)、Unstructured等库提取原始文本
- **文档清洗**：去除页眉页脚、水印、乱码字符、多余空白、特殊符号
- **元数据提取**：保留文档标题、作者、来源、时间戳等

**阶段二：文档分块（Chunking）**
- 将长文档切分为合适的语义单元（详见第2题）
- 常用工具：LangChain的TextSplitter、LlamaIndex的SentenceSplitter
- 保留块间重叠(overlap)以保持上下文连贯

**阶段三：向量化与索引构建**
- 选择Embedding模型将文本块转为向量（详见第4题）
- 存储到向量数据库：Milvus、Pinecone、Weaviate、Qdrant、FAISS等
- 构建倒排索引 + 向量索引的混合索引结构

**阶段四：检索与生成**
- 用户Query → Query改写/扩展 → Embedding → 向量相似度搜索
- Top-K召回 → （可选）重排序 → 拼接上下文 → LLM生成回答
- 支持元数据过滤、时间范围过滤等预过滤

### 关键面试要点：
- 整个流程即：**Ingestion → Chunking → Embedding → Indexing → Retrieval → Generation**
- 文档处理要考虑格式多样性、编码问题、大文件处理
- 索引更新策略：增量更新 vs 全量重建，需考虑一致性

---

## 2. 文档分块策略设计

### 主要分块策略：

**1. 固定大小分块（Fixed-size Chunking）**
- 按固定token数切分（如512 tokens）
- 优点：简单、均匀
- 缺点：可能切断语义完整性

**2. 语义分块（Semantic Chunking）**
- 基于句子边界、段落分隔符切分
- 计算相邻句子的embedding相似度，在相似度"断点"处切分
- 保持语义完整性更好

**3. 递归分块（Recursive Chunking）**
- 按优先级分隔符递归切分：段落(\n\n) → 句子(\n) → 词语(空格)
- LangChain的RecursiveCharacterTextSplitter

**4. 文档结构感知分块（Structure-aware Chunking）**
- 对Markdown按标题层级(# ## ###)切分
- 对HTML按标签结构（h1, h2, section）切分
- 保留层级结构和标题路径

**5. 句子窗口检索（Sentence Window Retrieval）**
- 以单句为索引单元，检索时扩展上下文窗口
- LlamaIndex的SentenceWindowNodeParser

### 关键参数：
- **Chunk Size**：通常256-1024 tokens（考虑模型上下文窗口）
- **Chunk Overlap**：通常10%-20%重叠，保证边界语义不丢失
- **分块策略选择**：代码用固定大小，文档用语义分块，结构化文档用结构感知分块

### 高级技巧：
- 父子文档模式（Parent-Child）：小块检索，大块喂给LLM
- 多粒度索引：同时建立段落级和句子级索引
- 动态分块：根据内容类型自适应调整块大小

---

## 3. 表格和图片的解析方法

### 表格解析：

**方案一：传统OCR + 表格识别**
- Camelot / Tabula：PDF表格提取
- Tesseract OCR + 表格结构识别
- 缺点：复杂表格（合并单元格、无边框）效果差

**方案二：多模态LLM**
- GPT-4V / Claude / Qwen-VL：直接识别表格并转Markdown
- 将表格图片输入多模态模型，输出结构化格式
- 适合复杂表格场景

**方案三：专用表格解析模型**
- Table Transformer（Microsoft）：检测表格结构
- PaddleOCR + PP-Structure：端到端表格识别
- Unstructured.io的表格解析模块

**方案四：Markdown/LaTeX表示**
- 解析后转为Markdown表格格式存储
- 同时保留原始文本描述供检索

### 图片解析：

**方案一：多模态Embedding直接索引**
- 使用CLIP / SigLIP等多模态模型将图片转向量
- 图文统一向量空间，支持以文搜图
- 检索时使用相同的多模态Embedding模型

**方案二：图片描述生成 + 文本索引**
- 用视觉模型（BLIP-2 / GPT-4V）生成图片描述文本
- 将描述文本嵌入索引
- 检索到描述文本时同时返回原图URL

**方案三：OCR + 布局分析**
- PaddleOCR提取图中文字
- LayoutParser / Unstructured分析文档布局
- 区分文字区域、图片区域、表格区域

### 多模态RAG最佳实践：
- 图片和表格统一转为"文本摘要 + 向量 + 原始文件引用"
- 使用Multi-Vector Retriever：一个图片可有多个向量（摘要向量、OCR向量、标签向量）
- 检索时返回原始图片/表格给LLM直接理解

---

## 4. Embedding模型选型和评估指标

### 主流Embedding模型：

| 模型 | 维度 | 特点 | 适用场景 |
|------|------|------|----------|
| text-embedding-3-large (OpenAI) | 256/1024/3072 | 多语言，Dimension可调 | 通用 |
| bge-large-zh-v1.5 (BAAI) | 1024 | 中文最佳 | 中文场景 |
| m3e-base/large | 768/1024 | 中文开源 | 中文场景 |
| stella-mrl-large-zh | 1024 | 中文，Matryoshka | 中文场景 |
| multilingual-e5-large | 1024 | 多语言 | 多语言场景 |
| GTE-Qwen2-7B-instruct | 3584 | 基于Qwen2，MTEB榜首 | 高精度 |
| Cohere embed-multilingual-v3 | 1024 | 多语言 | 企业级 |

### 评估指标：

**1. MTEB（Massive Text Embedding Benchmark）**
- 涵盖分类、聚类、配对、重排序、检索、STS等任务
- 目前英文榜单GTE-Qwen2领先，中文榜单BGE/stella领先
- 面试必提的评估标准

**2. C-MTEB（中文版MTEB）**
- 针对中文的评估基准
- 包含分类、聚类、语义相似度、检索等子任务

**3. 检索类核心指标**
- **Recall@K**：Top-K结果中包含正确答案的比例
- **MRR（Mean Reciprocal Rank）**：第一个正确答案排名的倒数均值
- **NDCG@K**：考虑排序位置的归一化折损累计增益
- **Hit Rate@K**：Top-K中至少命中一个的比例

**4. 选型考量因素**
- **语言支持**：中文场景优先选BGE/m3e/stella
- **向量维度**：高维度精度高但存储大，可选Matryoshka Embedding降维
- **上下文长度**：能否处理长文档（如Jina AI的8K token）
- **推理速度**：本地部署 vs API调用
- **成本**：开源免费 vs 商业API按量计费

---

## 5. MySQL和Elasticsearch数据一致性保证

### 业界方案：

**方案一：CDC（Change Data Capture）**
- 使用Debezium + Kafka监听MySQL Binlog
- 变更事件异步写入ES
- 最终一致性，延迟秒级
- 适合对实时性要求不极高的场景

**方案二：双写（Dual Write）**
- 应用层同时写MySQL和ES
- 可用本地事务表 + 重试机制保证
- 问题：网络分区时可能出现不一致

**方案三：Canal（阿里开源）**
- 模拟MySQL Slave，解析Binlog
- 推送变更到ES（通过MQ或直接写入）
- 国内最常用方案

**方案四：应用层事务消息**
- 基于本地事务表（Outbox Pattern）
- 写MySQL时同时写入outbox表，异步消费写入ES
- 保证At-Least-Once语义

### 一致性保证策略：

**1. 最终一致性（最常用）**
- 接受短暂不一致（秒级延迟）
- 通过Binlog/CDC异步同步
- 配合监控告警

**2. 强一致性方案**
- MySQL和ES都写成功后返回（性能差）
- 分布式事务（如Seata）但会大幅增加延迟

**3. 数据对账与修复**
- 定时全量/增量对账任务
- 发现不一致时自动修复
- 配合数据版本号比对

### 面试要点：
- 优先选择CDC/Canal方案，技术成熟度高
- 需要确保顺序性（同一文档的变更按序写入）
- 幂等性设计（ES的doc id做幂等）
- 监控延迟和积压

---

## 6. 手动干预切片的方法和原因

### 手动干预的必要性：

**核心原因：**
- 自动分块可能切断关键语义（如法律条款互相引用）
- 表格/代码等结构化内容需要特殊处理
- 领域知识决定的最佳切分点无法自动化
- QA对需要问题-答案成对，不能拆开

### 手动干预方法：

**方法一：自定义分隔符**
```python
# LangChain中指定分隔符优先级
text_splitter = RecursiveCharacterTextSplitter(
    separators=["\n## ", "\n### ", "\n#### ", "\n", "。", ".", " "],
    chunk_size=1000,
    chunk_overlap=100
)
```

**方法二：预定义切分规则**
- 对Markdown按二级标题切分
- 对法律文档按"第X条"切分
- 对代码按函数/类边界切分
- 对FAQ按Q&A对切分

**方法三：元数据标记法**
- 在文档中插入特殊标记：`<!-- CHUNK_BREAK -->`
- 预处理器识别标记强制在此处切分
- 也可用`<!-- NO_BREAK -->`标记禁止切分

**方法四：自定义Chunking Pipeline**
- 编写预处理脚本，根据文档类型应用不同规则
- 配置化管理切分规则，支持A/B测试
- LlamaIndex的IngestionPipeline支持自定义Transform

**方法五：人机协同标注**
- 标注人员标记切分边界
- 少量人工标注 + 模型学习自动切分
- 长期积累领域切分规则库

### 面试要点：
- 自动分块是基础，手动干预是提效关键
- 常见场景：法律条款、问答对、表格数据、多级标题文档
- 关键是**规则可配置、干预可追溯、效果可评估**

---

## 7. 召回和重排策略

### 召回策略（Retrieval）：

**1. 稀疏检索（Sparse Retrieval）**
- BM25 / TF-IDF：基于关键词匹配
- 优点：精确匹配好，可解释
- 缺点：无语义理解，同义词无法召回

**2. 稠密检索（Dense Retrieval）**
- 基于向量相似度（Cosine / Dot Product / Euclidean）
- 优点：语义理解强，鲁棒性好
- 缺点：可能漏掉精确关键词匹配

**3. 混合检索（Hybrid Retrieval）**
- BM25 + 向量检索的融合
- 使用RRF（Reciprocal Rank Fusion）或加权融合
- **当前RAG系统最推荐的召回方式**

**4. 多路召回**
- 不同粒度召回（文档级 + 段落级 + 句子级）
- 不同Embedding模型召回
- 元数据过滤 + 向量检索

### 重排策略（Reranking）：

**1. Cross-Encoder重排序**
- 模型：bge-reranker-v2、Cohere Rerank、Jina Reranker
- Query与每个候选文档拼接后打分
- 精度高但速度慢，通常Top-100→Top-10

**2. LLM-based重排序**
- 用LLM对候选排序（如RankGPT）
- 适合小批量高精度场景

**3. 多因子排序**
- 向量相似度 + BM25分数 + 时间衰减 + 文档权重
- LambdaMART等学习排序算法

### 典型Pipeline：
```
Query → 多路召回(向量+BM25) → Top-100
     → Cross-Encoder Reranker → Top-10
     → LLM重排(可选) → Top-5
     → 拼接上下文 → LLM生成
```

### 面试关键点：
- 混合检索(稀疏+稠密)是标准答案
- RRF融合公式：`score(d) = Σ 1/(k + rank_i(d))`
- 重排器在精度和延迟之间权衡
- 可以加多样性优化（MMR算法）

---

## 8. 提高模型回答准确率的方法

### 检索端优化：
1. **Query改写**：用户问题→检索优化问题（HyDE、Multi-Query、Step-back）
2. **Query扩展**：同义词扩展、子问题分解
3. **混合检索**：BM25 + 向量 + 元数据过滤
4. **更好的Embedding模型**：选型优化

### 上下文构建优化：
5. **文档重排序**：Cross-Encoder Reranker
6. **上下文压缩**：LLMLingua等压缩无关内容
7. **上下文选择与排序**：相关性从高到低排列
8. **引用源标注**：每个事实标注来源，可追溯

### Prompt工程：
9. **结构化Prompt**：系统提示 + 上下文 + 用户问题 + 输出约束
10. **Few-shot示例**：提供期望格式的示例
11. **思维链(CoT)**：要求模型逐步推理
12. **明确"不知道"**：Prompt中要求无相关信息时诚实回答

### 后处理与校验：
13. **事实一致性校验**：用NLI模型验证答案与来源是否一致
14. **答案过滤**：低置信度答案需要降级处理
15. **Self-Reflection**：让LLM自我检查答案质量

### 系统架构优化：
16. **多轮对话改写**：将对话历史浓缩为检索Query
17. **缓存热门问题**：减少重复计算
18. **A/B测试**：持续优化各环节参数
19. **用户反馈闭环**：收集点赞/踩，优化检索和生成

### 核心公式：
> **准确率 = 检索质量 × 上下文质量 × 生成质量 × 校验质量**

---

## 9. "Lost in the Middle"问题处理

### 问题定义：
LLM在处理长上下文时，对**中间位置**的信息关注度最低，而对开头(Primacy Bias)和结尾(Recency Bias)的信息关注度更高。这是2023年斯坦福等机构研究发现的现象。

### 影响：
- 关键文档放在中间位置可能被模型"忽略"
- 随着上下文增长，模型的Recall整体下降
- 10+个文档时问题尤其严重

### 解决方案：

**方案一：重排序内容位置**
- 将最相关文档放在开头和结尾
- 相关性递减的U型排列
- 实验证明可提升10-20%的准确率

**方案二：上下文压缩与筛选**
- 减少喂给LLM的文档数量（Top-3到Top-5）
- 用LLMLingua、Selective Context压缩
- 质量优于数量

**方案三：分步检索与生成**
- 第一轮：检索相关文档
- 第二轮：从检索结果中提取关键片段
- 第三轮：仅用关键片段生成

**方案四：Map-Reduce / Map-Rerank**
```
Map: 每个文档独立回答 → 得分
Reduce: 汇总最高分答案 → 最终答案
```

**方案五：迭代式检索（IRCoT等）**
- 不一次性检索所有文档
- 根据已检索内容决定下一步检索什么
- 递归搜索，避免上下文过长

**方案六：使用支持长上下文的模型**
- Claude 3 (200K)、GPT-4 Turbo (128K)、Gemini 1.5 Pro (1M)
- 但即使支持长上下文，中间信息丢失问题依然存在

### 面试要点：
- 承认这是LLM的已知局限
- 核心思路：**减少文档数量 + 优化文档排列 + 分步处理**
- 结合Reranker确保最重要的信息不丢失

---

## 10. 控制模型幻觉（RAG / Prompt / 输出约束角度）

### RAG角度：

**1. 高质量检索**
- 确保检索结果与问题高度相关
- 设置相关性阈值，低分文档不纳入上下文
- 混合检索 + 重排，提升Top-K精度

**2. 来源标注与引用**
- 每个检索文档附带来源URL/标题
- Prompt要求模型引用来源：`请基于[来源X]回答…`
- 输出格式约束：`<引用>来源1</引用>事实陈述`

**3. 检索结果校验**
- 使用NLI（自然语言推理）模型验证
- 判断"答案"是否被"检索上下文"支持（Entailment / Contradiction / Neutral）
- 矛盾时拒绝回答或降级处理

**4. 多路验证**
- 同一问题多路检索
- 交叉验证信息一致性
- 不一致时标记为低置信度

### Prompt角度：

**5. 显式约束指令**
```
你只能根据提供的上下文回答。如果上下文中没有相关信息，
请直接回答"根据已有资料，我无法回答这个问题"，
不要编造任何信息。
```

**6. 结构化输出格式**
- 要求模型区分"事实"和"推断"
- JSON格式输出，包含confidence字段

**7. Few-shot反幻觉示例**
- 提供"没有信息时正确拒绝"的示例
- 提供"有信息时严格引用"的示例

**8. 角色定位Prompt**
```
你是一个严格的信息检索助手。你的回答必须：
- 每个陈述都可在上下文中找到依据
- 不确定时标明"据推测"
- 没有信息来源时明确说明
```

### 输出约束角度：

**9. 后处理过滤**
- 事实一致性评分（FactCC, AlignScore等）
- 幻觉检测模型（如Vectara的HHEM）
- 低质量回答自动降级为"无法回答"

**10. Self-Reflection（自我反思）**
```
请检查你的回答中，是否有任何信息在提供的上下文中找不到依据。
如果有，请重新回答，仅使用上下文中的信息。
```

**11. 约束解码（Constrained Decoding）**
- 使用Outlines / Guidance库限制输出格式
- 强制输出必须包含引用标记
- JSON Schema约束确保结构化输出

**12. 知识边界标记**
- 在输出中标记知识来源时间
- `[知识截止: 2024年]` / `[来源: 文档A，第3段]`

### 三条防线总结：
```
第一道防线（RAG层）：检索高质量、高相关文档
第二道防线（Prompt层）：严格约束只基于上下文回答
第三道防线（输出层）：后校验 + Self-Reflection + 输出过滤
```

### 面试核心表述：
> "控制幻觉需要**分层防御**：检索层确保输入质量，Prompt层约束行为边界，输出层校验事实一致性。三层缺一不可。"

---

## 附录：RAG面试高频追问

1. **如何处理检索不相关的文档？** → 相关性阈值过滤 + Reranker
2. **如何评估RAG系统质量？** → RAGAS框架（Faithfulness, Answer Relevancy, Context Precision, Context Recall）
3. **如何支持多轮对话RAG？** → 对话历史改写为独立Query + 上下文窗口管理
4. **如何处理实时数据更新？** → 增量索引 + CDC + 版本管理
5. **向量数据库选型？** → 规模<100万用FAISS/Chroma，>100万用Milvus/Qdrant，SaaS用Pinecone

---

---

## 相关笔记

- [[AI Agent 面试题与答案]] — 完整索引页
- [[Agent 架构面试题-Agent核心篇]] — Agent 核心（10题）
- [[Agent 架构面试题-系统设计篇]] — 系统设计与基础（10题）
