# 一次性配置指南

> 这些步骤在 GitHub 网页侧完成，**只做一次**。完成后 push 即自动部署。

## 0. 先决条件

- 已创建源仓库 `RUIFENGN/ruifeng-blog-source`（推荐 private）
- 已有产物仓库 `RUIFENGN/ruifeng.github.io`（GitHub Pages 已启用）
- 本地已 clone 源仓库且 `git submodule update --init` 拉好主题

## 1. 生成跨仓库部署密钥

在本地任意临时目录执行：
```bash
ssh-keygen -t ed25519 -C "blog-deploy" -f blog_deploy_key -N ""
# 生成 blog_deploy_key (私钥) 和 blog_deploy_key.pub (公钥)
```

## 2. 在**产物仓库**添加 Deploy Key（公钥）

打开 https://github.com/RUIFENGN/ruifeng.github.io/settings/keys

- Add deploy key
- Title: `blog-source-deploy`
- Key: 粘贴 `blog_deploy_key.pub` 全文
- **必须勾选** `Allow write access`
- Add key

## 3. 在**源仓库**添加 Secret（私钥）

打开 https://github.com/RUIFENGN/ruifeng-blog-source/settings/secrets/actions

- New repository secret
- Name: `ACTIONS_DEPLOY_KEY`
- Secret: 粘贴 `blog_deploy_key` 全文（含 `-----BEGIN OPENSSH PRIVATE KEY-----` 起止行）
- Add secret

> 添加完后，本地的 `blog_deploy_key*` 两个文件就可以删了。

## 4. 确认产物仓库的默认分支

打开 https://github.com/RUIFENGN/ruifeng.github.io/settings

确认默认分支是 `master`（与 `.github/workflows/deploy.yml` 里 `publish_branch: master` 一致）。
若产物仓库默认分支是 `main`，请把 workflow 中的 `publish_branch` 改成 `main`。

## 5. 确认 GitHub Pages 配置

打开 https://github.com/RUIFENGN/ruifeng.github.io/settings/pages

- Source: `Deploy from a branch`
- Branch: `master` / `(root)`
- Save

## 6. 触发首次部署

```bash
cd ruifeng-blog-source
git add . && git commit -m "init: rebuild blog source"
git push -u origin main   # 源仓库默认分支假设是 main
```

打开 https://github.com/RUIFENGN/ruifeng-blog-source/actions 观察 workflow 跑通。

## 7. 验证线上

- https://RUIFENGN.github.io/ —— 首页只剩 2 篇文章
- https://RUIFENGN.github.io/wrt/一名-entj-的一年计划/ —— 旧链接仍可访问
- https://RUIFENGN.github.io/index.xml —— RSS 正常

---

## 可能的坑

| 现象 | 解决 |
|---|---|
| Actions 报 `Permission denied (publickey)` | Deploy Key 没勾 write，或 Secret 内容有缺漏（特别是开头/结尾换行） |
| 产物仓库 push 成功但页面没变 | GitHub Pages 缓存，1-2 分钟后强制刷新（Cmd+Shift+R） |
| 主题样式跟原站有差 | 对照 [diary 主题 README](https://github.com/amazingrise/hugo-theme-diary) 调 `[params]` |
| 中文 URL 编码差异 | Hugo 默认会做 percent-encoding，与原站一致，无需特殊处理 |
