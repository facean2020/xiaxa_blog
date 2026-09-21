# xiaxa_blog

个人 Hugo 博客，使用 [Hugo Theme Stack](https://github.com/CaiJimmy/hugo-theme-stack) 构建，并通过 GitHub Actions 部署到 GitHub Pages。

本地预览：

```bash
hugo server -D
```

## 发布新博客

在项目根目录执行：

```bash
hugo new content/post/my-first-post/index.md
```

然后编辑生成的 `content/post/my-first-post/index.md`，将文章头部的 `draft = true` 改为 `draft = false`，例如：

```toml
+++
title = "我的第一篇文章"
date = 2026-09-21T12:00:00+08:00
draft = false
tags = ["随笔"]
categories = ["生活"]
+++
```

本地确认无误后提交并推送：

```bash
git add content/post/my-first-post/index.md
git commit -m "Add my first post"
git push
```

推送到 `main` 后，GitHub Actions 会自动构建并发布到 GitHub Pages。
