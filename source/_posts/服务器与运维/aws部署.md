---
title: aws部署
date: 2026-07-21 22:06:04
categories:
  - 服务器与运维
---

下面按一次真实访问来复盘你现在的 `snowie-ai.com` 链路。

**1. 用户输入域名**
用户访问：

```text
https://snowie-ai.com
https://www.snowie-ai.com
```

浏览器第一步不是访问服务器，而是先问 DNS：

```text
snowie-ai.com 这个域名对应哪里？
```

**2. GoDaddy 只负责注册，不负责解析**
`snowie-ai.com` 是在 GoDaddy 买的域名，但 GoDaddy 页面显示：

```text
DNS 提供商：Amazon Route 53
```

意思是：域名还在 GoDaddy 名下，但 DNS 解析权已经交给 AWS Route 53。
所以以后你要改 `snowie-ai.com` 的 DNS，主要去 AWS Route 53，不是在 GoDaddy 的 DNS 表里改。

**3. Route 53 决定流量去 CloudFront**
Route 53 里有这些关键记录：

```text
snowie-ai.com      A Alias -> d2yszg8wf7offi.cloudfront.net
www.snowie-ai.com  A Alias -> d2yszg8wf7offi.cloudfront.net
origin.snowie-ai.com A -> 100.53.168.188
admin.snowie-ai.com  A -> 100.53.168.188
```

重点是前两条：

```text
snowie-ai.com -> CloudFront
www.snowie-ai.com -> CloudFront
```

所以用户访问主站时，不是直接打到服务器 IP，而是先进入 CloudFront。

**4. CloudFront 是网站入口**
CloudFront 分发信息：

```text
Distribution ID: E32L0461I331PZ
CloudFront 域名: d2yszg8wf7offi.cloudfront.net
备用域名:
  snowie-ai.com
  www.snowie-ai.com
源站:
  origin.snowie-ai.com
```

CloudFront 做几件事：

```text
HTTPS 证书
全球 CDN 加速
HTTP -> HTTPS 跳转
缓存静态内容
把动态请求转发给源站
```

你的证书在 ACM 里，证书域名是 `snowie-ai.com`。CloudFront 用自定义域名时，ACM 证书必须在 `us-east-1`，这一点你现在也是这样配置的。

**5. CloudFront 怎么分流请求**
你现在有 6 条行为规则：

```text
/api/*        -> 不缓存，转发给源站
/video/*      -> 缓存优化
*.*           -> 使用源站 Cache-Control
/_next/image* -> Next.js 图片优化策略
/ws/*         -> 不缓存，适合 WebSocket
默认 (*)      -> 不缓存
```

这说明它不是一个纯静态网站，而是一个 Next.js 应用。

可以这样理解：

```text
页面 HTML          -> CloudFront 转发/缓存策略处理
JS/CSS/图片       -> CloudFront 缓存
/api/*            -> 必须回源，因为是动态接口
/ws/*             -> 必须回源，因为是 WebSocket
/_next/image*     -> Next.js 图片优化
```

**6. CloudFront 回源到 origin**
CloudFront 的源站是：

```text
origin.snowie-ai.com
```

Route 53 里它解析到：

```text
100.53.168.188
```

所以实际链路是：

```text
用户
 -> snowie-ai.com
 -> Route 53
 -> CloudFront
 -> origin.snowie-ai.com
 -> 100.53.168.188
```

**7. 源站服务器跑什么**
我查到响应头：

```text
Server: nginx/1.24.0 (Ubuntu)
X-Powered-By: Next.js
x-nextjs-cache: HIT
Via: CloudFront
X-Cache: Miss from cloudfront
```

这说明源站大概率是：

```text
Ubuntu 服务器
nginx
Next.js 应用
```

典型内部结构是：

```text
CloudFront
  -> nginx :80 或 :443
      -> Next.js :3000
```

nginx 的作用通常是：

```text
监听 80/443
反向代理到 Next.js
处理 gzip、header、上传大小限制等
可能也处理 SSL，如果 CloudFront 到源站用 HTTPS
```

Next.js 应用负责真正生成页面、处理 API、WebSocket 等。

**8. 为什么还要 origin 子域名**
不用直接把 CloudFront 源站写成 IP，而是写：

```text
origin.snowie-ai.com
```

好处是以后服务器换 IP，只需要改 Route 53 的：

```text
origin.snowie-ai.com A 新 IP
```

CloudFront 配置不用动。

这是一种比较标准的做法。

**9. admin 子域名是什么**
Route 53 里还有：

```text
admin.snowie-ai.com -> 100.53.168.188
```

它没有走 CloudFront，而是直接指向源站 IP。

这通常有几种可能：

```text
后台管理入口
服务器管理面板
内部 admin 应用
绕过 CloudFront 的调试入口
```

这个我只能从 DNS 推断，还没进入页面验证。

**10. 邮箱相关记录**
`snowie-ai.com` 还有：

```text
MX -> Google 邮箱
TXT -> google-site-verification
TXT -> SPF
_dmarc -> DMARC
```

这些和网站部署无关，是邮箱和域名验证用的。

简单讲：

```text
MX       决定收邮件去哪里
SPF      告诉别人哪些服务器可以代表你发邮件
DMARC    邮件防伪策略
TXT      各种第三方平台验证域名所有权
```

**当前架构图**

```text
GoDaddy
  只负责注册 snowie-ai.com
  Nameserver 指向 AWS Route 53

Route 53
  snowie-ai.com       -> CloudFront
  www.snowie-ai.com   -> CloudFront
  origin.snowie-ai.com -> 100.53.168.188
  admin.snowie-ai.com  -> 100.53.168.188
  MX/TXT/DMARC         -> 邮箱和验证

CloudFront
  域名: d2yszg8wf7offi.cloudfront.net
  绑定: snowie-ai.com, www.snowie-ai.com
  证书: ACM us-east-1 snowie-ai.com
  源站: origin.snowie-ai.com
  行为:
    /api/* 不缓存
    /ws/* 不缓存
    静态资源按规则缓存
    HTTP 跳 HTTPS

源站服务器
  IP: 100.53.168.188
  OS: Ubuntu
  Web server: nginx/1.24.0
  App: Next.js
```

**你以后自己部署时照这个顺序理解**

```text
1. 服务器先能跑起来
2. nginx 能把请求转给 Next.js
3. origin 子域名指向服务器 IP
4. CloudFront 指向 origin 子域名
5. ACM 证书绑定到 CloudFront
6. 主域名和 www 指向 CloudFront
7. GoDaddy Nameserver 指向 Route 53
```

最核心的一句话：

```text
GoDaddy 管域名归属，Route 53 管解析，CloudFront 管入口和 HTTPS，nginx/Next.js 管真正的网站。
```

---
