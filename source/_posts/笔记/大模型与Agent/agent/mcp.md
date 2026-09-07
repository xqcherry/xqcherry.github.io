---
title: mcp
date: 2026-04-12 23:16:45
categories:
  - 笔记
  - 大模型与Agent
tags:
  - agent
---
## 什么是MCP协议

全称：**Model Context Protocol**（模型上下文协议）

mcp协议是一个让AI应用与外部数据源，服务器之间通信的协议，使得AI应用可以安全，拓展的访问外部文件，数据库，web应用，api等上下文，避免工具调用部分的碎片化和私有集成



## MCP的架构的组成部分

MCP架构包含三个组成部分

- MCP Host：运行AI模型并管理连接的客户端，决定何时调用工具
- MCP Server：暴露本地或远程资源的程序，通过 MCP 协议向 Host 提供 **Prompts** (预设提示词)、**Resources** (只读数据) 和 **Tools** (可执行函数)
- MCP客户端：将 Host 的意图翻译成符合 MCP 标准的 JSON-RPC 指令,维持与多个 MCP Server 的长连接或进程间通信,通常作为模块内嵌在MCP Host



## MCP使用的底层通信协议与消息格式

- 消息格式：**JSON-RPC 2.0:** MCP 的应用层协议基于 JSON-RPC，这使得它具有良好的跨语言兼容性
- 通信方式：支持**stdio（标准输入输出，适合本地进程）** 或 **HTTP + SSE（服务器发送事件，适合远程）**



## 解释 Resources, Prompts, 和 Tools 的区别

- Resources：**只读**的上下文信息（如文档、代码库、日志）
- Prompts：预定义的模板，用于引导模型执行特定任务
- Tools：**可执行**的函数，模型可以调用它们来改变世界，类似api接口



## 有了 MCP 为什么还需要 RAG？它解决了什么痛点？

- **标准化**：以前每个 Agent 都要单独适配 GitHub、Slack、DB 的接口。有了 MCP，开发者只需写一次 Server，任何支持 MCP 的 Client 都能直接使用

- **解耦:** 将复杂的业务逻辑（如 SQL 拼接、文件过滤）封装在 Server 端，Host 端只需关注调度逻辑



## 在生产环境下，MCP 如何保证数据安全？

- **权限沙箱:** MCP Server 通常作为独立进程运行。Host 可以通过进程隔离限制其访问范围
- **本地优先:** MCP 强调本地集成，敏感数据（如企业内部文档）不需要上传到云端处理，直接在本地通过 stdio 传输给 Host，Host 决定发送给模型的内容



## 为什么这样分？（架构设计的艺术）

- **解耦 ：** **Server** 不关心你是用 Claude 还是 DeepSeek 调用它。它只管遵循 MCP 协议。
- **安全 ：** **Host** 运行在你的环境里，它是“守门人”。如果模型想删掉你整个数据库，**Host/Client 层**可以设置权限拦截，不把指令发给 **Server**。
- **复用 ：** 你写好一个“校园文档 RAG Server”，它就像一个 **USB 打印机**。任何支持 MCP 协议的 **Host**（不管是同学的 VS Code 还是你的机器人）只要插上（配置一下路径），就能立刻获得这个能力。

