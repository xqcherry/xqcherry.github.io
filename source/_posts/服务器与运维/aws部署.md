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

## 迁移盘点记录（2026-07-21）

### 迁移目标

将旧 AWS 账号的全部资源迁移到新账号；旧账号关闭后，现有服务应继续按原有方式运行。迁移采用新旧并行、最终短暂维护同步数据、DNS 切换的方式。

### 已确认：新 AWS 账号

信息来自当前已登录的 AWS Console（仅只读查看）：

```text
账号名称：Jiujiu Love
账号 ID：430689517894
当前 Region：us-east-1
已有 EC2 实例 ID：i-09df9915ca7cf0961
实例名称：Jiujiu
界面线索显示实例类型：t4g.micro
```

### 当前限制与待确认项

```text
本机未安装 AWS CLI。
AWS Console 的 EC2 实例详情内容受本机代理/防火墙影响，无法加载；因此尚未确认新实例的公网 IP、私有 IP、VPC、子网、安全组、EBS 磁盘、AMI、密钥对和 IAM Role。
CloudShell 入口未发现可用环境。
尚未开始修改、创建、停止或删除任何 AWS 资源。
```

### 后续盘点顺序

1. SSH 只读检查新服务器的系统、磁盘、端口、Docker、nginx 和现有部署。
2. 通过旧账号和旧服务器盘点全部 AWS 资源、应用配置及数据。
3. 完成备份与恢复验证后，再开始新账号重建和切换。

### 已确认：旧 AWS 账号生产资源

信息来自 `old-aws-cloud-inventory-20260721-113635` 的只读 CloudShell 导出：

```text
旧账号 ID：459858401176
生产 Region：us-east-1
EC2：i-0826aa4bf4224d64a（running，t4g.xlarge）
公网 IPv4：100.53.168.188（未发现 Elastic IP；切换后 IP 会变化）
私网 IPv4：172.31.18.148
AMI：ami-0ab515046786de9dc
密钥对：snowie
IAM Instance Profile：无
VPC：vpc-091c77078acad1964
子网：subnet-0d99e2e1d4ade6023
EBS：vol-0b7342614b546fe1e，60 GiB，gp3，未加密
```

```text
Route 53 公有托管区：snowie-ai.com.（Z0813315ICM7GPZK1FSM，14 条记录）
CloudFront：E32L0461I331PZ，d2yszg8wf7offi.cloudfront.net，已部署
CloudFront 源站：origin.snowie-ai.com，HTTPS-only，TLS 1.2，读取超时 30 秒
CloudFront 额外依赖：WAF Web ACL CreatedByCloudFront-080db571（必须重建/迁移规则）
ACM：us-east-1，snowie-ai.com + *.snowie-ai.com，已签发、自动续期；不可导出，必须在新账号重新申请
S3：aws-cloudtrail-logs-459858401176-eeae3f6a（CloudTrail 日志桶，需在旧账号关闭前处理保留或导出）
```

尚缺：Route 53 的完整记录值、生产安全组的入/出站规则、WAF 规则详情、旧服务器内应用与数据库实际配置。

### 已确认：生产安全组与 WAF

生产安全组 `sg-0af6ffc7f2146206c`（`launch-wizard-1`）的当前规则：

```text
入站 IPv4：TCP 80（HTTP）来自 0.0.0.0/0
入站 IPv4：TCP 443（HTTPS）来自 0.0.0.0/0
入站 IPv4：TCP 22（SSH）来自 0.0.0.0/0
出站 IPv4：全部流量到 0.0.0.0/0
```

迁移时保留 Web 端口开放；SSH 不复制为全网开放，改为仅允许管理员固定公网 IP（或后续使用 SSM）。

CloudFront WAF Web ACL `CreatedByCloudFront-080db571` 的四条规则目前均为 **Count（计数）模式**，没有阻断流量：

```text
AWS-RateBasedRule-IP-300-CreatedByCloudFront
AWS-AWSManagedRulesAmazonIpReputationList
AWS-AWSManagedRulesCommonRuleSet
AWS-AWSManagedRulesKnownBadInputsRuleSet
```

迁移时在新账号以相同顺序和 Count 模式重建，确保行为不变。

### 已确认：完整 Route 53 DNS 记录

完整导出已保存为 `snowie-route53-records.json`。新账号创建托管区后，**不要复制旧区的 NS/SOA**（新托管区会自动生成）；其余记录按下列规则迁移：

```text
snowie-ai.com A Alias、www A Alias：改指向新 CloudFront 分发
origin.snowie-ai.com A、admin.snowie-ai.com A：改为新服务器公网 IP
MX、SPF、DMARC、Google verification、DKIM/SPF CNAME、pay CNAME：原样复制
```

需要原样保留的邮件/验证记录包括：

```text
apex MX -> Google Workspace（5 条）
apex TXT -> google-site-verification + SPF 委派
dc-aa8e722993._spfm TXT -> include:_spf.google.com
_dmarc TXT -> p=quarantine
fde._domainkey CNAME + fdesp CNAME -> GoDaddy 邮件服务
_domainconnect CNAME、pay CNAME
```

