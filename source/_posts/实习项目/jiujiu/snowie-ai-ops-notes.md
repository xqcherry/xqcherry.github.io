---
title: snowie-ai-ops-notes
date: 2026-07-12 16:57:31
categories:
  - 笔记
  - 实习项目
tags:
  - jiujiu
---
# snowie-ai.com 架构与运维笔记

## 一、完整请求链路

```
用户访问 snowie-ai.com
  → GoDaddy（域名注册，Nameserver 指向 Route 53）
  → AWS Route 53（DNS 解析）
  → CloudFront（CDN、HTTPS 证书、缓存）
  → origin.snowie-ai.com → 100.53.168.188
  → nginx :443（反向代理、分流）
      → /ws/*         → 后端 :8080
      → /api/auth/signin|callback|csrf|session|providers|signout → Next.js :3000
      → /api/auth/sync|me|logout → 后端 :8080
      → /api/chat/stream → 后端 :8080（流式响应）
      → /api/*        → 后端 :8080
      → /*            → Next.js :3000
```

**一句话：** GoDaddy 管归属，Route 53 管解析，CloudFront 管入口和加速，nginx 管分流，Next.js 管页面，后端管业务逻辑。

---

## 二、各服务职责

| 服务 | 端口 | 职责 |
|------|------|------|
| nginx | 80/443 | 反向代理、HTTP→HTTPS、请求分流 |
| Next.js | 3000 | 前端页面渲染、NextAuth 登录流程 |
| 后端 | 8080 | 业务 API、WebSocket、用户鉴权 |
| Admin | 2890 | 管理后台（独立子域名，直连不走 CloudFront） |

---

## 三、Nginx Location 匹配规则

### 优先级（高→低）

```
= 精确匹配 > ^~ 前缀不查正则 > ~ 正则匹配 > / 普通前缀
```

### 各类型说明

**精确匹配 `=`**
```nginx
location = /api/auth/me { ... }
```
路径必须完全等于 `/api/auth/me`，多一个字符都不行。

**前缀匹配 `^~`**
```nginx
location ^~ /.well-known/acme-challenge/ { ... }
```
以指定路径开头就匹配，匹配后不再检查正则。用于 Let's Encrypt 证书验证。

**正则匹配 `~`**
```nginx
location ~ ^/api/auth/(signin|callback|csrf|session|providers|signout)(/|$) { ... }
```
用正则表达式匹配，`~` 区分大小写，`~*` 不区分大小写。

**普通前缀 `/`**
```nginx
location / { ... }
```
兜底规则，所有没被匹配到的请求都走这里。

### 匹配示例

| 请求路径 | 匹配的 location | 转发到 |
|---------|----------------|--------|
| `/api/auth/me` | `= /api/auth/me` | :8080 |
| `/api/auth/signin` | `~ ^/api/auth/(signin\|...)` | :3000 |
| `/api/users` | `/api/` | :8080 |
| `/api/chat/stream` | `/api/chat/stream`（前缀匹配在 /api/ 后面，但不以 /api/ 开头的不抢） | :8080 |
| `/dashboard` | `/` | :3000 |

---

## 四、proxy_pass 反向代理

### 必备 Header

```nginx
proxy_set_header Host $host;                    # 原始域名
proxy_set_header X-Real-IP $remote_addr;        # 真实用户 IP
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # 代理链 IP
proxy_set_header X-Forwarded-Proto $scheme;     # 原始协议 https
```

**去掉会怎样：** 后端不知道用户是 HTTPS 访问，可能重定向到 `http://127.0.0.1:3000`，用户看到报错。

### WebSocket 专用

```nginx
proxy_http_version 1.1;          # 必须 HTTP/1.1
proxy_set_header Upgrade $http_upgrade;         # 升级头
proxy_set_header Connection "upgrade";          # 告诉后端要升级连接
proxy_read_timeout 3600s;        # 1小时没数据才断
proxy_buffering off;             # 不缓冲，实时转发
```

**去掉 Upgrade/Connection：** WebSocket 握手失败，客户端收到 400。

### 流式响应专用

```nginx
proxy_buffering off;    # 不缓冲，后端每产生一点数据就立即发给客户端
proxy_cache off;        # 不缓存，每次都是新内容
```

**默认 buffering on：** AI 回复会卡住不动，然后一次性全部弹出来。

