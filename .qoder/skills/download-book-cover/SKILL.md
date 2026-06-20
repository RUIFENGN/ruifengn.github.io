# Download Book Cover & Deploy

从微信读书（WeRead）官方源搜索并下载书籍封面图片，更新模板，自动 git push 触发 GitHub Actions 部署到线上。

## 触发条件
当用户需要为书架/影视/桌游栏目添加新条目，或需要替换现有封面图片时使用。

## 项目架构

```
ruifeng-blog-source/          ← Hugo 源码仓库（编辑+构建+push）
├── layouts/index.html        ← 首页模板（书架条目在这里）
├── static/covers/            ← 封面图存放目录
├── .github/workflows/deploy.yml  ← CI/CD
└── ...

Git push → GitHub Actions 自动 hugo build → 部署到 ruifeng.github.io
```

**关键路径：**
- 源码仓库：`/Users/nieruifeng/Downloads/Projects/ruifeng-blog-source/`
- Git remote：`https://github.com/RUIFENGN/ruifeng-blog-source.git` (branch: main)
- 部署目标：`https://github.com/RUIFENGN/ruifeng.github.io.git` (branch: master)

## 数据源

**首选源：微信读书 (WeRead)**
- 搜索页：`https://weread.qq.com/web/search/books?keyword={书名}`
- 图片CDN：`https://cdn.weread.qq.com/weread/cover/{path}.jpg`
- 特点：访问极快（<0.3s），返回官方出版封面，250×360 JPEG，电子书风格全出血填满
- 可靠性：已验证稳定可用，无需认证

**备选源：MyAnimeList CDN**（仅限动漫/影视）
- 图片CDN：`https://cdn.myanimelist.net/images/anime/{id}/{filename}`
- 特点：约0.7s，适合动漫作品封面

**备选源：BoardGameGeek CloudFront**（仅限桌游）
- 图片CDN：`https://cf.geekdo-images.com/{path}`
- 特点：约1.2s，需设置 `Referer: https://boardgamegeek.com/` 或下载到本地

## 执行步骤

### Step 1: 搜索书籍

```bash
curl -sL --max-time 10 --connect-timeout 6 \
  -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36" \
  "https://weread.qq.com/web/search/books?keyword={URL编码的书名}" \
  -o /tmp/weread-search.html
```

### Step 2: 从HTML中提取完整封面URL

```bash
grep -oE 'https://cdn\.weread\.qq\.com/weread/cover/[^"'\'']*\.(jpg|png)' /tmp/weread-search.html | head -5
```

输出示例：
```
https://cdn.weread.qq.com/weread/cover/95/YueWen_843464/t6_YueWen_843464.jpg
https://cdn.weread.qq.com/weread/cover/49/cpplatform_nghc3iwwyna7eyep6g7fcq/t6_cpplatform_nghc3iwwyna7eyep6g7fcq1739959995.jpg
```

### Step 3: 选择正确的结果

- 搜索结果按相关性排序，**第一个URL**通常是最匹配的书籍封面
- 如果第一个下载后文件大小 < 10KB，尝试第二个结果
- 可以下载后用 `file` 命令验证是否为有效 JPEG

### Step 4: 下载封面图片

```bash
curl -sL --max-time 10 --connect-timeout 6 \
  "{Step 2提取到的完整URL}" \
  -o {目标路径}
```

### Step 5: 验证图片

```bash
file {目标路径}  # 应显示 "JPEG image data"
ls -la {目标路径}  # 文件大小应 > 10KB（通常 30-100KB）
```

### Step 6: 放置到正确目录

封面图直接放到源码仓库：
```
/Users/nieruifeng/Downloads/Projects/ruifeng-blog-source/static/covers/{filename}.jpg
```

### Step 7: 编辑模板添加条目

编辑 `/Users/nieruifeng/Downloads/Projects/ruifeng-blog-source/layouts/index.html`，在对应 section 的 slice 中追加：

```go
(dict "cover" "/covers/{filename}.jpg" "title" "书名" "url" "https://链接")
```

### Step 8: Git 提交并推送（自动部署）

```bash
cd /Users/nieruifeng/Downloads/Projects/ruifeng-blog-source
git add static/covers/{filename}.jpg layouts/index.html
git commit -m "feat: 书架添加《书名》"
git push origin main
```

推送后 GitHub Actions 会自动：
1. checkout 源码
2. `hugo --minify --gc` 构建
3. 部署 public/ 到 ruifeng.github.io (master)

线上约 1-2 分钟后可见。

## 完整示例

下载《掌控习惯》封面：

