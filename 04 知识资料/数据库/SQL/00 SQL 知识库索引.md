---
title: SQL 知识库索引
source: "[[04 知识资料/面试与技术复习/戴斌0805 XMind 解析]]"
created: 2026-06-12
updated: 2026-06-17
tags:
  - 知识库
  - 数据库
  - SQL
  - 面试
  - 知识库索引
status: evergreen
confidence: medium
---

# SQL 知识库索引

> 来源：[[04 知识资料/面试与技术复习/戴斌0805 XMind 解析|戴斌0805 XMind 完整解析]]
> 原始文件（1031 行）已拆分为结构化知识库，原单体文件归档至 [[07 已归档/原始笔记/SQL|已归档/SQL]]。

## 一句话总结

SQL（Structured Query Language）是关系型数据库的标准语言，涵盖 DDL/DML/DCL/TCL 四类操作，以 MySQL 为核心深入索引（B+Tree）、事务（ACID/MVCC）、日志（redo log/binlog/undo log）、执行过程与性能优化，是后端开发与面试的核心知识域。

## 知识库目录

1. [[01 SQL 基础与语法]] — SQL 分类、SELECT 查询、JOIN 连接、UNION 组合、视图、函数
2. [[02 数据库设计与范式]] — ER 图、范式（1NF/2NF/3NF）、主键外键、drop/delete/truncate
3. [[03 MySQL 索引]] — 索引数据结构、B+Tree、聚簇/非聚簇、覆盖索引、最左前缀、索引下推
4. [[04 MySQL 事务与隔离级别]] — ACID、四种隔离级别、MVCC 实现、并发问题
5. [[05 MySQL 日志系统]] — redo log、binlog、undo log、两阶段提交
6. [[06 SQL 执行过程与性能优化]] — 基础架构、查询/更新流程、EXPLAIN 执行计划
7. [[07 MySQL 面试题精粹]] — 高频面试题：字段类型、存储引擎、锁、优化

## 核心地图

```text
SQL / MySQL
├── SQL 基础
│   ├── DDL：CREATE / ALTER / DROP / TRUNCATE
│   ├── DML：SELECT / INSERT / UPDATE / DELETE
│   ├── DCL：GRANT / REVOKE
│   └── TCL：COMMIT / ROLLBACK
├── 数据库设计
│   ├── ER 图 → 关系模型
│   ├── 范式：1NF → 2NF → 3NF
│   └── 主键 / 外键 / 级联
├── 索引
│   ├── 数据结构：Hash → BST → AVL → 红黑树 → B+Tree
│   ├── 类型：聚簇 / 非聚簇 / 覆盖 / 联合 / 前缀
│   ├── 原则：最左前缀 / 索引下推
│   └── 优化：避免失效 / EXPLAIN 分析
├── 事务
│   ├── ACID：原子性 / 一致性 / 隔离性 / 持久性
│   ├── 隔离级别：RU → RC → RR → Serializable
│   └── MVCC：隐藏字段 + Read View + undo log
├── 日志
│   ├── redo log：物理日志，崩溃恢复
│   ├── binlog：逻辑日志，主从复制
│   ├── undo log：逻辑日志，事务回滚 + MVCC
│   └── 两阶段提交：redo prepare → binlog → redo commit
└── 执行过程
    ├── Server 层：连接器 → 分析器 → 优化器 → 执行器
    ├── 存储引擎层：InnoDB / MyISAM
    └── EXPLAIN：type / key / rows / Extra
```

## 精读优先级

### 高频必看
- [[03 MySQL 索引]] — 面试必问，B+Tree 与索引优化
- [[04 MySQL 事务与隔离级别]] — ACID、MVCC、隔离级别对比
- [[05 MySQL 日志系统]] — redo log vs binlog、两阶段提交
- [[06 SQL 执行过程与性能优化]] — 一条 SQL 的执行全流程

### 基础巩固
- [[01 SQL 基础与语法]] — SQL 分类、JOIN、子查询
- [[02 数据库设计与范式]] — 范式理论、主键外键设计

### 面试冲刺
- [[07 MySQL 面试题精粹]] — 高频面试题速查

## 适合沉淀的问题

- SQL 语句的执行流程是怎样的？（从连接器到存储引擎）
- B+Tree 为什么适合做数据库索引？和 B 树、Hash、红黑树相比优势在哪？
- 聚簇索引和非聚簇索引的区别？什么是回表？
- 最左前缀匹配原则是什么？联合索引 (a,b,c) 下 `WHERE b=1 AND c=1` 会走索引吗？
- MySQL 的四种隔离级别分别解决什么问题？InnoDB 的 RR 如何解决幻读？
- redo log 和 binlog 的区别？为什么需要两阶段提交？
- MVCC 的实现原理？RC 和 RR 下 Read View 的生成时机有何不同？
- EXPLAIN 执行计划中 type 字段各值的含义和优劣排序？
- MyISAM 和 InnoDB 的区别？
- 索引失效的常见场景有哪些？

## 关联笔记

- [[04 知识资料/数据库/Redis/00 Redis 知识库索引|Redis 知识库]]
- [[04 知识资料/面试与技术复习/面试与技术复习索引|面试与技术复习索引]]
- [[04 知识资料/知识库总索引|知识库总索引]]
