---
title: TCP三次握手
date: 2026-00-00
categories:
  - 八股
tags:
  - 计网
---

## TCP三次握手

![](/images/计网/1.png)

1. 第一次握手：客户端发SYN报文(SYN = 1), 携带随机序列号(seq = x)
2. 第二次握手：服务端收到SYN后，回发一个SYN+ACK报文(SYN = 1, ACK = 1)，确认号是客户端序列号+1(ack = x+1)，同时携带自己的随机序列号(seq = y)
3. 第三次握手：客户端收到SYN+ACK后，发送一个ACK报文，确认号是服务端序列号+1(ack = y+1)