```bash
# 1. 搜索
curl -sL --max-time 10 \
  -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36" \
  "https://weread.qq.com/web/search/books?keyword=%E6%8E%8C%E6%8E%A7%E4%B9%A0%E6%83%AF" \
  -o /tmp/wr-search.html

# 2. 提取封面URL（取第一个）
COVER_URL=$(grep -oE 'https://cdn\.weread\.qq\.com/weread/cover/[^"'\'']*\.(jpg|png)' /tmp/wr-search.html | head -1)
echo "$COVER_URL"
# 输出: https://cdn.weread.qq.com/weread/cover/25/yuewen_26934843/t6_yuewen_269348431702464198.jpg

# 3. 下载
curl -sL --max-time 10 "$COVER_URL" -o static/covers/atomic-habits.jpg

# 4. 验证
file static/covers/atomic-habits.jpg
# JPEG image data, baseline, precision 8, 250x361, components 3
```

## 批量下载示例

```bash
DST="/Users/nieruifeng/Downloads/Projects/ruifeng-blog-source/static/covers"
for pair in "掌控习惯:atomic-habits" "深度工作:deep-work" "原则:principles" "人类简史:sapiens" "刻意练习:peak"; do
  KEYWORD=$(echo $pair | cut -d: -f1)
  FILENAME=$(echo $pair | cut -d: -f2)
  ENCODED=$(python3 -c "import urllib.parse; print(urllib.parse.quote('$KEYWORD'))")
  URL=$(curl -sL --max-time 10 -H "User-Agent: Mozilla/5.0" \
    "https://weread.qq.com/web/search/books?keyword=$ENCODED" | \
    grep -oE 'https://cdn\.weread\.qq\.com/weread/cover/[^"'\'']*\.(jpg|png)' | head -1)
  curl -sL --max-time 10 "$URL" -o "$DST/$FILENAME.jpg"
  echo "$FILENAME: $(file -b "$DST/$FILENAME.jpg")"
done
```

## 一键完整流程（从书名到线上）

```bash
# === 输入 ===
BOOK_NAME="掌控习惯"
FILE_NAME="atomic-habits"
BOOK_URL="https://www.goodreads.com/book/show/40121378"
REPO="/Users/nieruifeng/Downloads/Projects/ruifeng-blog-source"

# === 1. 下载封面 ===
ENCODED=$(python3 -c "import urllib.parse; print(urllib.parse.quote('$BOOK_NAME'))")
COVER_URL=$(curl -sL --max-time 10 -H "User-Agent: Mozilla/5.0" \
  "https://weread.qq.com/web/search/books?keyword=$ENCODED" | \
  grep -oE 'https://cdn\.weread\.qq\.com/weread/cover/[^"'\'']*\.(jpg|png)' | head -1)
curl -sL --max-time 10 "$COVER_URL" -o "$REPO/static/covers/$FILE_NAME.jpg"
file "$REPO/static/covers/$FILE_NAME.jpg"  # 验证

# === 2. 编辑模板（手动或由AI完成）===
# 在 layouts/index.html 的 $books slice 中追加:
# (dict "cover" "/covers/$FILE_NAME.jpg" "title" "$BOOK_NAME" "url" "$BOOK_URL")

# === 3. 本地预览验证 ===
cd "$REPO" && hugo server -D &
# 打开 http://localhost:1313 确认显示正确

# === 4. 提交并部署 ===
cd "$REPO"
git add static/covers/$FILE_NAME.jpg layouts/index.html
git commit -m "feat: 书架添加《$BOOK_NAME》"
git push origin main
# GitHub Actions 自动构建部署，约1-2分钟后线上可见
```

## 注意事项

- 搜索结果中的第一个 `cdn.weread.qq.com` URL 即为最匹配书籍的封面
- URL 中的书名需要 URL 编码（`python3 -c "import urllib.parse; print(urllib.parse.quote('书名'))"`）
- 所有封面统一为 250×360 JPEG 格式，电子书全出血风格
- `object-fit: cover` 会自动填满容器边界
- 图片文件命名使用英文小写+连字符，如 `atomic-habits.jpg`
- **git push 后无需其他操作**，GitHub Actions 全自动构建+部署

## 不可用的源（当前网络环境）

以下在当前环境下超时或不可用，不要尝试：
- OpenLibrary (`covers.openlibrary.org`) — 连接超时
- Google Books (`books.google.com`) — 连接超时
- Google APIs (`googleapis.com`) — 连接超时
- jsDelivr (`cdn.jsdelivr.net`) — 连接超时
- 豆瓣图片 (`img*.doubanio.com`) — 返回 418
- 当当/京东 — URL 不可预测，需动态获取
