---
title: "第一篇文章：博客是怎么搭起来的"
date: 2026-09-27
tags: ["Hugo", "GitHub Pages"]
author: "wangfei"
---

这是第一篇测试文章，用来验证发布流程是否走得通。

## 发布流程

1. 新文章放在 `content/posts/` 目录下
2. 提交 PR（PR 会自动跑一次构建检查）
3. 合并到 `main` 分支后自动部署到 GitHub Pages

文章的 frontmatter 固定包含四个字段：`title` / `date` / `tags` / `author`。

```text
content/posts/hello-world.md
```
