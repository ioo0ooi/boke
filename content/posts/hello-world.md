---
title: "欢迎使用本博客"
date: 2026-10-05T10:00:00+08:00
tags: ["公告", "Hugo"]
author: "mql"
---

这是本博客的第一篇文章，用于验证 Hugo 构建与 GitHub Pages 自动部署链路。

## 写作规范

- 新文章放在 `content/posts/` 目录，文件名即 URL slug（如 `my-post.md` -> `/posts/my-post/`）。
- frontmatter 必须包含四个字段：`title`、`date`、`tags`、`author`。
- 正文使用 Markdown。

## 发布流程

1. 在新分支上写文章；
2. 向 `main` 提交 Pull Request；
3. PR 合并到 `main` 后，GitHub Actions 自动构建并部署到 GitHub Pages。
