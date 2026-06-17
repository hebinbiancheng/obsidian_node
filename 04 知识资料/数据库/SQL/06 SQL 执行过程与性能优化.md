---
type: knowledge
status: evergreen
tags:
  - 知识库
  - 数据库
  - SQL
  - 面试
confidence: medium
created: 2026-06-17
---

# 06 SQL 执行过程与性能优化

## 一句话结论

MySQL 采用 Server 层 + 存储引擎层的分层架构，一条 SQL 经过连接器 → 分析器 → 优化器 → 执行器 → 存储引擎的完整链路。通过 EXPLAIN 分析执行计划是性能优化的核心手段。

---

## MySQL 基础架构

```text
┌─────────────────────────────────────────┐
│              Server 层                    │
│  连接器 → 查询缓存 → 分析器 → 优化器 → 执行器  │
│  内置函数、存储过程、触发器、视图、binlog      │
├─────────────────────────────────────────┤
│           存储引擎层（插件式）               │
│  InnoDB │ MyISAM │ Memory │ ...          │
│  (redo log)                              │
└─────────────────────────────────────────┘
```

### Server 层组件

| 组件 | 职责 |
|------|------|
| **连接器** | 身份认证、权限校验、连接管理 |
| **查询缓存** | 缓存 SELECT 结果（MySQL 8.0 已移除） |
| **分析器** | 词法分析（提取关键字）+ 语法分析（校验语法） |
| **优化器** | 选择最优执行方案（索引选择、JOIN 顺序） |
| **执行器** | 校验权限 → 调用引擎接口 → 返回结果 |

---

## 查询语句执行流程

```text
权限校验 → 查询缓存（8.0 跳过）→ 分析器 → 优化器 → 权限校验 → 执行器 → 存储引擎
```

示例：`SELECT * FROM tb_student WHERE age='18' AND name='张三';`

1. 权限校验（无权限直接报错）
2. 查询缓存（8.0 前，命中直接返回）
3. 分析器：词法分析（提取 SELECT、表名、条件）→ 语法分析
4. 优化器：选择最优索引和查询顺序
5. 执行器：再次权限校验 → 调用引擎接口 → 返回结果

---

## 更新语句执行流程

```text
分析器 → 权限校验 → 执行器 → 引擎
  → Buffer Pool 中修改数据
  → 记录 redo log（prepare）
  → 记录 binlog
  → redo log（commit）
  → 返回成功
```

示例：`UPDATE tb_student SET age='19' WHERE name='张三';`

> 更新语句不走查询缓存，且会导致相关表的查询缓存全部失效。

---

## EXPLAIN 执行计划

### 核心字段

| 字段 | 含义 | 关注点 |
|------|------|--------|
| `id` | SELECT 序号 | 值越大优先级越高；相同则从上到下 |
| `select_type` | 查询类型 | SIMPLE / PRIMARY / SUBQUERY / DERIVED / UNION |
| `table` | 访问的表 | — |
| **`type`** | **访问类型** | 性能关键指标 |
| `possible_keys` | 可能使用的索引 | NULL 表示无可用索引 |
| **`key`** | **实际使用的索引** | NULL 表示未使用索引 |
| `key_len` | 索引使用长度 | 越短越好 |
| `rows` | 预估扫描行数 | 越少越好 |
| **`Extra`** | **额外信息** | 关注 Using filesort / temporary |

### type 性能排序（优 → 差）

```
system > const > eq_ref > ref > range > index > ALL
```

| type | 含义 | 示例 |
|------|------|------|
| `system` | 表仅一行（const 特例） | 系统表 |
| `const` | 主键/唯一索引等值匹配，最多一行 | `WHERE id = 1` |
| `eq_ref` | 连表时，驱动表每行在被驱动表仅匹配一行 | 主键/唯一索引 JOIN |
| `ref` | 普通索引等值匹配，可能多行 | `WHERE name = '张三'` |
| `range` | 索引范围扫描 | `WHERE id > 10` |
| `index` | 全索引扫描 | 遍历整个索引树 |
| `ALL` | 全表扫描 | 性能最差 |

### Extra 关键值

| 值 | 含义 | 建议 |
|----|------|------|
| `Using index` | 覆盖索引，无需回表 ✅ | 最佳 |
| `Using index condition` | 索引下推 ✅ | 较好 |
| `Using where` | WHERE 过滤 | 正常 |
| `Using filesort` | 外部排序 ⚠️ | 需优化 |
| `Using temporary` | 使用临时表 ⚠️ | 需优化 |
| `Using join buffer` | 连表无索引 ⚠️ | 需优化 |

---

## 常见优化手段

| 场景 | 优化方案 |
|------|----------|
| 慢查询 | EXPLAIN 分析 → 加索引 / 改写 SQL |
| 深度分页 | 游标分页 / 延迟关联 |
| 大表 JOIN | 小表驱动大表 / 索引优化 |
| ORDER BY 慢 | 利用索引排序（索引已有序） |
| GROUP BY 慢 | 加联合索引 / 减少分组字段 |
| COUNT 慢 | MyISAM 直接取 / InnoDB 用覆盖索引 |
| IN vs EXISTS | 外层大表用 IN，内层大表用 EXISTS |

---

## 常见问题

### Q1：一条 SQL 查询很慢，如何排查？

1. 用 `EXPLAIN` 查看执行计划，关注 type（是否为 ALL/index）、key（是否用到索引）、rows（扫描行数）、Extra（是否有 filesort/temporary）
2. 检查是否索引失效（LIKE '%x'、函数/计算、隐式转换、OR 条件）
3. 用 `SHOW PROFILE` 分析各阶段耗时
4. 检查慢查询日志，定位具体 SQL

### Q2：MySQL 8.0 为什么移除查询缓存？

查询缓存以 SQL 语句为 key，任何对表的更新都会使该表所有缓存失效。在写密集场景下，缓存频繁失效，维护成本高于收益。MySQL 8.0 彻底移除，建议在应用层（如 Redis）做缓存。

---

## 关联笔记

- [[00 SQL 知识库索引]] — 返回知识库索引
- [[03 MySQL 索引]] — 索引优化详解
- [[05 MySQL 日志系统]] — 更新语句中的日志流程
- [[07 MySQL 面试题精粹]] — 优化相关面试题
