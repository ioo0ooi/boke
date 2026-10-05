# boke

GitHub Pages + Hugo 静态博客。站点地址：<https://ioo0ooi.github.io/boke/>

## 目录结构

```
content/posts/    # 文章（Markdown）
archetypes/       # hugo new 文章模板
themes/ananke     # 主题（git submodule）
.github/workflows/hugo.yml  # Pages 自动部署
```

## 文章 frontmatter

每篇文章必须包含四个字段：

```yaml
---
title: "文章标题"
date: 2026-10-05T10:00:00+08:00
tags: ["标签1", "标签2"]
author: "mql"
---
```

## 发布流程

1. 新文章放入 `content/posts/`，在新分支提交；
2. 向 `main` 发起 Pull Request；
3. PR 合并到 `main` 后，GitHub Actions 自动构建部署（PR 本身不触发部署）。

## 本地预览

```bash
hugo server -D
```
