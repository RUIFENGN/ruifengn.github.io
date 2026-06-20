# 个人博客（清辞）源工程重构与维护方案

## Context（背景）

当前 `RUIFENGN.github.io` 仓库（即本工作区）只剩 Hugo 构建后的**静态产物**（HTML/CSS/JS/RSS/sitemap），承担源代码角色的 Hugo 工程（`config`、`content/*.md`、`themes/`、`layouts/`、`archetypes/`）已经全部丢失，导致博客无法新增/修改文章，也无法升级主题或调整配置。

从产物中可还原的关键事实：
- Hugo 版本：**0.116.1**（见 `index.html` `<meta name="generator">`）
- 主题：[hugo-theme-diary](https://github.com/amazingrise/hugo-theme-diary)（开源，可直接拉回）
- 站点：标题「清辞」、副标题「每个人的心中都有一团火 / 但路过的人只能看到烟」、baseURL `https://RUIFENGN.github.io/`
- 顶栏菜单：`/images`、`/wrt`、`/tech`、`/sports`、`/tags`、`/index.xml`
- 实质有效文章 **仅 2 篇**：
  - [`wrt/一名-entj-的一年计划/`](file:///Users/nieruifeng/Downloads/Projects/ruifeng.github.io/wrt/一名-entj-的一年计划/index.html)
  - [`wrt/对重复一件事的思考/`](file:///Users/nieruifeng/Downloads/Projects/ruifeng.github.io/wrt/对重复一件事的思考/index.html)
- 其余 `wrt/first_post`、`tech/first_blog`、`tags/first_tag` 均为 DRAFT 空壳，按需求弃用
- 正文可从 [`wrt/index.xml`](file:///Users/nieruifeng/Downloads/Projects/ruifeng.github.io/wrt/index.xml) 与对应 `index.html` 反推为 Markdown，工作量极小

## 目标

1. 重建可持续维护的 Hugo 源工程
2. 保留现有 URL 与视觉（继续用 diary 主题）
3. 双仓库分工 + GitHub Actions 自动构建发布
4. 仅保留 2 篇真实文章，从干净状态重启

## 仓库拓扑

```
ruifeng-blog-source (新建，源工程)         RUIFENGN.github.io (现有，纯产物)
├── content/                                ├── index.html
├── themes/diary/ (submodule)               ├── wrt/...
├── hugo.toml                  ─Actions─►   ├── tags/...
├── .github/workflows/deploy.yml            ├── posts/...
└── ...                                     └── (由 Action 全量覆盖)
```

- **源仓库** 可设为 private，便于存草稿
- **产物仓库**（`RUIFENGN.github.io`）保持 public，由 GitHub Pages 自动服务

## 实施任务

### Task 1：在工作区**外**初始化 Hugo 源工程

> 不要在本工作区（产物仓库）内搭建源工程，二者必须物理分离。
> 建议路径：`/Users/nieruifeng/Downloads/Projects/ruifeng-blog-source/`

```bash
brew install hugo                       # 装最新版即可，下面 workflow 会锁版本
hugo new site ruifeng-blog-source
cd ruifeng-blog-source
git init
git submodule add https://github.com/amazingrise/hugo-theme-diary themes/diary
```

### Task 2：还原 `hugo.toml`（核心配置反推）

```toml
baseURL = "https://RUIFENGN.github.io/"
languageCode = "en-us"
title = "清辞"
theme = "diary"

[params]
  subtitle = "每个人的心中都有一团火\n但路过的人只能看到烟"
  copyrightStartYear = 2023
  customCSS = []
  customJS = []

[[menu.main]]
  name = "摄影"
  url = "/images"
  weight = 10
[[menu.main]]
  name = "写作"
  url = "/wrt"
  weight = 20
[[menu.main]]
  name = "技术"
  url = "/tech"
  weight = 30
[[menu.main]]
  name = "运动"
  url = "/sports"
  weight = 40
[[menu.main]]
  name = "标签"
  url = "/tags"
  weight = 50
[[menu.main]]
  name = "RSS Feed"
  url = "/index.xml"
  weight = 60

[permalinks]
  wrt = "/wrt/:slug/"
  tech = "/tech/:slug/"

[markup.goldmark.renderer]
  unsafe = true   # 主题 README 通常需要
```

> 实际字段名以 diary 主题 README 为准；先按此初版跑通，再对照线上视觉微调。

### Task 3：迁移 2 篇有效文章（Page Bundle 形式）

```
content/
└── wrt/
    ├── 一名-entj-的一年计划/
    │   └── index.md
    └── 对重复一件事的思考/
        └── index.md
```

每篇 front matter 模板：
```yaml
---
title: "一名 ENTJ 的一年计划"
date: 2023-08-08
description: "计划是一次大脑的狂欢！一次对愿景的挑战！"
tags: ["Writing"]
draft: false
---
```

正文从产物中两份来源择优转回 Markdown：
- 优先 [`wrt/xxx/index.html`](file:///Users/nieruifeng/Downloads/Projects/ruifeng.github.io/wrt/) 中 `<article>` 主体（保留标题层级）
- 辅助 `wrt/index.xml` 的 `<description>` 文本（已是脱标签纯文本，便于校对）

### Task 4：配置跨仓库自动部署

**4.1** 在产物仓库 `RUIFENGN.github.io` 加 Deploy Key（**Allow write access** 必须勾选）：
```bash
ssh-keygen -t ed25519 -C "blog-deploy" -f blog_deploy_key
# 公钥 blog_deploy_key.pub → 产物仓库 Settings → Deploy keys → Add
# 私钥 blog_deploy_key     → 源仓库 Settings → Secrets → ACTIONS_DEPLOY_KEY
```

**4.2** 源仓库 `.github/workflows/deploy.yml`：
```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.116.1'
          extended: true

      - name: Build
        run: hugo --minify --gc

      - name: Deploy to Pages repo
        uses: peaceiris/actions-gh-pages@v4
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }}
          external_repository: RUIFENGN/RUIFENGN.github.io
          publish_branch: main
          publish_dir: ./public
          force_orphan: true
          user_name: 'github-actions[bot]'
          user_email: 'github-actions[bot]@users.noreply.github.com'
          commit_message: 'deploy: ${{ github.sha }}'
```

> `force_orphan: true` 让产物仓库每次只保留一条提交，避免历史无限膨胀。

### Task 5：本地验证 → 推送源仓库 → 验证流水线

```bash
hugo server -D                # 本地 http://localhost:1313 比对线上
git add . && git commit -m "init"
git remote add origin git@github.com:RUIFENGN/ruifeng-blog-source.git
git push -u origin main
# 观察 Actions 跑通，访问 https://RUIFENGN.github.io 验证
```

### Task 6：备份与归档当前产物仓库

在覆盖前，把 **本工作区当前快照** 打 tag 备份：
```bash
cd /Users/nieruifeng/Downloads/Projects/ruifeng.github.io
git tag pre-rebuild-snapshot
git push origin pre-rebuild-snapshot
```
万一新流水线产物有问题，可随时回退。

## 长期维护建议

| 维度 | 做法 |
|---|---|
| 写新文章 | `hugo new wrt/标题/index.md`，写完去 draft，push 即发布 |
| 图片管理 | Page Bundle 同目录存放，迁移/移动时图片不脱节 |
| 主题升级 | `git submodule update --remote themes/diary` 后本地 `hugo server` 验收再 push |
| Hugo 版本 | 在 workflow 中显式锁版本（已锁 0.116.1），升级走 PR |
| 备份策略 | 源仓库本身就是 Git 历史；额外建议 1 年 1 次本地 `git bundle` 异地存档 |
| 草稿/隐私 | front matter `draft: true` 或源仓库设为 private |

## 验证清单

- [ ] 本地 `hugo server` 首页文章列表只剩 2 篇，标题/日期/摘要与线上一致
- [ ] 文章详情页排版（TOC、正文、暗色模式切换）与线上一致
- [ ] `/index.xml`、`/wrt/index.xml`、`/sitemap.xml` 正常生成
- [ ] push 源仓库后，Actions 绿、产物仓库收到一条 `deploy: <sha>` 提交
- [ ] 浏览器无强缓存访问 `https://RUIFENGN.github.io/`，效果等价于现状
- [ ] 旧 URL `https://RUIFENGN.github.io/wrt/一名-entj-的一年计划/` 仍可访问

## 风险与对策

| 风险 | 对策 |
|---|---|
| diary 主题最新版与 0.116.1 不兼容 | submodule 指向兼容 commit；或同步升级 Hugo 版本到主题要求 |
| 重建后某些参数（subtitle 多行、permalink）与线上有微差 | 用 `pre-rebuild-snapshot` tag 做 diff 校对，逐项调整 |
| Deploy Key 推送失败 | 检查公钥是否勾选 write、私钥换行是否完整 |
| 中文目录 URL 编码差异 | Hugo 默认会做 percent-encoding，与现有产物一致，无需特殊处理 |

## 不在本次范围

- 摄影、运动等空栏目的内容补全（按需后续添加）
- 评论系统、统计、SEO 优化（保持与现状一致即可）
- 主题大改造或换皮（如有需要另起任务）
