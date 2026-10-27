# AI_Post_Daily

Hugo 静态博客，部署在 GitHub Pages，推送即自动发布。

- 站点地址：https://crzkn.github.io/AI_Post_Daily/
- 构建：Hugo（extended，版本见 `.github/workflows/deploy.yml`）
- 主题：PaperMod（git submodule，位于 `themes/PaperMod`）
- 部署：`main` 分支 push 触发 `.github/workflows/deploy.yml` → GitHub Pages（Actions 构建源）

## 写一篇新文章

```bash
hugo new content/posts/my-post.md   # 自动生成四字段 frontmatter
```

frontmatter 固定四字段：

```yaml
---
title: "标题"
date: 2026-10-27T21:00:00+08:00
tags: ["tag1", "tag2"]
author: "CrzKn"
---
```

## 发布流程

```bash
git checkout -b post/my-post
git add content/posts/my-post.md
git commit -m "post: my-post"
git push origin post/my-post
gh pr create --fill        # 或用网页发 PR
```

PR 合并到 `main` 后 Actions 自动构建并部署，无需手动操作。也可以本地 `hugo server -D` 预览后再推。

## 克隆（注意子模块）

```bash
git clone --recurse-submodules https://github.com/CrzKn/AI_Post_Daily.git
```
