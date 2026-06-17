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

# 01 SQL 基础与语法

## 一句话结论

SQL 是操作关系型数据库的标准语言，分为 DDL（定义）、DML（操纵）、DCL（控制）、TCL（事务控制）四大类，核心查询能力包括 SELECT 子句、JOIN 连接、子查询、UNION 组合、视图和聚合函数。

---

## SQL 四大分类

| 分类 | 全称 | 核心指令 | 功能 |
|------|------|----------|------|
| DDL | Data Definition Language | `CREATE` `ALTER` `DROP` `TRUNCATE` | 定义数据库对象（表、索引、视图） |
| DML | Data Manipulation Language | `SELECT` `INSERT` `UPDATE` `DELETE` | 访问和操纵数据（CRUD） |
| DCL | Data Control Language | `GRANT` `REVOKE` | 控制用户访问权限 |
| TCL | Transaction Control Language | `COMMIT` `ROLLBACK` | 管理事务 |

> DDL 操作立即生效，不可回滚，不触发 trigger；DML 操作需事务提交后生效，可回滚。

---

## SELECT 查询核心

### 基本语法与子句执行顺序

```text
SELECT → DISTINCT → FROM → JOIN → ON → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
```

| 子句 | 作用 | 要点 |
|------|------|------|
| `DISTINCT` | 去重 | 作用于所有列，所有列相同才算重复 |
| `WHERE` | 行过滤 | 在 GROUP BY 前执行，不能使用聚合函数 |
| `GROUP BY` | 分组汇总 | 常配合 COUNT/SUM/AVG/MAX/MIN |
| `HAVING` | 分组过滤 | 在 GROUP BY 后执行，可使用聚合函数 |
| `ORDER BY` | 排序 | ASC（默认）/ DESC；支持多列排序 |
| `LIMIT` | 限制行数 | `LIMIT offset, count`，offset 从 0 开始 |

### WHERE vs HAVING

| 维度 | WHERE | HAVING |
|------|-------|--------|
| 过滤对象 | 行（原始记录） | 分组（聚合结果） |
| 聚合函数 | ❌ 不可用 | ✅ 可用 |
| 执行时机 | GROUP BY 之前 | GROUP BY 之后 |
| 能否单独使用 | ✅ | ❌（一般配合 GROUP BY） |

---

## JOIN 连接

### 连接类型

| 类型 | 说明 |
|------|------|
| `INNER JOIN` | 返回两表匹配的行（默认 JOIN） |
| `LEFT JOIN` | 返回左表所有行，右表无匹配为 NULL |
| `RIGHT JOIN` | 返回右表所有行，左表无匹配为 NULL |
| `FULL OUTER JOIN` | 返回两表所有行（MySQL 不直接支持） |
| `CROSS JOIN` | 笛卡尔积 |

### ON vs WHERE

- `ON`：连接条件，决定临时表的生成
- `WHERE`：在临时表生成后，对临时表进行筛选
- 执行顺序：先 ON 生成临时表 → 再 WHERE 筛选

### JOIN vs UNION

| 维度 | JOIN | UNION |
|------|------|-------|
| 方向 | 水平（列合并） | 垂直（行合并） |
| 列要求 | 列可以不同 | 列数和列顺序必须相同 |
| 结果 | 笛卡尔积 | 去重合并（UNION ALL 不去重） |

---

## 子查询

子查询是嵌套在较大查询中的 SQL 查询，必须放在括号 `()` 内。

| 位置 | 用途 | 返回要求 |
|------|------|----------|
| `WHERE` 子句 | 作为过滤条件 | 单行单列 / 多行单列 / 单行多列 |
| `FROM` 子句 | 作为临时表 | 多行多列，需用 `AS` 起别名 |

### 常用操作符

| 操作符 | 作用 |
|--------|------|
| `IN` | 匹配指定列表中的任意值 |
| `BETWEEN` | 匹配范围内的值 |
| `LIKE` | 模式匹配（`%` 任意字符，`_` 单个字符） |
| `AND` / `OR` / `NOT` | 逻辑组合（AND 优先级高于 OR） |

> ⚠️ `LIKE '%xxx'` 以 `%` 开头会导致索引失效，查询极慢。

---

## 视图（VIEW）

| 维度 | 说明 |
|------|------|
| 本质 | 基于 SQL 结果集的虚拟表，不存储数据 |
| 作用 | 简化复杂查询、权限控制、数据格式转换 |
| 限制 | 不能建索引，不能有触发器 |

---

## 常见问题

### Q1：`WHERE` 和 `HAVING` 的根本区别是什么？

`WHERE` 在数据分组前过滤行，不能使用聚合函数；`HAVING` 在分组后过滤分组结果，可以使用聚合函数。`WHERE` 可单独使用，`HAVING` 一般配合 `GROUP BY`。

### Q2：`LEFT JOIN` 后加 `WHERE` 条件，和加 `ON` 条件有什么区别？

`ON` 条件影响 JOIN 时临时表的生成——右表不满足 ON 条件的行不会参与连接。`WHERE` 在临时表生成后过滤——如果 WHERE 过滤了右表字段为 NULL 的行，实际上把 LEFT JOIN 变成了 INNER JOIN 的效果。

---

## 关联笔记

- [[00 SQL 知识库索引]] — 返回知识库索引
- [[02 数据库设计与范式]] — 数据库设计基础
- [[06 SQL 执行过程与性能优化]] — SQL 执行全流程
