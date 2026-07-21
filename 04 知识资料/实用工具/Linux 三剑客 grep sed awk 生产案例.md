---
title: Linux 三剑客精讲：grep sed awk 生产常用案例
type: knowledge
category: 实用工具
status: evergreen
source_type: 微信公众号
created: 2026-07-17
tags:
  - Linux
  - grep
  - sed
  - awk
  - 运维
  - 生产实战
source: https://mp.weixin.qq.com/s/T1evSswGL-1mBMPCx6oU7w
confidence: high
---

# Linux 三剑客精讲：grep sed awk 生产常用案例

> 原文：[微信文章](https://mp.weixin.qq.com/s/T1evSswGL-1mBMPCx6oU7w) · 2026-07-17
> 原始资料：`^[raw/articles/wechat-linux-three-swords-2026.html]`

---

## 一句话总结

查找用 **grep**，改文件用 **sed**，统计分析用 **awk**。三者组合管道可解决 99% 的 Linux 文本处理场景。

---

## 一、核心定位

| 工具 | 定位 | 用途 |
|------|------|------|
| **grep** | 筛选过滤 | 找内容——精准匹配、日志筛选、报错排查 |
| **sed** | 流式编辑 | 改内容——批量替换、删行、改配置（无需打开文件） |
| **awk** | 数据分析 | 拆统计——拆分字段、求和、格式化输出 |

---

## 二、grep 生产实战

### 基础高频

```bash
grep -i "error" app.log           # 忽略大小写
grep -A5 -B5 "ERROR" app.log      # 显示前后5行上下文（排查堆栈）
grep -c "Exception" app.log       # 统计报错总量
```

### 正则匹配

```bash
grep "^ERROR" app.log             # ERROR 开头的行
grep "[0-9]$" app.log             # 数字结尾的行
grep -v "^#" nginx.conf | grep -v "^$"  # 过滤注释和空行
```

### 批量多文件

```bash
grep "超时" *.log
```

---

## 三、sed 生产实战

### 替换

```bash
sed 's/80/8080/g' nginx.conf           # 预览（不改文件）
sed -i 's/localhost/127.0.0.1/g' application.yml  # 直接修改
sed -i 's/192\.168\.1\.100/192.168.1.101/g' host.conf  # 特殊字符转义
```

### 删除

```bash
sed -i '/debug/d' app.log     # 删除含 debug 的行
sed -i '1d' data.txt          # 删第一行
sed -i '$d' data.txt          # 删最后一行
sed -i '/^$/d' app.log        # 删空行
```

### 实用技巧

```bash
sed -i 's/^log/#log/g' application.properties  # 注释所有 log 配置
```

> ⚠️ 生产避坑：先不带 `-i` 预览，确认无误再真改。

---

## 四、awk 生产实战

### 字段提取

```bash
awk '{print $1}' access.log            # 第1列（IP）
awk '{print $1,$NF}' access.log        # 第1列 + 最后一列
awk -F ',' '{print $2,$3}' data.csv    # CSV 自定义分隔符
```

### 统计分析

```bash
# 访问量 Top10 IP
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head -10

# 各状态码分布
awk '{print $9}' access.log | sort | uniq -c
```

### 数据计算

```bash
# 响应时间总和与平均
awk '{sum+=$10;count++} END{print "总和:",sum,"平均:",sum/count}' access.log
```

### 条件筛选

```bash
awk '/ERROR/ {print $1,$2,$NF}' app.log
```

---

## 五、组合实战

```bash
# 过滤 ERROR，统计 Top10 报错内容
grep "ERROR" app.log | grep -v "^$" | awk '{print $NF}' | sort | uniq -c | sort -nr

# 清洗配置文件：去注释、去空行、去空格
grep -v "^#" nginx.conf | grep -v "^$" | sed 's/ //g'

# 日志转 CSV
sed 's/ /,/g' access.log | awk '{print $1","$4","$7","$9}' > access.csv
```

---

## 相关笔记

- [[K8s 生产故障全景图 8大场景速查]] — 八域排查速查
- [[后端面试/CDN 缓存原理与面试题]] — Nginx 缓存
