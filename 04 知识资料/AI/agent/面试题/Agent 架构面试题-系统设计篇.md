---
title: AI Agent 架构面试题 - 系统设计与基础篇
date: 2026-07-04
tags: [面试, 系统设计, 多租户, SSE, Java, Go, 线程池, HashMap, 算法]
category: 知识资料
source: https://mp.weixin.qq.com/s/XWBa1X-iLt7pOc-w4CLgvA
---

# AI Agent 架构面试题 - 系统设计与基础篇

> 原文：[微信文章](https://mp.weixin.qq.com/s/XWBa1X-iLt7pOc-w4CLgvA)
> 本文是 [[AI Agent 面试题与答案]] 的系统设计与基础专题拆分（共10题）。

---

# 系统设计与基础面试题答案整理

---

## 1. 多租户隔离实现方式

多租户（Multi-tenancy）是指单个软件实例同时服务多个客户（租户），核心是**数据隔离**和**资源隔离**。

### 数据隔离的三种方案

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **独立数据库** | 每个租户一个独立Database | 隔离性最强，安全性最高，易于备份恢复 | 成本高，维护复杂，连接池压力大 |
| **共享数据库 + 独立Schema** | 同一DB，每个租户一个Schema | 隔离性较好，成本适中 | Schema管理复杂，跨租户查询困难 |
| **共享数据库 + 共享Schema** | 所有租户同一套表，通过tenant_id区分 | 成本最低，扩展性好 | 隔离性最弱，需严格保证SQL带tenant_id过滤 |

### 资源隔离方式
- **应用层隔离**：ThreadLocal绑定租户上下文，AOP拦截器注入tenant_id
- **连接池隔离**：每个租户分配独立的数据库连接池
- **容器化隔离**：每个租户独立部署在K8s Pod中（高端客户）
- **网关隔离**：通过API网关路由不同租户到不同后端服务

### 关键技术点
- 租户上下文传递：请求入口 → ThreadLocal → MyBatis拦截器 → SQL自动追加tenant_id
- 数据迁移与扩容：分库分表时按tenant_id进行水平拆分
- 共享数据表（如字典表）可使用全局公共表（`tenant_id = 0`或`NULL`）

---

## 2. SSE流式输出中断后的内容保留和恢复

SSE（Server-Sent Events）是单向流式推送协议，比WebSocket轻量。中断后的内容保留与恢复是核心难点。

### 方案一：前端缓存 + 断点重连
```
服务端: 每条消息携带自增eventId
客户端: 记录lastEventId
重连时: 请求头带上 Last-Event-ID
服务端: 根据lastEventId从消息队列中回溯未送达的消息
```

### 方案二：服务端消息队列缓冲
- 每个SSE连接维护一个Ring Buffer（环形队列）
- 连接断开时，buffer内的消息不丢弃
- 重连后从buffer中恢复（需保证时间窗口内的消息不丢失）
- Redis Streams作为持久化消息队列

### 方案三：前端内容缓存 + 重新请求
- 前端将已渲染内容存储在SessionStorage
- 断开后自动重连，携带`Start-From-Message-Id`
- 服务端根据ID继续推送后续内容，前端拼接

### GPT/大模型场景特化方案
- **token级断点续传**：服务端保存已生成的完整文本，客户端重连后从`resume`位置继续
- **会话级持久化**：大模型回答写入数据库（如对话历史表），前端轮询+SSE并存
- **兜底方案**：SSE断开超过N秒 → 提示用户刷新，从后端接口拉取完整回复

---

## 3. 模型响应速度优化方法

### 推理层面优化
| 方法 | 原理 | 效果 |
|------|------|------|
| **KV Cache** | 缓存已计算的Key/Value，避免重复计算 | 减少约50%推理时间 |
| **量化（INT8/INT4）** | 降低模型精度，减少计算量 | 推理速度提升2-4倍 |
| **Flash Attention** | IO-aware的注意力计算优化 | 内存占用降低，速度提升 |
| **vLLM/PagedAttention** | 分页管理KV Cache，减少碎片 | 吞吐量提升显著 |
| **投机解码（Speculative Decoding）** | 小模型草稿 + 大模型验证 | 延迟降低2-3倍 |
| **模型并行（Tensor/Pipeline）** | 多GPU并行推理 | 大模型推理必备 |

### 工程层面优化
- **流式输出（SSE/WebSocket）**：首token延迟从数秒降到毫秒级
- **模型预热**：服务启动时加载模型到显存，避免首次请求冷启动
- **批处理（Batching）**：动态合并多个请求，提高GPU利用率
- **语义缓存**：相似问题命中缓存直接返回（如GPTCache）
- **CDN边缘部署**：小型模型部署在边缘节点，减少网络延迟

---

## 4. Prompt层面提速和稳定性优化

### 提速优化
- **精简System Prompt**：删除冗余说明，每个字都有目的
- **Few-shot示例压缩**：用最少的示例表达模式，2-3个示例通常足够
- **结构化Prompt**：用XML标签或Markdown分隔不同部分，减少模型解析开销
- **前缀缓存**：API服务会缓存相同前缀的Prompt，固定System Prompt+Templates前缀
- **控制输出长度**：`max_tokens`参数限流，避免生成超长无用内容
- **减少CoT链长度**：只在必要的推理场景使用思维链

### 稳定性优化（防幻觉/格式保证）
- **显式格式约束**：明确要求JSON/Markdown/列表格式输出
- **少量示例引导**：提供2-3个正确格式的示例（few-shot）
- **负面约束**：明确禁止行为，如"不要输出解释，只输出JSON"
- **结构化输出**：使用function calling / JSON mode
- **分步验证**：先让模型输出"推理过程"，再输出"最终答案"
- **temperature控制**：简单任务用低temperature(0-0.3)，创意任务用高值(0.7-1.0)
- **输出校验后处理**：前端对输出做正则校验，不合规则重试

---

## 5. 完整Prompt的结构组成

一个完整的生产级Prompt通常包含以下层次：

```
┌─────────────────────────────────┐
│  1. System Prompt（系统提示）       │  ← 角色、能力、限制、输出格式
├─────────────────────────────────┤
│  2. Context / Background（上下文）  │  ← 业务背景、用户信息、对话历史
├─────────────────────────────────┤
│  3. Instructions（任务指令）        │  ← 具体的任务描述
├─────────────────────────────────┤
│  4. Examples / Few-shot（示例）    │  ← 输入输出示例（Format + Pattern）
├─────────────────────────────────┤
│  5. Constraints（约束条件）         │  ← 长度限制、格式要求、禁止事项
├─────────────────────────────────┤
│  6. Output Format（输出格式）       │  ← JSON Schema、Markdown模板
├─────────────────────────────────┤
│  7. User Query（用户问题）          │  ← 实际要处理的问题
└─────────────────────────────────┘
```

### 各部分的精细拆解

**System Prompt 核心要素：**
- Role（角色）：你是一个XX专家
- Capability（能力边界）：你能做什么/不能做什么
- Tone（语气）：专业/友好/简洁
- Operational Rules（操作规则）：step-by-step / 调用工具 / 追问机制

**Few-shot 设计原则：**
- 示例覆盖边界情况（正常+异常）
- 示例格式与期望输出一致
- 示例数量：分类任务3-5个，复杂推理2-3个

**Template友好写法：**
```
System: 你是{role}，专注于{domain}。
规则：
1. 始终用中文回答
2. 不了解时诚实说不知道
3. 输出格式：{format}

示例1：
输入：{input1}
输出：{output1}

现在请处理：{user_query}
```

---

## 6. Token和字符的区别

### 核心概念
| 维度 | Token | 字符（Character） |
|------|-------|-------------------|
| **定义** | 模型处理的最小语义单元 | 文本的最小书写单位 |
| **粒度** | 可能是字、词、子词甚至标点 | 单个Unicode码点 |
| **示例** | "GPT"可能是1个token | "GPT"是3个字符 |
| **关系** | 1 token ≈ 0.75个英文单词 ≈ 中文1-2个字 |

### 中文Token化细节
- **GPT系列（BPE）**：中文一个字通常1-2个token，"人工智能"≈6个token
- **DeepSeek系列**：中文压缩较好，通常1-1.5个token/字
- **英文**：平均1 token ≈ 0.75个词，"artificial intelligence"可能是2个token

### 关键影响
- **计费**：API按token计费，中文约1.5-2倍于同等意义的英文
- **上下文窗口**：同样字符数的中文消耗更多token（限制了上下文窗口有效内容量）
- **生成速度**：token/s是衡量生成速度的指标，与字符/s有换算差异

### 实际换算
```
英文: 1000 tokens ≈ 750 words ≈ 3000 characters
中文: 1000 tokens ≈ 600-1000 汉字 ≈ 1500-2000 characters
```

---

## 7. Java线程池核心参数

### ThreadPoolExecutor七大核心参数

```java
public ThreadPoolExecutor(
    int corePoolSize,        // 核心线程数
    int maximumPoolSize,     // 最大线程数
    long keepAliveTime,      // 空闲线程存活时间
    TimeUnit unit,           // 时间单位
    BlockingQueue<Runnable> workQueue,  // 任务队列
    ThreadFactory threadFactory,        // 线程工厂
    RejectedExecutionHandler handler    // 拒绝策略
)
```

### 参数详解

| 参数 | 含义 | 常用值 |
|------|------|--------|
| **corePoolSize** | 常驻核心线程数（即使空闲也不回收） | CPU密集型=N+1，IO密集型=2N+1 |
| **maximumPoolSize** | 最大允许线程数 | corePoolSize的1.5-2倍 |
| **keepAliveTime** | 非核心线程空闲超时被回收 | 60s |
| **workQueue** | 任务等待队列 | ArrayBlockingQueue(有界)/LinkedBlockingQueue(无界)/SynchronousQueue |
| **threadFactory** | 创建线程的工厂，用于命名和分组 | 自定义带业务名称的工厂 |
| **handler** | 队列满且线程数达max时的拒绝策略 | 见下方 |

### 四个拒绝策略
| 策略 | 行为 |
|------|------|
| **AbortPolicy（默认）** | 抛RejectedExecutionException |
| **CallerRunsPolicy** | 由调用者线程执行任务（降级） |
| **DiscardPolicy** | 静默丢弃新任务 |
| **DiscardOldestPolicy** | 丢弃队列中最旧的任务，重试新任务 |

### 任务提交流程
```
提交任务 → 核心线程数未满? → 创建新线程执行
           ↓ 已满
         队列未满? → 入队等待
           ↓ 已满
         最大线程数未满? → 创建新线程执行
           ↓ 已满
         执行拒绝策略
```

### JDK内置线程池
- `Executors.newFixedThreadPool(n)` → 固定线程数（⚠️LinkedBlockingQueue无界，可能OOM）
- `Executors.newCachedThreadPool()` → 无限扩容（⚠️可能大量创建线程）
- `Executors.newSingleThreadExecutor()` → 单线程串行
- `Executors.newScheduledThreadPool(n)` → 定时/周期任务

> ⚠️ **阿里规约**：禁止使用Executors创建线程池，应通过ThreadPoolExecutor构造方法手动指定参数。

---

## 8. 进程/线程/协程区别及协程优势场景

### 三者核心区别

| 维度 | 进程（Process） | 线程（Thread） | 协程（Coroutine） |
|------|----------------|----------------|-------------------|
| **资源分配** | 操作系统分配资源的最小单位 | CPU调度的基本单位 | 用户态调度的执行单元 |
| **内存** | 独立地址空间，进程间隔离 | 共享进程内存 | 共享线程内存 |
| **切换开销** | 大（上下文切换，TLB刷新） | 中（内核态切换） | 小（用户态切换，无系统调用） |
| **创建开销** | 大 | 中 | 极小（KB级别） |
| **通信方式** | IPC（管道/共享内存/消息队列） | 共享变量（需加锁） | 共享变量（Channel通信） |
| **调度者** | 操作系统 | 操作系统 | 程序自身/运行时 |
| **适用场景** | 隔离性要求高的独立服务 | CPU密集 + 多核并行 | IO密集 + 高并发 |

### 协程的优势场景

**1. 高并发IO密集型（最佳场景）**
- Web服务器同时处理数万连接（如Go的goroutine，Python async/await）
- 数据库连接池管理
- API网关请求转发

**2. 低延迟要求**
- 协程切换纳秒级，线程切换微秒级
- 实时系统、游戏服务器、交易系统

**3. 资源受限环境**
- 嵌入式设备（内存有限，线程过多导致OOM）
- Serverless函数（冷启动敏感）

### 协程实现模式
| 语言 | 实现 |
|------|------|
| **Go** | goroutine + GMP调度模型 |
| **Python** | async/await + asyncio事件循环 |
| **Kotlin** | suspend函数 + 协程框架 |
| **Java** | Project Loom虚拟线程（JDK21+） |
| **JavaScript** | Promise/async-await（单线程事件循环） |

### 一句话总结
> 进程是资源分配单位，线程是CPU调度单位，协程是用户态并发单元。协程适合IO密集型高并发场景，用同步写法实现异步性能。

---

## 9. HashMap底层原理

### 数据结构（JDK 1.8+）

```
数组 + 链表 + 红黑树

索引位 → [Node0] → [Node1] → [Node2]  ← 链表（长度<8）
索引位 → [TreeNode] ← 红黑树（长度≥8，且数组长度≥64）
```

### 核心常量
| 常量 | 值 | 含义 |
|------|----|------|
| DEFAULT_INITIAL_CAPACITY | 16 | 默认初始容量（必须为2的幂） |
| DEFAULT_LOAD_FACTOR | 0.75 | 默认负载因子 |
| TREEIFY_THRESHOLD | 8 | 链表转红黑树阈值 |
| UNTREEIFY_THRESHOLD | 6 | 红黑树退化为链表阈值 |
| MIN_TREEIFY_CAPACITY | 64 | 树化最小数组容量 |

### put流程
```
1. 计算hash: (h = key.hashCode()) ^ (h >>> 16)  ← 高16位与低16位异或
2. 计算桶位置: index = (n - 1) & hash  ← 等价于 hash % n（n为2的幂）
3. 桶为空 → 直接放入
4. 桶非空:
   a. 判断头节点key是否相等 → 相等则覆盖
   b. 判断是否为红黑树节点 → 树操作
   c. 否则遍历链表:
      - 找到相等key → 覆盖
      - 到尾节点 → 尾插法
      - 插入后链表长度>=8 → treeifyBin()
5. size++，判断是否超过threshold（capacity * loadFactor）→ 扩容
```

### 扩容机制（resize）
- 容量变为原来的**2倍**
- 重新计算每个节点的桶位置：
  - 原位置不变（hash & oldCap == 0）
  - 新位置 = 原位置 + oldCap
- JDK1.8使用**高低位链表**优化迁移效率

### 为什么容量是2的幂？
1. `(n-1) & hash` 等价于 `hash % n`，位运算比取模快
2. 扩容时节点重分布只需判断新增bit位，迁移高效

### JDK 1.7 头插法 → 死循环问题
- 1.7在多线程扩容时，头插法导致链表成环 → CPU 100%
- 1.8改用尾插法避免了死循环，但仍不保证线程安全
- 并发场景应使用 `ConcurrentHashMap`

### 为什么线程不安全
- put时数据覆盖
- 扩容时的数据丢失
- size ++ 的非原子性

### 时间复杂度
- 理想情况（无冲突）：O(1)
- 链表查找：O(n) — 链表长度
- 红黑树查找：O(log n)

---

## 10. 手撕题：判断数组排序后能否组成等差数列

### 题目描述
给定一个整数数组`arr`，判断将数组重新排序后，是否可以使任意相邻两项的差相等，即能否形成一个等差数列。

**示例：**
- 输入：`arr = [3,5,1]` → 输出：`true`（排序后 [1,3,5] 公差为2）
- 输入：`arr = [1,2,4]` → 输出：`false`

### 解法一：排序后遍历（简单直观）
```python
def canMakeArithmeticProgression(arr):
    arr.sort()                          # O(n log n)
    diff = arr[1] - arr[0]
    for i in range(2, len(arr)):
        if arr[i] - arr[i-1] != diff:
            return False
    return True
```
- 时间复杂度：O(n log n)
- 空间复杂度：O(1)

### 解法二：数学公式法（O(n)，推荐）
```python
def canMakeArithmeticProgression(arr):
    n = len(arr)
    if n <= 2:
        return True
    
    min_val = min(arr)
    max_val = max(arr)
    
    # 公差必须为整数（整数数组）
    if (max_val - min_val) % (n - 1) != 0:
        return False
    
    diff = (max_val - min_val) // (n - 1)
    
    # 公差为0的特殊情况
    if diff == 0:
        return True  # 所有元素相等
    
    # 用Set检查每个元素是否在等差数列中
    seen = set()
    for num in arr:
        if (num - min_val) % diff != 0:  # 不是等差的整数倍
            return False
        if num in seen:                   # 重复元素
            return False
        seen.add(num)
    
    return len(seen) == n
```
- 时间复杂度：O(n)
- 空间复杂度：O(n)

### Java实现
```java
public boolean canMakeArithmeticProgression(int[] arr) {
    int n = arr.length;
    if (n <= 2) return true;
    
    int min = Integer.MAX_VALUE, max = Integer.MIN_VALUE;
    for (int num : arr) {
        min = Math.min(min, num);
        max = Math.max(max, num);
    }
    
    if ((max - min) % (n - 1) != 0) return false;
    
    int diff = (max - min) / (n - 1);
    if (diff == 0) return true;
    
    Set<Integer> set = new HashSet<>();
    for (int num : arr) {
        if ((num - min) % diff != 0) return false;
        if (!set.add(num)) return false; // add返回false表示重复
    }
    return true;
}
```

### 考点总结
- 等差数列的性质：`a_n = a_1 + (n-1)d`，首项+末项可反推公差
- Set判重技巧
- 对公差为0的特殊情况处理
- 数学优化的思路优于暴力排序

---

---

## 相关笔记

- [[AI Agent 面试题与答案]] — 完整索引页
- [[Agent 架构面试题-Agent核心篇]] — Agent 核心（10题）
- [[Agent 架构面试题-RAG篇]] — RAG 专题（10题 + 附录）
