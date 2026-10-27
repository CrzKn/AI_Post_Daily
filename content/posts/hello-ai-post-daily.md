---
title: "Hello AI Post Daily"
date: 2026-10-27T21:00:00+08:00
tags: ["hugo", "github-pages", "公告"]
author: "CrzKn"
---

这是 AI_Post_Daily 的第一篇文章，用于验证「写 Markdown → 提 PR → 合并 main → 自动部署」的完整流程。

## 发布流程

1. 在 `content/posts/` 下新建 Markdown 文件，例如 `content/posts/my-post.md`。
2. frontmatter 只保留四个字段：`title`、`date`、`tags`、`author`。
3. 推送到分支并发起 Pull Request。
4. PR 合并进 `main` 后，GitHub Actions 自动构建 Hugo 并部署到 GitHub Pages。

## 本地预览（可选）

```bash
hugo server -D
```

打开 http://localhost:1313/AI_Post_Daily/ 即可预览。
