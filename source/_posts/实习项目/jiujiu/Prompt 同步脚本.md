---
title: Prompt 同步脚本
date: 2026-06-12 15:27:46
categories:
  - 笔记
  - 实习项目
tags:
  - jiujiu
---
# Prompt 同步脚本

这些脚本用于将当前生效的 Prompt 模板同步到数据库中。
它们不会调用管理 API，也不会重启任何服务。

## 从服务器导出

拉取本目录后，先安装脚本依赖：

```
cd scripts/prompts && npm install
```

执行导出：

```
npm --prefix scripts/prompts run export
```

默认情况下，脚本会按顺序读取：`DATABASE_URL` → `.env.server` → 本地 Spring 数据源默认配置。导出文件会写入 `config/prompts/current.json`。

## 应用到开发环境

预览变更：

```
npm --prefix scripts/prompts run apply -- --dry-run
```

应用变更：

```
npm --prefix scripts/prompts run apply
```

默认情况下，脚本会按顺序读取：`DATABASE_URL` → `.env.local` → 本地 Spring 数据源默认配置。

## 开发环境启动时自动应用

`scripts/dev.cmd up|restart` 和 `scripts/dev.sh up|restart` 会在启动基础设施后、启动后端之前自动执行：

```
npm --prefix scripts/prompts run apply-if-present
```

如果 `config/prompts/current.json` 文件不存在或其中没有导出任何 prompt，该命令会直接跳过（视为成功）。

## 覆盖默认配置

需要使用其他环境文件或快照文件时：

```
npm --prefix scripts/prompts run export -- --env .env.server_test
npm --prefix scripts/prompts run apply -- --file config/prompts/current.json
```

直接使用 Postgres 连接字符串：

```
npm --prefix scripts/prompts run apply -- --database-url postgres://postgres:postgres@localhost:5432/aichat
```