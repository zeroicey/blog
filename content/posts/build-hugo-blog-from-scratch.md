+++
title = '从零搭建个人博客：Hugo + Cloudflare Pages 完整教程'
date = '2026-08-08T14:30:00+08:00'
draft = false
tags = ['Hugo', 'Cloudflare Pages', '博客', '教程', 'R2', 'Pagefind', 'giscus', 'SEO']
description = '以本博客为例，从安装 Hugo 到部署上线，再到站内搜索、评论区、搜索引擎收录，一份完整的免费博客搭建教程。'
+++

你现在看到的这个网站，就是一篇"教程本身"——它由 Hugo 生成、托管在 Cloudflare Pages 上、图片放在 Cloudflare R2、有站内搜索和 GitHub 评论区。整套方案 **零服务器成本**，只有域名费（几十块一年）。

这篇文章把从 0 到上线的完整过程记录下来，跟着做你也能拥有一个同样的博客。

# 技术栈总览

先看整条链路：

| 环节 | 选型 | 为什么 |
|---|---|---|
| 静态站点生成 | Hugo (extended) | 单二进制、秒级构建、原生多语言 |
| 主题 | hugo-coder | 简洁、内置多语言/深色模式/评论扩展点 |
| 代码托管 | GitHub | 免费、和 Cloudflare Pages 无缝集成 |
| 部署 | Cloudflare Pages | 免费、自动构建、全球 CDN |
| 图床 | Cloudflare R2 + 自定义域名 | 免费额度 10GB，零出口流量费 |
| 站内搜索 | Pagefind | 纯静态索引、无需后端、免费 |
| 评论区 | giscus | 基于 GitHub Discussions，零数据库 |
| 域名 | zeroicey.me（在 Cloudflare 管理） | 解析和 CDN 同一处，配置最省事 |

# 第一步：安装 Hugo

Hugo 有普通版和 extended 版，**请装 extended**（支持 SCSS 等特性，主流主题基本都要求）。

macOS：

```bash
brew install hugo
```

