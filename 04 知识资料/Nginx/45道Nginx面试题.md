---
title: 45道Nginx面试题（附答案）
date: 2026-07-02
tags: [nginx, 面试题, 运维, 负载均衡, 反向代理]
category: 知识资料
source: https://mp.weixin.qq.com/s/XaVEKKySHLxSlUxBakdjdw
---

# 45 道 Nginx 面试题（附答案）

> 原文: [微信文章](https://mp.weixin.qq.com/s/XaVEKKySHLxSlUxBakdjdw)

---

## 基础篇

### 1. 你知道的 Web 服务有哪些？

Apache、Nginx、IIS、Tomcat、Lighttpd、WebLogic。

### 2. 为什么要用 Nginx？

- **跨平台、配置简单**
- **高并发**：处理 2-3 万并发，官方监测支持 5 万
- **内存消耗小**：开启 10 个仅占 150M
- **内置健康检查**：宕机自动切换
- **节省带宽**：支持 GZIP 压缩、浏览器缓存
- **稳定性高**：宕机概率极小

### 3. Nginx 性能为什么比 Apache 高？

Nginx 采用 **epoll/kqueue** 模型，Apache 采用 **select** 模型。

> select 版宿管大妈挨个房间找人；epoll 版宿管大妈直接记住房号。

### 4. epoll 的三个函数

```c
int epoll_create(int size);   // 创建句柄
int epoll_ctl(...);           // 注册监听事件
int epoll_wait(...);          // 等待事件
```

### 5. Nginx 和 Apache 的核心区别

| | Nginx | Apache |
|---|-------|--------|
| 模型 | 异步非阻塞 | 同步多进程 |
| 并发 | 一个进程处理万级连接 | 一个连接一个进程 |
| 静态文件 | 比 Apache 高 3 倍+ | 较弱 |
| 动态请求 | 需配合后端 | 直接支持 PHP |
| 适用 | 高性能场景 | 稳定、兼容性好 |

---

## 命令与配置

### 6. Nginx 常用命令

```bash
nginx                    # 启动
nginx -s stop            # 快速停止
nginx -s quit            # 优雅停止
nginx -s reload          # 重载配置（平滑重启）
nginx -c /path/nginx.conf  # 指定配置文件
nginx -v                 # 版本
nginx -t                 # 检查配置文件语法
nginx -h                 # 帮助
```

### 19. 默认配置文件结构

```
全局配置（无{}包裹）
events {}        # 连接配置
http {
    server {}    # 虚拟主机
    location {}  # URI 匹配
}
```

### 20. Location 匹配规则

| 修饰符 | 含义 |
|--------|------|
| 留空 | 前缀匹配 |
| `=` | 精确匹配 |
| `~` | 大小写敏感正则 |
| `~*` | 大小写不敏感正则 |
| `^~` | 前缀匹配，不检查正则 |

优先级：`= > ^~ > ~ / ~* > 前缀匹配`

---

## 代理与负载均衡

### 7. 正向代理 vs 反向代理

| | 正向代理 | 反向代理 |
|---|---------|---------|
| 位置 | 客户端 | 服务端 |
| 代理对象 | 代理客户端 | 代理服务端 |
| 用途 | 隐藏客户端 IP、访问控制 | 负载均衡、缓存、安全 |

### 22. 负载均衡分发策略

| 策略 | 说明 |
|------|------|
| **轮询**（默认）| 逐一分配 |
| **权重** (weight) | 按权重比例分配 |
| **ip_hash** | 同 IP 固定到同一后端（解决 session） |
| **fair**（第三方）| 按响应时间分配 |
| **url_hash**（第三方）| 按 URL hash 分配 |

### 24. 四层负载均衡

需启用 `--with-stream` 编译参数，在 `stream {}` 块中配置。

### 14. Session 不同步怎么办？

- **ip_hash**：同 IP 固定到同一后端
- **spring_session + Redis**：session 放缓存共享

### 11. 后端健康检查

- 方式一：`ngx_http_proxy_module` + `ngx_http_upstream_module`
- 方式二：`nginx_upstream_check_module`（推荐）

---

## 优化篇

### 13. Nginx 优化清单

- Gzip 压缩、expires 缓存、epoll 事件模型
- 隐藏版本号、防盗链、禁止 IP 访问
- 防 DOS（`limit_conn`、`limit_req`）
- 日志切割、错误页面定制
- FastCGI buffer/cache 优化
- SSL 优化
- Linux 内核参数优化

### 关键参数

```nginx
worker_processes auto;           # CPU 核数
worker_cpu_affinity auto;        # CPU 绑定
worker_rlimit_nofile 65535;      # 最大文件描述符
use epoll;
worker_connections 65535;        # 每进程最大连接数
keepalive_timeout 65;
```

---

## 状态码速查

| 状态码 | 含义 |
|--------|------|
| 200 | 成功 |
| 301 | 永久重定向 |
| 302 | 临时重定向 |
| 304 | 未修改（缓存） |
| 400 | 错误请求 |
| 403 | 禁止访问 |
| 404 | 未找到 |
| 499 | 客户端主动断开 |
| 500 | 服务器内部错误 |
| 502 | 错误网关（后端挂了） |
| 503 | 服务不可用 |
| 504 | 网关超时 |

---

## 常用模块

| 模块 | 作用 |
|------|------|
| `ngx_http_core_module` | 核心 HTTP 配置 |
| `ngx_http_access_module` | 访问控制 |
| `ngx_http_gzip_module` | 压缩 |
| `ngx_http_proxy_module` | 反向代理 |
| `ngx_http_upstream_module` | 负载均衡 |
| `ngx_http_rewrite_module` | URL 重写 |
| `ngx_http_limit_conn_module` | 连接数限制 |
| `ngx_http_limit_req_module` | 请求速率限制 |
| `ngx_http_ssl_module` | SSL/TLS |
| `ngx_http_stub_status_module` | 状态监控 |

---

## 高可用

### 44. 高可用方案

- **Keepalived + Nginx**：虚拟 IP 漂移
- **Heartbeat + Nginx**：心跳检测 + 资源接管

---

## 相关概念

- **动静分离**：静态资源 Nginx 直接返回，动态请求转发后端
- **防盗链**：`valid_referers` 过滤非信任来源
- **跨域 CORS**：`add_header Access-Control-Allow-Origin`
- **Master/Worker 模型**：Master 管理，Worker 处理请求
- **缓存流程**：首次请求 → 后端获取 → Nginx 缓存 → 后续直接返回
