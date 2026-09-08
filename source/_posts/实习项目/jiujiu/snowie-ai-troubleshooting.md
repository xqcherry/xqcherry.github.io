---
title: snowie-ai-troubleshooting
date: 2026-07-12 17:12:31
categories:
  - 实习项目
tags:
  - jiujiu
---
# 故障排查实战笔记

## 场景一：502 Bad Gateway（后端挂了）

### 现象
用户访问网站，页面空白，几秒后显示 `502 Bad Gateway`。

### 排查过程

**第 1 步：判断问题在哪**

502 = nginx 到后端连接被拒，后端大概率挂了。

**第 2 步：确认后端是否在运行**

```bash
docker ps                          # 看哪些容器在跑
docker ps -a | grep postgres       # 连数据库容器也一起看
```

发现后端容器不在运行列表中。

**第 3 步：拉日志看为什么崩了**

```bash
docker logs --since 24h aichat-backend > aichat-backend_24h.logs 2>&1
```

日志显示 `PrismaClientInitializationError: Can't reach database server`。

**第 4 步：问题转移 —— 数据库连不上，查数据库容器**

```bash
docker ps -a | grep postgres
```

发现 PostgreSQL 容器也挂了，退出码 `137`。

**退出码含义：**
- 137 = 128 + 9 = 被 SIGKILL 杀掉（不是程序自己崩的，是系统强杀）

**第 5 步：确认是不是 OOM**

```bash
dmesg | grep -i "out of memory" | tail -5
```

看到 `Out of memory: Killed process ... (postgres)`。

**第 6 步：查内存**

```bash
free -h
```

2G 内存几乎用满，Swap 为 0。

### 根因链

```
用户看到 502
  → nginx 到后端不通
  → 后端容器崩溃（连不上数据库）
  → PostgreSQL 容器被 OOM Killer 杀掉
  → 服务器内存 2G 用满，没有 Swap
```

### 修复

- **短期：** `docker start aichat-db` 重启数据库
- **长期：** 加 Swap 或者升内存，配置容器内存限制

---

## 场景二：登录后白屏，未登录正常

### 现象
未登录的页面正常，登录后一直加载，最后白屏，没有报错弹窗。

### 排查过程

**第 1 步：看浏览器控制台**

用户按 F12，看 Network 标签：

```
/api/auth/session  → 200 OK（正常，走 Next.js :3000）
/api/auth/me       → 504 Gateway Timeout（超时，走后端 :8080）
```

定位到 `/api/auth/me` 接口超时。

**第 2 步：curl 直接测后端接口**

```bash
curl -w "\n时间: %{time_total}s\n" http://127.0.0.1:8080/api/auth/me
```

返回 `{"error":"database timeout"}`，耗时 31 秒。

CloudFront 默认超时 30 秒，所以用户看到 504。

**第 3 步：查数据库正在执行的查询**

```bash
docker exec -it aichat-db psql -U postgres -c \
  "SELECT pid, now() - query_start AS duration, query FROM pg_stat_activity WHERE state = 'active' ORDER BY duration DESC;"
```

发现有查询跑了 28 秒还没结束。

**第 4 步：查有没有锁**

```bash
docker exec -it aichat-db psql -U postgres -c \
  "SELECT pid, locktype, relation::regclass, mode, granted FROM pg_locks WHERE NOT granted;"
```

`users` 表有锁在等。

**第 5 步：找谁锁的**

```bash
docker exec -it aichat-db psql -U postgres -c \
  "SELECT pid, now() - query_start AS duration, query FROM pg_stat_activity WHERE state = 'active';"
```

发现一个 `UPDATE users SET last_active = now()` 跑了 2 小时没结束，持有表锁。

**第 6 步：短期修复，杀掉卡住的查询**

```bash
docker exec -it aichat-db psql -U postgres -c "SELECT pg_terminate_backend(9999);"
```

### 根因链

```
登录后白屏
  → /api/auth/me 超时
  → 后端查 users 表被锁住
  → 一个 UPDATE 跑了 2 小时没结束，持有表锁
  → 所有读 users 表的请求都在排队
```

### 修复

- **短期：** `pg_terminate_backend()` 杀掉卡住的查询
- **长期：** 查这个 UPDATE 为什么慢，加索引或加条件限制

---

## 排查通用思路

```
页面打不开
  → 502？后端挂了，查 docker ps 和 logs
  → 504？后端慢，curl 测接口耗时
  → 白屏没报错？看浏览器 Network 找到哪个请求挂了

接口慢
  → 查数据库 pg_stat_activity
  → 有没有锁？查 pg_locks
  → 谁锁的？找 duration 最长的查询

容器挂了
  → 退出码 137？OOM，查 dmesg | grep "out of memory"
  → 退出码 1？程序自己崩了，看 docker logs
  → 退出码 0？正常退出，查是不是被 docker stop 了

数据库连不上
  → ping 域名，DNS 对不对
  → docker ps -a，数据库容器在不在
  → 退出码是什么，决定下一步查什么
```

## 常用排查命令速查

