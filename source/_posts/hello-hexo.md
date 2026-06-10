---
title: Hello Hexo
date: 2026-06-10 10:00:00
categories:
  - 示例
tags:
  - Hexo
  - Markdown
---

这是一篇示例文章，用来确认博客已经从静态 HTML 迁移为 Hexo 源码结构。

以后新增文章时，把 Markdown 文件放在 `source/_posts/` 目录，并在文件开头保留 front matter：

```yaml
---
title: 文章标题
date: 2026-06-10 10:00:00
categories:
  - 分类名称
tags:
  - 标签名称
---
```

正文可以直接使用 Markdown 编写。推送到 GitHub 后，GitHub Actions 会自动生成并发布静态站点。
