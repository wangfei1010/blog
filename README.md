# blog

Hugo + GitHub Pages 静态博客。

- 站点地址：https://wangfei1010.github.io/blog/
- 主题：[PaperMod](https://github.com/adityatelange/hugo-PaperMod)（内嵌在 `themes/PaperMod/`，不是 submodule）

## 发布流程

1. 新文章放在 `content/posts/` 下：

   ```bash
   # 本机需先装 Hugo（本仓库的 CI 自带，不装也能发）
   hugo new content posts/my-post.md
   ```

2. frontmatter 固定四个字段：

   ```yaml
   ---
   title: "文章标题"
   date: 2026-09-27
   tags: ["标签一", "标签二"]
   author: "wangfei"
   ---
   ```

3. 开分支 → commit → push → 发 PR。PR 会自动跑 `.github/workflows/pr-build.yml` 做构建检查。
4. 合并到 `main` 后，`.github/workflows/deploy.yml` 自动构建并部署到 GitHub Pages。

## 本地预览

```bash
git clone --recursive git@github.com:wangfei1010/blog.git
cd blog
hugo server -D
```

访问 http://localhost:1313/blog/

注意：本机的 `github.com:443` 被网络层阻断，已配好 `~/.ssh/config` 走 `ssh.github.com:443`，并把 `https://github.com/` 自动改写为 SSH。

## 结构

```
content/posts/   文章
archetypes/      hugo new 的模板
themes/PaperMod/ 主题
assets/css/extended/custom.css  中文字体等自定义样式
hugo.toml        站点配置
```
