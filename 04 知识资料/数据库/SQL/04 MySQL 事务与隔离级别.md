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

# 04 MySQL 事务与隔离级别

## 一句话结论

事务是数据库操作的最小执行单元，遵循 ACID 特性。MySQL InnoDB 默认 RR（可重复读）隔离级别，通过 MVCC（快照读）+ Next-Key Lock（当前读）解决了幻读问题。

---

## ACID 特性

| 特性 | 含义 | 实现机制 |
|------|------|----------|
| **原子性** (Atomicity) | 事务是不可分割的最小单元，要么全做，要么全不做 | undo log（回滚日志） |
| **一致性** (Consistency) | 事务前后数据保持一致状态 | A+I+D 共同保证，C 是目的 |
| **隔离性** (Isolation) | 并发事务间互不干扰 | MVCC + 锁 |
| **持久性** (Durability) | 事务提交后数据永久保存 | redo log（重做日志） |

> A、I、D 是手段，C 是目的。

---

## 四种隔离级别

| 级别 | 名称 | 脏读 | 不可重复读 | 幻读 |
|------|------|:--:|:--------:|:--:|
| RU | READ-UNCOMMITTED | ✅ | ✅ | ✅ |
| RC | READ-COMMITTED | ❌ | ✅ | ✅ |
| **RR** | **REPEATABLE-READ** | ❌ | ❌ | ❌（InnoDB） |
| Serializable | SERIALIZABLE | ❌ | ❌ | ❌ |

> MySQL InnoDB 默认 RR，且通过 MVCC + Next-Key Lock 解决了幻读。

---

## 并发事务问题

| 问题 | 描述 |
|------|------|
| **脏读** | 读到其他事务未提交的数据 |
| **丢失修改** | 两个事务同时修改，一个覆盖另一个 |
| **不可重复读** | 同一事务内两次读取同一条记录，内容不同（UPDATE/DELETE 导致） |
| **幻读** | 同一事务内两次查询，记录数量不同（INSERT 导致） |

### 不可重复读 vs 幻读

| 维度 | 不可重复读 | 幻读 |
|------|-----------|------|
| 重点 | 记录内容被修改或删除 | 记录数量增加 |
| 操作类型 | UPDATE / DELETE | INSERT |
| 解决方案 | 行锁（Record Lock） | 间隙锁（Gap Lock）→ Next-Key Lock |

---

## 并发控制方式

| 方式 | 模式 | 实现 |
|------|------|------|
| **锁** | 悲观控制 | 共享锁（S）/ 排他锁（X），表锁 / 行锁 |
| **MVCC** | 乐观控制 | 隐藏字段 + Read View + undo log |

### 锁类型

| 锁 | 说明 | 兼容性 |
|----|------|--------|
| 共享锁（S 锁） | 读锁，允许其他事务加 S 锁 | S 兼容 S，S 不兼容 X |
| 排他锁（X 锁） | 写锁，不允许其他事务加任何锁 | X 不兼容任何锁 |

### 锁粒度

| 粒度 | 说明 | 引擎 |
|------|------|------|
| 表级锁 | 锁整张表，并发低 | MyISAM、InnoDB |
| 行级锁 | 锁特定行，并发高 | InnoDB（默认） |

---

## MVCC 实现原理

MVCC（多版本并发控制）通过维护数据的多个版本，实现读写不冲突。

### 三大组件

| 组件 | 作用 |
|------|------|
| **隐藏字段** | `DB_TRX_ID`（最后修改事务 ID）、`DB_ROLL_PTR`（回滚指针）、`DB_ROW_ID` |
| **Read View** | 事务快照，记录当前活跃事务列表，用于可见性判断 |
| **undo log** | 存储数据的历史版本，形成版本链 |

### Read View 关键字段

| 字段 | 含义 |
|------|------|
| `m_ids` | 创建快照时活跃（未提交）的事务 ID 列表 |
| `m_low_limit_id` | 下一个待分配的事务 ID（大于等于此 ID 的不可见） |
| `m_up_limit_id` | m_ids 中最小的 ID（小于此 ID 的可见） |
| `m_creator_trx_id` | 创建此 Read View 的事务 ID |

### 可见性判断算法

```
1. DB_TRX_ID < m_up_limit_id        → 可见
2. DB_TRX_ID >= m_low_limit_id      → 不可见，沿 undo log 链回溯
3. m_ids 为空                        → 可见
4. m_up_limit_id <= DB_TRX_ID < m_low_limit_id
   → 查 m_ids：找到 → 不可见；未找到 → 可见
5. 不可见时沿 DB_ROLL_PTR 回溯 undo log，重新判断
```

---

## RC vs RR：Read View 生成时机

| 隔离级别 | Read View 生成时机 | 效果 |
|----------|-------------------|------|
| **RC** | 每次 SELECT 前都生成新的 | 可读到其他事务已提交的修改 → 不可重复读 |
| **RR** | 仅在事务第一次 SELECT 时生成 | 整个事务使用同一个快照 → 可重复读 |

---

## InnoDB RR 如何解决幻读

| 读类型 | 机制 | 效果 |
|--------|------|------|
| **快照读**（普通 SELECT） | MVCC：事务开始后第一次查询生成 Read View，后续不变 | 读不到其他事务插入的数据 |
| **当前读**（SELECT ... FOR UPDATE / INSERT / UPDATE / DELETE） | Next-Key Lock = Record Lock + Gap Lock | 锁定已有行 + 间隙，阻止插入 |

---

## 常见问题

### Q1：InnoDB 的 RR 隔离级别真的完全解决了幻读吗？

在**快照读**场景下，MVCC 保证不会出现幻读。但在**当前读**场景下，如果仅使用行锁，仍可能出现幻读——InnoDB 通过 Next-Key Lock（行锁+间隙锁）锁定查询范围内的间隙，阻止其他事务插入，从而在当前读下也解决了幻读。

### Q2：RC 和 RR 如何选择？

RC 每次读取都获取最新已提交数据，锁冲突少，适合高并发读多写少场景（多数数据库的默认级别）。RR 通过 MVCC 保证可重复读，适合需要事务内数据一致性的场景（如对账、报表）。MySQL 默认 RR，且 InnoDB 的 RR 实现解决了幻读，性能损失不大。

---

## 关联笔记

- [[00 SQL 知识库索引]] — 返回知识库索引
- [[03 MySQL 索引]] — 索引与锁的关系
- [[05 MySQL 日志系统]] — undo log / redo log 详解
- [[07 MySQL 面试题精粹]] — 事务相关面试题
