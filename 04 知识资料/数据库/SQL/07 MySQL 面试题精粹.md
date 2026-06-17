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

# 07 MySQL 面试题精粹

## 一句话结论

MySQL 面试高频考点集中在：存储引擎对比（InnoDB vs MyISAM）、字段类型选择、锁机制、索引优化、事务隔离级别和日志系统。本章汇总常见面试题及核心要点。

---

## MySQL 基础

### Q：什么是关系型数据库？

建立在关系模型上的数据库，数据之间存在一对一、一对多、多对多的联系。常见的有 MySQL、PostgreSQL、Oracle、SQL Server、SQLite。

### Q：MySQL 有什么优点？

- 成熟稳定，开源免费
- 文档丰富，开箱即用
- 兼容性好，支持多种 OS 和开发语言
- InnoDB 事务支持优秀，RR 级别可解决幻读
- 支持分库分表、读写分离、高可用

---

## 字段类型

### 字段类型分类

| 分类 | 类型 | 常用 |
|------|------|------|
| 数值 | TINYINT / SMALLINT / MEDIUMINT / INT / BIGINT / FLOAT / DOUBLE / DECIMAL | INT、DECIMAL |
| 字符串 | CHAR / VARCHAR / TEXT / BLOB | CHAR、VARCHAR |
| 日期时间 | YEAR / TIME / DATE / DATETIME / TIMESTAMP | DATETIME、TIMESTAMP |

### CHAR vs VARCHAR

| 维度 | CHAR | VARCHAR |
|------|------|---------|
| 长度 | 定长 | 变长 |
| 存储 | 右边填充空格，检索时去掉 | 1-2 字节记录长度 |
| 适用 | 定长数据（MD5、身份证） | 变长数据（昵称、标题） |
| 内存 | 按定义长度分配 | 按定义长度分配（排序时消耗更多） |

### DECIMAL vs FLOAT/DOUBLE

| 维度 | DECIMAL | FLOAT/DOUBLE |
|------|---------|--------------|
| 类型 | 定点数 | 浮点数 |
| 精度 | 精确 | 近似 |
| 适用 | 金额等精度敏感场景 | 科学计算 |

### DATETIME vs TIMESTAMP

| 维度 | DATETIME | TIMESTAMP |
|------|----------|-----------|
| 时区 | 无关 | 相关 |
| 存储空间 | 8 字节 | 4 字节 |
| 时间范围 | 1000-9999 年 | 1970-2038 年 |

### NULL vs ''

| 维度 | NULL | ''（空字符串） |
|------|------|---------------|
| 含义 | 不确定的值 | 长度为 0 的字符串 |
| 空间 | 占用空间 | 不占用 |
| 比较 | `IS NULL` / `IS NOT NULL` | `=` `!=` 等比较运算符 |
| 聚合函数 | 被 SUM/AVG/COUNT(列名) 忽略 | 正常参与 |
| 相等性 | `NULL = NULL` → false | `'' = ''` → true |

> ⚠️ 不建议用 NULL 作为列默认值。

### 不推荐 TEXT 和 BLOB 的原因

- 不能有默认值
- 临时表只能用磁盘临时表
- 检索效率低
- 不能直接创建索引（需指定前缀长度）
- 消耗大量网络和 IO 带宽
- 可能导致 DML 操作变慢

---

## 存储引擎

### InnoDB vs MyISAM

| 维度 | InnoDB | MyISAM |
|------|--------|--------|
| 行级锁 | ✅ 支持 | ❌ 仅表锁 |
| 事务 | ✅ 支持（ACID） | ❌ 不支持 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | ✅ 支持（redo log） | ❌ 不支持 |
| MVCC | ✅ 支持 | ❌ 不支持 |
| 索引实现 | 聚簇索引（数据即索引） | 非聚簇索引（索引数据分离） |
| 缓存 | Buffer Pool（数据+索引） | Key Cache（仅索引） |
| 并发性能 | 高（行锁） | 低（表锁） |
| 默认引擎 | ✅（MySQL 5.5+） | ❌ |

> 绝大多数场景选择 InnoDB。MyISAM 仅在读密集、不需要事务的极简单场景下考虑。

---

## 锁机制

### 锁分类

| 维度 | 类型 |
|------|------|
| 读写 | 共享锁（S）/ 排他锁（X） |
| 粒度 | 表级锁 / 行级锁 |
| 算法 | Record Lock / Gap Lock / Next-Key Lock |

### 行锁类型（InnoDB）

| 锁 | 说明 |
|----|------|
| Record Lock | 锁定单行记录 |
| Gap Lock | 锁定索引间隙（防止插入） |
| Next-Key Lock | Record + Gap，锁定行及其前间隙 |

### 意向锁

表级锁，表明事务稍后要对表中的行加哪种锁（IS 意向共享 / IX 意向排他），用于快速判断表级锁与行级锁的兼容性。

---

## 性能优化

### 常见 SQL 优化手段

1. **避免 SELECT \***：只查需要的字段，利用覆盖索引
2. **合理使用索引**：WHERE / JOIN / ORDER BY 字段建索引
3. **避免索引失效**：函数/计算/隐式转换/LIKE '%x'/OR 无索引
4. **小表驱动大表**：JOIN 时小表在前
5. **分页优化**：深度分页用游标或延迟关联
6. **批量操作**：批量 INSERT 代替逐条插入
7. **避免大事务**：拆分为小事务，减少锁持有时间

### 深度分页优化

```sql
-- 不推荐：OFFSET 越大越慢
SELECT * FROM t LIMIT 1000000, 10;

-- 推荐：游标分页（基于有序主键）
SELECT * FROM t WHERE id > 1000000 LIMIT 10;

-- 或延迟关联
SELECT * FROM t
INNER JOIN (SELECT id FROM t LIMIT 1000000, 10) tmp USING(id);
```

### 数据冷热分离

- 热数据：近期频繁访问 → 存高性能存储（如 SSD / Redis）
- 冷数据：历史归档数据 → 存低成本存储
- 实现：按时间分区表 / 应用层路由 / 中间件

---

## 常见问题

### Q1：MySQL 如何存储 IP 地址？

推荐使用 `INT UNSIGNED` 存储 IPv4（4 字节），通过 `INET_ATON()` 和 `INET_NTOA()` 转换。比 VARCHAR(15) 节省空间，且支持范围查询。

### Q2：能用 MySQL 直接存储文件（图片、音视频）吗？

不推荐。文件存储会导致：表空间膨胀、备份恢复缓慢、大量网络 IO。推荐方案：文件存对象存储（OSS/S3），MySQL 只存文件 URL。

---

## 关联笔记

- [[00 SQL 知识库索引]] — 返回知识库索引
- [[02 数据库设计与范式]] — 数据库设计
- [[03 MySQL 索引]] — 索引详解
- [[04 MySQL 事务与隔离级别]] — 事务与锁
- [[05 MySQL 日志系统]] — 三大日志
- [[06 SQL 执行过程与性能优化]] — 性能优化