---

## 五、两套证书

| 证书 | 用途 | 位置 | 续期方式 |
|------|------|------|---------|
| ACM 证书 | CloudFront 用 | us-east-1 | AWS 自动续期 |
| Let's Encrypt | nginx 用 | /etc/letsencrypt/live/snowie-ai.com/ | Certbot 自动续期 |

**ACM 硬性要求：** CloudFront 绑定的证书必须在 us-east-1 创建，其他 region 的证书选不到。

**检查证书状态：**
```bash
certbot certificates           # 看到期时间
certbot renew --dry-run        # 测试自动续期能不能跑通
```

---

## 六、排查故障流程

### 用户说"网站打不开"

按这个顺序一层层查：

**第 1 步：DNS 解析**
```bash
dig snowie-ai.com              # 看是否指向 CloudFront
```

**第 2 步：CloudFront 到源站**
```bash
curl -I https://snowie-ai.com              # 测 CloudFront
curl -I https://origin.snowie-ai.com       # 绕过 CloudFront 直接测源站
curl -I -H "Host: snowie-ai.com" https://100.53.168.188  # 绕过域名直连 IP
```

三种结果判断：
- 三个都挂 → 源站有问题
- 前两个挂，第三个通 → DNS 或 CloudFront 有问题
- 只有第一个挂 → CloudFront 配置有问题

**第 3 步：看 HTTP 状态码**

| 状态码 | 问题在哪 |
|--------|---------|
| DNS 解析失败 | Route 53 记录丢了 |
| 连接超时 | 服务器安全组/防火墙没开端口 |
| 502 Bad Gateway | nginx 到后端连接被拒，后端没启动 |
| 504 Gateway Timeout | 后端启动了但响应太慢 |
| 403 Forbidden | 权限问题 |
| 413 Entity Too Large | 上传超过 client_max_body_size |

**第 4 步：nginx 状态**
```bash
nginx -t                       # 测配置语法
systemctl status nginx         # 看是否在运行
tail -f /var/log/nginx/error.log  # 看错误日志
```

**第 5 步：后端服务**
```bash
ss -tlnp                       # 看哪些端口在监听
curl http://127.0.0.1:3000     # Next.js 在不在
curl http://127.0.0.1:8080     # 后端在不在
```

应该看到：
```
LISTEN  *:80    → nginx
LISTEN  *:443   → nginx
LISTEN  *:3000  → Next.js
LISTEN  *:8080  → 后端
LISTEN  *:2890  → Admin
```

缺了哪个就是哪个没启动。

**第 6 步：systemd 服务管理**
```bash
# 查看服务状态
systemctl status snowie-ai
systemctl status snowie-admin

# 找不到服务名时
systemctl list-units --type=service | grep -i snow

# 重启服务
systemctl restart snowie-ai

# 看日志
journalctl -u snowie-ai -f
journalctl -u snowie-ai --since "10 minutes ago"
```

**第 7 步：证书过期**
```bash
certbot certificates
certbot renew --dry-run
```

---

## 七、CloudFront 配置

**Distribution ID:** E32L0461I331PZ
**CloudFront 域名:** d2yszg8wf7offi.cloudfront.net
**备用域名:** snowie-ai.com, www.snowie-ai.com
**源站:** origin.snowie-ai.com

### 行为规则

| 路径 | 缓存策略 |
|------|---------|
| /api/* | 不缓存，转发给源站 |
| /video/* | 缓存优化 |
| *.* | 使用源站 Cache-Control |
| /_next/image* | Next.js 图片优化策略 |
| /ws/* | 不缓存，适合 WebSocket |
| 默认 (*) | 不缓存 |

### 缓存刷新

```bash
aws cloudfront create-invalidation \
  --distribution-id E32L0461I331PZ \
  --paths "/*"
```

每月前 1000 条免费，超过要收费。

---

## 八、待优化项

1. **nginx.conf 的 SSL 协议** — 主配置里还有 TLSv1/1.1，应改成只留 TLSv1.2/1.3
2. **/api/chat/stream** — 建议改成 `^~` 或 `=` 匹配，防止以后加正则规则时被抢走
3. **admin 的 client_max_body_size** — 没配，默认 1m，上传大文件会 413
4. **Gzip** — 主配置里 gzip_types 被注释了，JSON/JS/CSS 没有压缩
