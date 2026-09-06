+++
title = '免 VPN 加速 GitHub：Cloudflare Workers 自建 gh-proxy 镜像代理'
date = '2026-09-06T21:00:00+08:00'
draft = false
tags = ['GitHub', 'Cloudflare', 'Workers', 'CDN', '代理', '自托管', '教程', '油猴脚本']
description = 'GitHub 在国内又慢又容易断，机场节点还不稳。用 Cloudflare Workers 部署开源 gh-proxy 单文件代理，绑自己的域名免 VPN 拉取 release、raw、archive 和 git clone，再让代理内核直连 + 油猴脚本在网页端自动加速。'

[cover]
  image = 'https://s3.blog.zeroicey.me/covers/github-acceleration-cloudflare-workers.jpg'
+++

在国内用 GitHub，有两件几乎每个人都逃不掉的事：**网页打不开、下载龟速**。`git clone` 卡在 `Receiving objects` 半小时不动，下载个 release 二进制跑到一半断了，`raw.githubusercontent.com` 更是常年招架不住——要么上 VPN，要么求助于机场节点。

可机场节点也不省心。我自己折腾过一轮又一轮：节点「分钟级活↔死摆动」，健康检查明明显示 alive、真流量一去就超时。拿它去拉 GitHub，体验就是「这次成功了，下次又不行，全看脸」。

其实这件事有更优雅的解法：**GitHub 的文件和源码都是静态资源，用一个架在 Cloudflare 边缘的反代就能加速**。开源项目 `gh-proxy` 把这件事做成了「单文件 + 无服务器」——扔进 Cloudflare Workers，绑上自己的域名，之后任何 GitHub 链接前面加一个前缀，就能免 VPN 直接拉：

```text
# 之前：直连，慢 / 断 / 被墙
git clone https://github.com/owner/repo.git
curl -L https://github.com/owner/repo/releases/download/v1.0/app.zip

# 之后：走自己的加速域名
git clone https://gh.example.com/https://github.com/owner/repo.git
curl -L https://gh.example.com/https://github.com/owner/repo/releases/download/v1.0/app.zip
```

支持的范围很全：**release 文件、archive 源码包、blob/raw 分支文件、gist，以及 `git clone`**。这篇文章完整记录我怎么把它搭起来，包括两个不折腾一次绝对不知道的坑（都在「让代理内核直连」那一节）。

> ⚠️ **替换说明**：文中域名 `gh.example.com` 是占位符，实际部署时换成你自己托管在 Cloudflare 上的域名（比如给主域加一个 `gh` 子域）。代理内核一节以 mihomo（Clash Meta）为例，路径 `/etc/mihomo/config.yaml` 是其默认配置位置，其他 Clash 衍生内核同理。

## 原理：为什么一个前缀就能加速

`gh-proxy` 本质是个「域名重写器」。你的请求打到 `gh.example.com` 之后，Worker 把 `gh.example.com/` 后面的路径剥出来，补回 `https://github.com` 重新拼成目标地址，再回源 GitHub 把内容拉回来。于是 GitHub 上所有静态资源——release 附件、raw 文件、仓库压缩包——全都变成了从一个前缀可达：

```
你的设备 ──► gh.example.com（Cloudflare 边缘，就近接入、几乎不丢包）
                │  Worker 拼回 https://github.com/...
                ▼
           github.com（美国源站，Cloudflare 出口不受墙）
```

**真正起加速作用的是 Cloudflare 的边缘网络**：请求在你本地就近进入 Cloudflare 节点，再由 Cloudflare 到 GitHub 源站这一段走的是它的骨干网，绕开了国内到 GitHub 那条又挤又不稳的国际链路。所以对你而言，只是把「连 GitHub」换成了「连 Cloudflare」——后者在国内的可用性高一个量级。

## 为什么必须绑自定义域名

这里有个一踩一个准的坑：**Cloudflare 分配给你的默认域名 `*.workers.dev`，在国内是访问不了的**。所以「部署到 Workers 就完事」在墙内是行不通的——你必须有一块**托管在 Cloudflare 上的自己的域名**，把它绑成 Worker 的自定义域名。这正好也是整个方案成立的前提：域名和 Worker 在同一个 Cloudflare 账号下，绑定就是一条路由的事。

