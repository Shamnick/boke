# 🌿 小小人间不慌张

> 个人博客 — 基于 Hugo + GitHub Pages

## 技术栈

- ⚡ **Hugo** — 极速静态站点生成
- 🚀 **GitHub Pages** — 免费托管 & 自动部署
- 📝 **Markdown** — 纯文本写作，专注内容

## 目录结构

```
boke/
├── archetypes/        # 文章模板
├── content/posts/     # 📝 文章目录（写在这！）
├── layouts/           # 主题模板
├── static/            # 静态资源
├── config.toml        # Hugo 配置
└── .github/workflows/ # CI/CD 自动部署
```

## 发布流程

1. 在 `content/posts/` 下新建 `.md` 文件
2. frontmatter 格式：
   ```yaml
   ---
   title: "文章标题"
   date: 2026-07-01
   tags: ["标签1", "标签2"]
   author: "Shamnick"
   ---
   ```
3. 提交 Pull Request
4. 合并到 `main` 分支 → GitHub Actions 自动构建部署 ✨

## 本地预览

```bash
hugo server -D
# 访问 http://localhost:1313/boke/
```
