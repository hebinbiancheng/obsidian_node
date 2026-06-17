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

# 05 MySQL 日志系统

## 一句话结论

MySQL InnoDB 依赖三大日志保证数据可靠性：redo log（崩溃恢复 → 持久性）、undo log（回滚 + MVCC → 原子性）、binlog（主从复制 → 一致性），并通过两阶段提交解决 redo log 与 binlog 的一致性问题。

---

## 三大日志对比

| 维度 | redo log | binlog | undo log |
|------|----------|--------|----------|
| 所属层 | InnoDB 引擎层 | MySQL Server 层 | InnoDB 引擎层 |
| 日志类型 | **物理日志**（页修改） | **逻辑日志**（SQL 原始逻辑） | **逻辑日志**（逆向 SQL） |
| 核心作用 | 崩溃恢复（持久性） | 主从复制、数据备份 | 事务回滚 + MVCC |
| 写入方式 | 循环写（固定大小） | 追加写（无限增长） | 段（segment）管理 |
| 刷盘策略 | `innodb_flush_log_at_trx_commit` | `sync_binlog` | 随 redo log 持久化 |

---

## redo log（重做日志）

### 核心原理

```
更新数据 → Buffer Pool 中修改 → 记录到 redo log buffer → 刷盘到 redo log 文件
```

> 为什么不直接刷数据页？数据页 16KB，随机写，性能差；redo log 几十 Byte，顺序写，速度快。

### 刷盘时机

| 触发条件 | 说明 |
|----------|------|
| 事务提交 | 由 `innodb_flush_log_at_trx_commit` 控制 |
| log buffer 空间不足 | 达到约一半容量时 |
| 后台线程 | 每秒刷一次 |
| Checkpoint | 脏页刷盘时同步刷 redo log |
| 正常关闭 | MySQL 关闭时全部刷盘 |

### 刷盘策略（innodb_flush_log_at_trx_commit）

| 值 | 行为 | 性能 | 安全性 |
|:--:|------|:--:|:--:|
| 0 | 每秒刷一次（事务提交不刷） | 最高 | 可能丢 1 秒数据 |
| **1** | 每次事务提交都刷盘 | 最低 | 最安全，不丢数据 |
| 2 | 每次提交写 page cache，每秒刷盘 | 中等 | MySQL 挂不丢，宕机丢 1 秒 |

> 默认值为 1，保证持久性。

### 日志文件组（环形数组）

```
[write pos] → 当前写入位置
[checkpoint] → 当前擦除位置
```

- write pos 追上 checkpoint → 日志文件组满 → 停止写入，推进 checkpoint
- MySQL 8.0.30+：文件数固定 32，大小 = `innodb_redo_log_capacity / 32`

---

## binlog（归档日志）

### 核心作用

- 主从复制：从库通过 binlog 同步数据
- 数据恢复：基于时间点恢复数据
- 所有存储引擎共用（Server 层）

### 三种记录格式

| 格式 | 内容 | 优点 | 缺点 |
|------|------|------|------|
| **statement** | SQL 语句原文 | 日志量小 | 函数如 `now()` 导致主从不一致 |
| **row** | 行数据变更详情 | 数据一致性好 | 日志量大，IO 消耗高 |
| **mixed** | 自动选择 | 折中方案 | — |

> 通常推荐 `row` 格式，保证同步可靠性。

### 写入机制

```
事务执行 → 写入 binlog cache → 事务提交 → write(page cache) → fsync(磁盘)
```

`sync_binlog` 控制刷盘时机：

| 值 | 行为 |
|:--:|------|
| 0 | 只 write，系统决定 fsync 时机（可能丢日志） |
| **1** | 每次提交都 fsync（最安全） |
| N | 累积 N 个事务后 fsync（折中） |

---

## undo log（回滚日志）

### 核心作用

1. **事务回滚**：将数据恢复到修改前的状态（原子性）
2. **MVCC**：提供历史版本数据，实现非锁定一致性读

### 存储结构

```
rollback segment（回滚段）
  └── undo log segment（每个事务分配）
        └── undo log 记录
```

- 每个 rollback segment 有 1024 个 undo log segment
- 已提交事务的 undo log 进入 history list，由 purge 线程清理

### 两种类型

| 类型 | 产生场景 | 清理时机 |
|------|----------|----------|
| insert undo log | INSERT 操作 | 事务提交后直接删除 |
| update undo log | UPDATE / DELETE 操作 | 提交后进入 history list，purge 线程清理 |

### 版本链

不同事务对同一行的修改形成 undo log 链表，链首是最新记录，链尾是最早记录。MVCC 通过 `DB_ROLL_PTR` 沿链回溯找到可见版本。

---

## 两阶段提交

### 为什么需要？

redo log 和 binlog 写入时机不同：
- redo log：事务执行过程中不断写入
- binlog：仅在事务提交时写入

若不一致，主从数据会不一致。

### 两阶段提交流程

```
1. redo log 进入 prepare 阶段
2. 写入 binlog
3. redo log 进入 commit 阶段
```

### 异常恢复逻辑

| 异常场景 | 恢复行为 |
|----------|----------|
| redo log prepare 后、binlog 写入前崩溃 | 回滚事务（binlog 无记录） |
| binlog 写入后、redo log commit 前崩溃 | 提交事务（binlog 有记录，认为完整） |

---

## 更新语句完整流程

```
分析器 → 权限校验 → 执行器 → 引擎
  → 1. 从 Buffer Pool / 磁盘读取数据
  → 2. 在内存中修改数据
  → 3. 写入 redo log（prepare 状态）
  → 4. 写入 binlog
  → 5. redo log 改为 commit 状态
  → 6. 返回更新成功
```

---

## 常见问题

### Q1：为什么有了 binlog 还需要 redo log？

binlog 是 Server 层的逻辑日志，所有引擎共用，但不具备崩溃恢复（crash-safe）能力。redo log 是 InnoDB 特有的物理日志，记录数据页的物理修改，支持崩溃后恢复未刷盘的数据。两者职责不同：binlog 用于复制和归档，redo log 用于崩溃恢复。

### Q2：redo log 的 prepare 状态为什么不能省略？

如果先写 redo log 直接提交再写 binlog，redo log 提交后崩溃 → binlog 丢失 → 从库少数据。如果先写 binlog 再写 redo log，binlog 写入后崩溃 → redo log 丢失 → 主库无法恢复，但 binlog 有记录 → 主从不一致。prepare 状态让 MySQL 在恢复时能判断事务是否完整（binlog 是否已写入），从而决定提交还是回滚。

---

## 关联笔记

- [[00 SQL 知识库索引]] — 返回知识库索引
- [[04 MySQL 事务与隔离级别]] — ACID 与 MVCC
- [[06 SQL 执行过程与性能优化]] — 更新语句执行流程
- [[07 MySQL 面试题精粹]] — 日志相关面试题