Linux / Windows 或其他系统，见[官方安装文档](https://gohugo.io/installation/)，或者直接下载 release 二进制。

验证：

```bash
hugo version
# hugo v0.161.1+extended ... VendorInfo=Homebrew
```

注意版本号里要带 **extended**。

# 第二步：创建站点并安装主题

```bash
hugo new site blog
cd blog
git init
```

然后安装主题。我用的是 [hugo-coder](https://github.com/luizdepra/hugo-coder)，用 **git submodule** 方式引入（方便后续 `git pull` 更新主题）：

```bash
git submodule add https://github.com/luizdepra/hugo-coder.git themes/hugo-coder
```

在 `hugo.toml` 里指定主题：

```toml
theme = 'hugo-coder'
```

# 第三步：配置站点

我的 `hugo.toml` 核心部分长这样（完整版见博客仓库）：

```toml
baseURL = 'https://zeroicey.me/'
theme = 'hugo-coder'
defaultContentLanguage = 'zh-cn'

[pagination]
  pagerSize = 10

[markup]
  [markup.goldmark.renderer]
    unsafe = true   # 允许正文里写原始 HTML

[params]
  avatarURL = 'images/avatar.webp'
  since = 2026
  colorScheme = 'auto'   # 跟随系统深色/浅色
  preconnect = ['https://s3.blog.zeroicey.me']  # 图床域名预连接，加速图片加载
```

## 多语言配置

博客是双语（中文 + English），Hugo 原生支持：

```toml
defaultContentLanguage = 'zh-cn'

[languages.zh-cn]
  label = '中文'
  locale = 'zh-CN'
  title = '雪宁韵'
  weight = 1

[languages.en]
  label = 'English'
  locale = 'en-US'
  title = 'zeroicey'
  weight = 2
```

## 导航菜单

```toml
[[languages.zh-cn.menu.main]]
  name = '博客'
  weight = 1
  url = 'posts/'

[[languages.zh-cn.menu.main]]
  name = '关于'
  weight = 2
  url = 'about/'
```

# 第四步：写第一篇文章

Hugo 文章是带 front matter 的 Markdown：

```markdown
+++
title = '文章标题'
date = '2026-08-08T14:30:00+08:00'
draft = false
tags = ['标签1', '标签2']
+++

正文...
```

双语站点要写一对文件：`content/posts/xxx.md` 是中文，`xxx.en.md` 是英文翻译，两个文件 front matter 的 `date` 相同。

本地预览（带草稿）：

```bash
hugo server -D
# 打开 http://localhost:1313
```

构建验证：

```bash
hugo --gc --minify
```

# 第五步：图片图床（Cloudflare R2）

博客图片放在 R2 bucket，通过自定义域名公开访问，域名 `s3.blog.zeroicey.me`。R2 免费额度 10GB 存储，并且 **零出口流量费**，对博客这种场景几乎免费。

创建 bucket：

```bash
npx wrangler r2 bucket create blog
```

在 Cloudflare 控制台给 bucket 绑定自定义域名（R2 设置 → Custom Domains → 填 `s3.blog.zeroicey.me`，会自动创建 CNAME）。

上传图片：

```bash
npx wrangler r2 object put blog/20260808123835.webp --file=~/Pictures/screenshot.webp --remote
```

> **重要**：`--remote` 必须加！不加的话只写入本地模拟实例，线上访问是 404。

上传后立即验证：

```bash
curl -s -o /dev/null -w "%{http_code}" https://s3.blog.zeroicey.me/20260808123835.webp
# 200 才可用
```

文章里直接引用公开 URL：

```markdown
![截图](https://s3.blog.zeroicey.me/20260808123835.webp)
```

# 第六步：站内搜索（Pagefind）

Hugo 本身没有搜索功能，社区标准方案是 **Pagefind** —— 构建时给静态站生成搜索索引，浏览器端离线搜索，零后端。

构建后生成索引：

```bash
npx --yes pagefind --site public
```

然后在站点里加一个搜索按钮 + 弹窗 UI。我用 `layouts/_partials/search.html` 实现了：

- 导航栏加一个放大镜按钮，点击弹出搜索框
- 懒加载 `/pagefind/pagefind-ui.js`（不影响首屏速度）
- 支持中文和英文界面（用 Hugo 的 i18n 翻译）

注意 Cloudflare Pages 的构建命令里要**加上 pagefind 这一步**（见第八步）。

# 第七步：GitHub 评论区（giscus）

评论系统选了 **giscus**：评论存进你仓库的 GitHub Discussions，访客用 GitHub 账号登录即可评论，零数据库、零成本。

主题 `hugo-coder` 内置了 giscus 支持，需要：

1. 仓库开启 Discussions（Settings → Features → Discussions）
2. 安装 [giscus App](https://github.com/apps/giscus) 并授权给仓库
3. 在 `hugo.toml` 配置：

```toml
[params.giscus]
  repo = 'zeroicey/blog'
  repoID = 'R_kgDOSjjc-g'
  category = 'General'
  categoryID = 'DIC_kwDOSjjc-s4DC64_'
  mapping = 'pathname'
  theme = 'preferred_color_scheme'
```

`repoID` 和 `categoryID` 是 GitHub 的内部 ID，可以在 [giscus.app](https://giscus.app) 网站上配置后自动生成，或者用 GitHub API 查询：

```bash
gh api graphql -f query='query {
  repository(owner: "zeroicey", name: "blog") {
    id
    discussionCategories(first: 20) { nodes { id name } }
  }
}'
```

另外我给 giscus 加了一个小改进：覆盖主题的 partial，让评论界面语言**跟随文章语言**（中文文章显示中文评论 UI，英文文章显示英文）。

# 第八步：部署到 Cloudflare Pages

Cloudflare Pages 有 **Git 集成**：连上 GitHub 仓库后，每次 push 到 main 分支自动构建部署。

在 Cloudflare 控制台：Workers & Pages → Create → Pages → Connect to Git → 选仓库。

## 关键：构建配置

**构建命令**（这个很重要，Cloudflare 云端预装的 Hugo 版本太旧，不支持新的配置格式，必须手动下载新版本）：

```bash
curl -sL https://github.com/gohugoio/hugo/releases/download/v0.161.1/hugo_extended_0.161.1_linux-amd64.tar.gz | tar -xz && ./hugo --gc --minify && npx --yes pagefind --site public
```

**输出目录**：

```
public
```

配好后，以后只需要：

```bash
git add . && git commit -m "post: xxx" && git push
```

剩下的全自动：Cloudflare 拉代码 → 构建 → 全球 CDN 发布，一般 1-2 分钟生效。

# 第九步：让搜索引擎收录

部署完别忘了几件事（不然别人搜不到你）：

1. 确认 `https://你的域名/robots.txt` 允许爬虫（`User-agent: *` + `Allow: /`）
2. 确认 sitemap 可访问：`/sitemap.xml`
3. 去 [Google Search Console](https://search.google.com/search-console) 添加站点，用 **网域** 类型，验证方式选 DNS TXT 记录（域名在 Cloudflare 的话，复制 TXT 值去 Cloudflare 加一条记录即可）
4. 验证通过后提交 sitemap：`sitemap.xml`
5. 用 **URL 检查** 工具提交首页，请求索引

Google 首次收录需要几天到几周，之后新文章通过 sitemap 会被自动发现。

# 完整目录结构

最后看一下整个站点的结构：

```
blog/
├── hugo.toml              # 站点配置
├── content/
│   └── posts/
│       ├── xxx.md         # 中文文章
│       └── xxx.en.md      # 英文翻译
├── layouts/               # 覆盖主题的模板
│   ├── _markup/
│   │   └── render-image.html   # 图片懒加载
│   └── _partials/
│       ├── search.html    # 搜索弹窗
│       ├── header.html    # 导航栏（含搜索按钮）
│       └── posts/
│           └── giscus.html     # 评论（语言自适应）
├── static/
│   └── images/
│       └── avatar.webp    # 头像
└── themes/
    └── hugo-coder/        # git submodule
```

# 成本小结

| 项目 | 费用 |
|---|---|
| Hugo / 主题 | 免费（开源） |
| GitHub 仓库 | 免费 |
| Cloudflare Pages 托管 | 免费（每月 500 次构建额度） |
| Cloudflare R2 图床 | 免费（10GB 存储 + 零出口流量） |
| Pagefind / giscus | 免费 |
| 域名 | 唯一的花费，几十块/年 |

这就是整套方案的全部成本。技术栈全部开源、零厂商锁定（Hugo 站点随时可以迁走），值得一试。