好在现在「自己的域名」门槛极低：任意一个域名把 NS 改到 Cloudflare（免费版即可），就能拿来用。绑定后访问 `gh.example.com`，请求在 CDN 层就被路由到你的 Worker 上了。

## Step 1：准备三样东西

- **Cloudflare 账号**（免费）+ 至少一个托管到 CF 的域名（本文用 `example.com`，给它加 `gh` 子域）。
- **wrangler**：Cloudflare 官方 CLI，`npm install -g wrangler`（或 `pnpm/npx`）。
- **一枚 API Token**：部署 Worker + 绑定自定义域名要用程序化方式。在 Cloudflare 后台 `My Profile → API Tokens → Create Token`，自定义权限：

| 作用域 | 资源 | 权限 |
| --- | --- | --- |
| Account | — | Workers Scripts · Edit |
| Account | — | Workers Routes · Edit |
| Zone | example.com | DNS · Edit |

> 也可以直接选现成的 **「Edit Cloudflare Workers」** 模板，再额外给 example.com 加一条 `DNS · Edit`。Token 只显示一次，存好。

## Step 2：拿到那个唯一的文件

`gh-proxy` 的 Workers 版**就一个 `index.js`**（一百多行），用不着克隆整个仓库——不过克隆看看源码也无妨：

```bash
# 二选一：只下这一个文件（更快）
curl -O https://raw.githubusercontent.com/hunshcn/gh-proxy/master/index.js

# 或克隆仓库
git clone https://github.com/hunshcn/gh-proxy
```

根路径 + 自定义域名的场景下，这个文件**一行都不用改**：`PREFIX='/'` 默认就对（除非你想把代理挂到某个子路径下，才需要改 `PREFIX` 并同步路由）。

## Step 3：写配置 + 部署

在 `index.js` 同目录建 `wrangler.toml`：

```toml
name = "gh-proxy"
main = "index.js"
compatibility_date = "2025-06-01"

# 自定义域名：要求 example.com 与 Worker 同账号
# wrangler 会自动帮你建 DNS 记录 + 绑定
routes = [
  { pattern = "gh.example.com", custom_domain = true }
]
```

然后部署：

```bash
export CLOUDFLARE_API_TOKEN="<你的 token>"
wrangler deploy
```

输出里出现 `gh.example.com (custom domain)` 就表示代码和域名都挂好了。这一条命令做了三件事：上传脚本、建 Worker 触发器、绑定自定义域名（连同 DNS 记录一起）。**全程不需要登 Cloudflare 网页点来点去。**

## Step 4：验收

```bash
# raw 文件
curl -s -o /dev/null -w "%{http_code}\n" \
  https://gh.example.com/https://raw.githubusercontent.com/hunshcn/gh-proxy/master/index.js
# → 200

# release 附件
curl -LO https://gh.example.com/https://github.com/owner/repo/releases/download/v1.0/app.tar.gz

# clone
git clone https://gh.example.com/https://github.com/hunshcn/gh-proxy
```

三个都通了，代理本体就绪。到这一步，网络通畅的设备已经能免 VPN 用上了。

## Step 5：让代理内核「直连」这个域名（真正的坑在这）

如果你的日常环境挂了 Clash / mihomo 之类的代理内核，浏览器和终端的流量都会先经过它。这时访问 `gh.example.com`，默认会被内核按「海外站点」丢到某个机场节点——**绕了一圈又回到那不稳定的机场上去了，你自己搭的加速等于白费**。

正确做法：在内核里把 `gh.example.com` 声明为 **DIRECT（直连）**，让它不经机场、直接连 Cloudflare 边缘。这一步看起来只是加一条规则，但我在这里栽了三个跟头，逐个说：

**坑 1：`hosts` 只能放顶层，别塞进 `dns:` 段。** mihomo 的静态解析配置是一段顶层 `hosts:`（和 `rules:`、`dns:` 平级）。如果你像我一样顺手把它写进 `dns:` 段下面，它会被**静默忽略**，没有任何报错。

**坑 2：直连也会「解析失败」。** 加了 `DOMAIN,gh.example.com,DIRECT` 之后，日志里依然出现：

```text
dial DIRECT (match Domain/gh.example.com) ...
error: dns resolve failed: context deadline exceeded
```

