---
title: WireGuard 内网穿透部署流程总结
date: 2026-06-28 20:13:28
categories:
  - 服务器与运维
---

# WireGuard 内网穿透部署流程总结

最终结构

```
公网
 |
VPS（WireGuard Server）
10.0.0.1
 |
 +---- A服务器
 |     10.0.0.2
 |
 +---- B服务器
       10.0.0.3
```



## FLOW

### 目标服务器上

1. 安装 WireGuard

   ```
   sudo apt update
   sudo apt install wireguard
   ```

2. 生成密钥

   ```
   wg genkey | tee privatekey | wg pubkey > publickey
   ```

3. 查看公钥私钥

   ```
   cat privatekey
   cat publickey
   ```

4. 配置 B 的 WireGuard

   ```
   sudo nano /etc/wireguard/wg0.conf
   ```

   ```
   [Interface]
   PrivateKey = B私钥
   Address = 10.0.0.3/24
   
   
   [Peer]
   PublicKey = VPS公钥
   Endpoint = VPS公网IP:51820
   AllowedIPs = 10.0.0.0/24
   PersistentKeepalive = 25
   ```

5. 启动

   ```
   sudo systemctl enable wg-quick@wg0
   sudo systemctl start wg-quick@wg0
   ```

   ```
   sudo wg show
   ```

### 云服上

1. 添加服务器节点

   ```
   sudo nano /etc/wireguard/wg0.conf
   ```

   ```
   [Peer]
   PublicKey = B公钥
   AllowedIPs = 10.0.0.3/32
   ```

   ```
   sudo wg-quick down wg0
   sudo wg-quick up wg0
   ```

### 测试

分别ping对应网段即可