### 新账号迁移进度

已在新 AWS 账号创建 `snowie-ai.com` 公共 Route 53 托管区。新托管区的 Nameserver（**暂不在 GoDaddy 切换**）为：

```text
ns-202.awsdns-25.com.
ns-1667.awsdns-16.co.uk.
ns-557.awsdns-05.net.
ns-1463.awsdns-54.org.
```

已在旧账号的当前权威托管区新增 ACM DNS 验证 CNAME（不承载业务流量），用于验证新账号证书：

```text
_808b0329d22ac794fee4cafa4238be4d.snowie-ai.com.
  -> _7091effa8c70076afcb5d849f76bd2fc.jkddzztszm.acm-validations.aws.
```

该记录同时验证 `snowie-ai.com` 与 `*.snowie-ai.com`；等待新账号 ACM 状态变为 `Issued`。

新账号 ACM 证书现已签发，可用于之后创建新 CloudFront 分发；旧站仍由旧 CloudFront 和旧 Route 53 提供服务，尚未切流。

新账号已创建未关联资源的 CloudFront 全局 WAF Web ACL：`snowie-ai-cloudfront-waf`。规则顺序与旧 WAF 一致，且均为 Count 模式：

```text
1. AWS-RateBasedRule-IP-300-CreatedByCloudFront
2. AWS-AWSManagedRulesAmazonIpReputationList
3. AWS-AWSManagedRulesCommonRuleSet
4. AWS-AWSManagedRulesKnownBadInputsRuleSet
```

新账号 VPC `vpc-0e67ea384dbd2f794` 中已创建 `snowie-ai-web` 安全组，当前尚未绑定到实例：入站为 HTTP 80、HTTPS 443（公网）和 SSH 22（仅管理员当前公网 IP），出站允许全部流量。

新服务器实例类型已调整为 `t4g.large`，作为迁移期间的初始规格；上线后依据实际 CPU/内存使用情况决定是否升至旧生产环境的 `t4g.xlarge`。

新 Route 53 托管区已导入 8 组静态邮件、验证和支付记录；连同自动生成的 NS/SOA，当前共 10 条记录。主站、www、origin、admin 记录留待新服务与新 CloudFront 就绪后创建。

### 已确认：旧服务器部署形态

```text
系统：Ubuntu 24.04.4 LTS，ARM64（新服务器同为 Graviton/ARM，镜像与构建需使用 ARM64）
旧实例内存：15 GiB；新 t4g.large 上线后需重点监控内存
旧根盘：约 58 GiB，已用 40 GiB；新根盘 60 GiB 容量匹配
项目目录：/home/ubuntu/jiujiu-web
部署文件：docker-compose.yml；前端在 frontend/，管理端在 admin-frontend/
nginx：监听 80/443；主站代理至 Docker 暴露的 3000/8080，管理端代理至 2890
```

旧服务器 Docker 端口包括 3000、8080、2890、5432、6379 及 19001/19530/9091；新安全组仅开放 80/443/SSH，数据库与内部服务不应在新安全组开放公网端口。

### 已确认：Docker 服务与持久化数据

Compose 服务（均为 ARM64 镜像）：

```text
frontend（3000）
backend（8080）
admin-frontend（2890 -> 容器 80）
db（PostgreSQL 16，卷 jiujiu-web_pg_data）
redis（卷 jiujiu-web_redis_data；会话无需迁移）
minio（卷 7beae4... -> /data，以及 jiujiu-web_minio_data -> /minio_data）
etcd（卷 jiujiu-web_etcd_data）
milvus（卷 jiujiu-web_milvus_data）
```

迁移策略：先在新机部署相同 Compose 结构；最终维护窗口停止旧端的写入和 MinIO/Milvus/etcd/PostgreSQL 容器，导出 PostgreSQL、同步 MinIO 与 Milvus/etcd 卷，再在新机恢复并验证。Redis 作为会话缓存不迁移。

### 新服务器克隆状态与迁移调整

新服务器并非空白：已带有旧服务器约 8 天前的项目、Docker 镜像和全部持久化卷，但所有 Compose 容器目前停止。根盘已使用约 53 GiB（仅余约 5 GiB），必须先扩容才能安全构建和增量同步。

代码不能仅用 Git 拉取：旧机当前提交为 `ac8d110`，新机为旧提交 `c3278c6`；旧机还有未提交的 `config/prompts/current.json` 修改及未跟踪备份文件。后续从旧机以 rsync 做项目文件增量同步，并在最终维护窗口同步数据增量。

新机已清理中断同步留下的部分覆盖并从本地备份恢复，然后按要求执行 `git pull --ff-only`；当前新机提交为 `19868ac`（`dev model update`）。旧机仍为 `ac8d110`，因此在启动新机前必须明确选择：将旧机也升级到 `19868ac`，或将新机固定回 `ac8d110` 并单独同步旧机未提交的提示词修改。Docker 容器仍全部停止，未迁移数据。
