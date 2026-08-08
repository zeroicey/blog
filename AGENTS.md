# AGENTS.md

Hugo 多语言博客(中文 + English),部署在 Cloudflare Pages。用户(雪宁韵 / zeroicey)使用 AI 助手写作、传图、发布文章。

## 项目概览

- **框架**: Hugo v0.161+ (extended),主题 `hugo-coder`(git submodule)
- **内容**: `content/posts/*.md`(中文)+ `*.en.md`(英文翻译,双语成对维护)
- **配置**: `hugo.toml`,默认语言 zh-cn,baseURL `https://zeroicey.me/`
- **本地构建**: `hugo --gc --minify`,产物在 `public/`(不入库)
- **发布流程**: git commit + push 到 `main` → Cloudflare Pages 自动构建部署

## 写新文章的模板

```markdown
+++
title = '标题'
date = 'YYYY-MM-DDTHH:MM:SS+08:00'
draft = false
tags = ['标签1', '标签2']
+++

正文...
```

- 中英文成对:写 `content/posts/xxx.md` 后必须同步写 `content/posts/xxx.en.md`
- 日期用当前时间(UTC+8),`draft = true` 不会发布
- 写完跑 `hugo --gc --minify` 验证构建

## 图片上传(R2 图床)

图片放 Cloudflare R2 bucket `blog`,公开域名 `https://s3.blog.zeroicey.me/`。

```bash
npx wrangler r2 object put blog/<key>.png --file=<本地路径> --remote
```

**关键注意**:
- **必须加 `--remote`**,否则只写入本地模拟实例,线上 404
- key 用时间戳格式,如 `20260808123835.png`(保持和现有图片一致)
- 上传后立即用 curl 验证:200 才可用;404 说明缓存问题,换新 key 重传(无法 purge 缓存,oauth scope 缺 zone 权限)
- 支持 PNG 等常见格式,content-type 自动识别

## 部署

- 生产:push 到 `main` → Cloudflare Pages 自动构建
- 构建命令: `curl -sL https://github.com/gohugoio/hugo/releases/download/v0.161.1/hugo_extended_0.161.1_linux-amd64.tar.gz | tar -xz && ./hugo --gc --minify`(云端 Hugo 版本太旧,必须用此命令)
- 输出目录:`public`
- 验证:`curl -s --noproxy '*' -o /dev/null -w "%{http_code}" https://zeroicey.me/posts/<slug>/`
- 文章可能出现短暂缓存延迟,等待后重试

## Cloudflare 凭据

- wrangler OAuth token 位于 `~/Library/Preferences/.wrangler/config/default.toml`
- token 会过期,wrangler 命令会自动刷新
- 手动 API 调用(如 purge cache)可能因 scope 不足失败,优先用 wrangler CLI

## 常用命令

```bash
hugo --gc --minify                              # 本地构建
hugo server -D                                  # 本地预览(含草稿)
npx wrangler r2 object list blog                # 列出图床文件
npx wrangler pages deployment list --project-name blog  # 查看部署状态
```
