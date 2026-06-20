# 清辞 · 博客源工程

Hugo 0.116.1 + [hugo-theme-diary](https://github.com/amazingrise/hugo-theme-diary)。
本仓库为**源工程**；构建产物自动发布到 [`RUIFENGN/ruifeng.github.io`](https://github.com/RUIFENGN/ruifeng.github.io)。

## 目录结构

```
.
├── archetypes/           新文章模板
├── content/              文章内容
│   └── wrt/              「写作」分类
├── layouts/              站点级 layout 覆盖（可选）
├── static/               静态资源
├── themes/diary/         主题（git submodule）
├── hugo.toml             站点配置
└── .github/workflows/    自动构建发布
```

## 本地开发

```bash
# 1. 安装 Hugo（macOS）
brew install hugo

# 2. 拉主题（首次 clone 仓库后必做）
git submodule update --init --recursive
# 或首次添加：
# git submodule add https://github.com/amazingrise/hugo-theme-diary themes/diary

# 3. 本地预览
hugo server -D
# 浏览器访问 http://localhost:1313/
```

## 写新文章

```bash
hugo new wrt/文章标题/index.md
# 编辑 content/wrt/文章标题/index.md，把 draft: true 改成 false
git add . && git commit -m "post: 文章标题"
git push
```

push 后 GitHub Actions 会自动构建并把产物推送到 `RUIFENGN/ruifeng.github.io`，
GitHub Pages 1-2 分钟内生效。

## 部署机制

源仓库 push → Actions：
1. `peaceiris/actions-hugo@v3` 安装 Hugo 0.116.1（extended）
2. `hugo --minify --gc` 构建 `public/`
3. `peaceiris/actions-gh-pages@v4` 用 Deploy Key 跨仓库推送到产物仓库 `master` 分支

需要在 GitHub 网页侧完成的一次性配置见 [`SETUP.md`](./SETUP.md)。

## 维护要点

- **图片**：放在文章自身目录（page bundle），与 `index.md` 同级
- **草稿**：front matter `draft: true`，`hugo server -D` 可预览
- **主题升级**：`git submodule update --remote themes/diary` → 本地验证 → push
- **回退**：产物仓库的 tag `pre-rebuild-snapshot` 是重构前的最后快照
