---
title: "用 PR 流程发第二篇文章"
date: 2026-09-27
tags: ["Hugo", "工作流"]
author: "wangfei"
---

这篇是通过完整 PR 流程发出来的：开分支 → 提 PR → PR 自动构建检查 → 合并到 `main` → 自动部署。

## 两个阶段

- **PR 阶段**：`.github/workflows/pr-build.yml` 跑一次构建，不部署
- **合并后**：`.github/workflows/deploy.yml` 构建并部署到 GitHub Pages

如果 PR 的构建检查失败，说明 frontmatter 或 Markdown 有问题，合并前就能发现。