原因叠了两层：机器没有可用的 IPv6 路由，而内核 `ipv6: true` 会把 AAAA 记录也拿来拨号；再加上这是个「冷域名」，内核按 `fallback-filter`（非大陆 IP 必须走 `1.1.1.1/8.8.8.8` 这类 DoH）去解析——这些 DoH 直连在大陆又恰好被墙。于是解析就卡死超时了。（那些老域名如 `cloudflare.com` 碰巧没事，只是因为在缓存里是热的。）

**坑 3：改了 DNS 相关配置必须 `restart`，不是 `reload`。** HUP 热重载不会重建 DNS resolver，你改了 `hosts` 也会表现得「没生效」。

**最终正确配置**（把域名钉死到一个可达的 IPv4 边缘 IP，一步绕开上面所有问题）：

```yaml
# 顶层，不是 dns 段内
hosts:
  "gh.example.com": "172.67.1.2"   # 填你域名实际解析出的 A 记录（IPv4）

rules:
  - 'DOMAIN,gh.example.com,DIRECT'
```

```bash
# 必须完整重启（HUP 不重建 DNS）
sudo systemctl restart mihomo && sleep 20

# 验证：200 + 日志里出现 using DIRECT
curl -x http://127.0.0.1:7890 -o /dev/null -w "%{http_code}\n" \
  https://gh.example.com/
sudo journalctl -u mihomo --since "1 min ago" | grep gh.example.com
```

日志里看到 `match Domain(gh.example.com) using DIRECT`，就大功告成：走代理内核的流量现在也直连 Cloudflare 边缘、不经机场了。

## Step 6：网页端自动加速（油猴脚本）

命令行搞定了，浏览器里呢？总不能在每个 GitHub 页面手动把链接前缀补了吧。答案是脚本管理器 + 一个重写链接的用户脚本：

1. **装脚本管理器**：Chrome/Edge 用 [Tampermonkey（篡改猴）](https://www.tampermonkey.net/)，Firefox 用 Tampermonkey 或 Violentmonkey。
2. **装加速脚本**（二选一）：
   - [**GitHub加速下载**](https://greasyfork.org/zh-CN/scripts/504224)：专为 gh-proxy 设计，装完把加速地址设成你的 `https://gh.example.com/` 即可。
   - [**Github 增强 - 高速下载**](https://greasyfork.org/zh-CN/scripts/412245)：功能更全的老牌脚本，点扩展图标 → 自定义加速源，`Raw` / `Git Clone` / `Release(Code ZIP)` 三处都填 `https://gh.example.com/`。

装好后，网页上的 raw 文件、release 下载、clone 链接会自动改走你的加速域名，和命令行体验一致。

## 稳定性、额度与边界

- **额度**：Cloudflare Workers 免费版每天 10 万次请求、每分钟 1000 次。一个人自用（哪怕频繁 clone、下 release）绰绰有余；真不够了 $5/月的付费档是 1000 万次/月。
- **稳定性**：方案本身极稳——Worker 是无状态的，不存在「跑了几天崩了」的问题。真正的变量只有 Cloudflare 免费边缘偶尔在国内慢/抖，但那也比直连 GitHub、比不稳定的机场节点稳定得多。
- **边界（重要）**：作者提供的公共演示站早已不堪重负，**强烈建议自建、且只给自己用**。别把加速域名贴到公开场合——免费额度经不起被滥用，公开了就等着被人薅到限流。
- **安全**：`gh-proxy` 只代理 GitHub 相关域名（release/archive/raw/blob/gist/info/git-*），不是通用代理，无法被拿来访问任意网站，滥用风险天然很低。

## 收尾

回头看，这件事的价值不在于「又一个代理」，而在于**把一个高频的痛点换成了一个零维护的基础设施**：一次部署，永久生效，CLI 和浏览器两路都免 VPN。配合前面几篇搭好的 Tailscale 组网，家里的机器、云端服务器、日常开发都收在了同一套自托管体系里——这也是一直以来我想达成的样子：**网络入口这条路，能自己掌握的地方，都自己掌握。**

如果你也受够了 GitHub 的龟速，抽十分钟照这篇走一遍，大概率会后悔「怎么没早点搞」。