```bash
# 容器相关
docker ps                              # 看运行中的容器
docker ps -a                           # 包括已停止的
docker logs --since 24h <name>         # 拉最近 24 小时日志
docker inspect <name> | grep -i db     # 看容器环境变量
docker stats                           # 实时看容器 CPU/内存

# 端口和进程
ss -tlnp                               # 看哪些端口在监听
curl http://127.0.0.1:3000             # 直接测本地服务
curl -w "\n时间: %{time_total}s\n" URL # 测接口带耗时

# 系统资源
free -h                                # 看内存
dmesg | grep -i "out of memory"        # 查 OOM 记录
df -h                                  # 看磁盘

# 数据库
docker exec -it aichat-db psql -U postgres -c "SELECT ..."  # 执行 SQL
pg_stat_activity                       # 当前活跃查询
pg_locks                               # 锁信息
pg_terminate_backend(pid)              # 杀掉卡住的查询

# 网络
dig snowie-ai.com                      # DNS 解析
curl -I https://snowie-ai.com          # 测 CloudFront
curl -I https://origin.snowie-ai.com   # 测源站
curl -I -H "Host: snowie-ai.com" https://100.53.168.188  # 测 IP 直连

# Nginx
nginx -t                               # 测配置语法
tail -f /var/log/nginx/error.log       # 看错误日志
```

---

## 场景三：部分用户打不开网站（DNS 层问题）

### 现象
部分用户反馈打不开网站，浏览器显示 `DNS_PROBE_FINISHED_NXDOMAIN`，但你自己访问正常。

### 排查过程

**第 1 步：确认域名本身没问题**

```bash
dig snowie-ai.com @8.8.8.8   # 用 Google DNS 查，不走用户本地 DNS
```

返回正常 IP，说明域名没丢。

**第 2 步：确认你自己的 DNS 也正常**

```bash
dig snowie-ai.com
```

正常，排除你自己网络的问题。

**第 3 步：让用户查他的 DNS**

让用户在自己电脑上运行：

```
nslookup snowie-ai.com
```

看到 `DNS request timed out` 或者 `Server: 192.168.1.1`（路由器 DNS），说明用户的 DNS 服务器有问题。

**第 4 步：让用户换 DNS 测试**

```
nslookup snowie-ai.com 8.8.8.8
```

换了 Google DNS 就正常了，确认是用户本地 DNS 的问题。

**第 5 步：如果换 DNS 也不行，查是否被污染**

```bash
dig snowie-ai.com @8.8.8.8 +short      # Google DNS → 13.224.xxx.xxx（正常）
dig snowie-ai.com @114.114.114.114 +short  # 114 DNS → 0.0.0.0（被污染）
dig snowie-ai.com @223.5.5.5 +short    # 阿里 DNS → 0.0.0.0（被污染）
```

**第 6 步：确认 Route 53 记录没被改**

登 AWS Console 看 Route 53，A 记录正常指向 CloudFront。

### 根因链

```
部分用户打不开
  → 浏览器报 DNS_PROBE_FINISHED_NXDOMAIN 或无法访问
  → dig 对比不同 DNS 服务器
  → 国内 DNS 返回 0.0.0.0，Google DNS 返回正常 IP
  → Route 53 记录正常
  → DNS 被污染（运营商层面劫持）
```

### 修复

- **给用户：** DNS 改成 `8.8.8.8`（Google）或 `1.1.1.1`（Cloudflare）
- **自己：** 这个问题不在你服务器，是运营商 DNS 的问题

---

## DNS 相关错误速查

| 现象 | 含义 | 问题在哪 |
|------|------|---------|
| `DNS_PROBE_FINISHED_NXDOMAIN` | 域名不存在 | DNS 记录丢了或本地 DNS 有问题 |
| `ERR_CONNECTION_TIMED_OUT` | DNS 能解析，但服务器连不上 | 防火墙/安全组没开端口 |
| `ERR_CONNECTION_REFUSED` | 连上了但被拒 | 端口没开或服务没跑 |
| dig 返回正常 IP | DNS 正常 | 问题在别的层 |
| dig 返回 `0.0.0.0` | DNS 被污染或域名被暂停 | 查不同 DNS 对比确认 |
| dig 返回 `NXDOMAIN` | 域名不存在 | 查 Route 53 记录是否还在 |
| dig 无响应 | DNS 服务器挂了 | DNS 服务本身的问题 |
| 不同 DNS 返回不同结果 | 配置没同步或有污染 | 对比 8.8.8.8 和 114.114.114.114 |

## DNS 排查命令

```bash
# 指定 DNS 服务器查询（绕过本地 DNS）
dig snowie-ai.com @8.8.8.8 +short        # Google
dig snowie-ai.com @1.1.1.1 +short        # Cloudflare
dig snowie-ai.com @114.114.114.114 +short # 国内 114
dig snowie-ai.com @223.5.5.5 +short      # 阿里

# 查 TTL
dig snowie-ai.com +noall +answer          # 看 TTL 值，300=5分钟缓存

# 用户端排查（Windows）
nslookup snowie-ai.com                    # 用默认 DNS 查
nslookup snowie-ai.com 8.8.8.8           # 换 DNS 查
```

## 浏览器错误码速查

| 用户看到的 | 问题在哪 |
|-----------|---------|
| `DNS_PROBE_FINISHED_NXDOMAIN` | DNS 解析失败，域名查不到 |
| `ERR_CONNECTION_TIMED_OUT` | DNS 能解析，但服务器连不上 |
| `ERR_CONNECTION_REFUSED` | 连上了但被拒，端口没开或服务没跑 |
| `502 Bad Gateway` | nginx 到后端不通 |
| `504 Gateway Timeout` | 后端太慢 |
| `503 Service Unavailable` | 服务过载或维护中 |
| `413 Request Entity Too Large` | 上传文件超过 nginx 的 client_max_body_size |
