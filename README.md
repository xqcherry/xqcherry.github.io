# Hexo Blog

这是一个 Hexo 源码仓库。文章和页面用 Markdown 维护，推送到 `master` 后由 GitHub Actions 自动构建并发布到 GitHub Pages。

## 常用命令

```bash
npm install
npm run server
npm run new -- "文章标题"
npm run build
```

## 内容位置

- 文章：`source/_posts/*.md`
- 关于页：`source/about/index.md`
- 资源页：`source/resources/index.md`
- 站点配置：`_config.yml`
- NexT 主题配置：`_config.next.yml`

## 发布

`.github/workflows/pages.yml` 会在推送到 `master` 时运行：

1. 安装依赖
2. 执行 `npm run build`
3. 上传 `public/`
4. 发布到 GitHub Pages

如果仓库的 Pages 设置还没有使用 GitHub Actions，需要在 GitHub 仓库的 Settings -> Pages 中选择 GitHub Actions。
