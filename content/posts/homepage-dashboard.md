+++
title = '自托管导航面板 Homepage：一个 YAML 生成漂亮的工作台'
date = '2026-09-06T22:50:00+08:00'
draft = false
tags = ['Homepage', '导航面板', '自托管', 'Docker', 'Tailscale', '教程']
description = '服务一多，浏览器书签栏就放不下了。用 Homepage（gethomepage）搭一面卡片式导航工作台：YAML 声明式配置、内建 Docker 集成一个面板看所有服务的在线状态与 CPU/内存，再配上本地自托管图标和极光背景，颜值与实用兼得。'
[cover]
  image = 'https://s3.blog.zeroicey.me/covers/homepage-dashboard.jpg'
+++

系列的前面几篇，把家里这套基础设施的「硬装」基本做完了：机器被 Tailscale 拽进同一个虚拟局域网、无头主机能被远程操作、每个服务都有免端口的 HTTPS 域名、监控面板盯着每台机器的资源水位。

可问题也来了——**服务越来越多，我反而记不住入口了。** 网盘、监控、笔记、音乐、书库、LLM 网关、自动化……这些服务散在一堆 `*.internal` 域名里，每次用都得翻书签、想端口。我需要的是一面「工作主页」：打开浏览器，所有服务一眼在列，点一下就走。

这篇就安利一个正好干这事的项目——[Homepage](https://github.com/gethomepage/homepage)。

## 为什么是 Homepage

做「导航首页」的自托管项目其实不少，我大致都摸过一遍：

| 工具 | 定位 | 我不选它的理由 |
| --- | --- | --- |
| Homer | 极简纯静态导航，YAML 配置 | 就是一面图标墙，没有服务和资源状态 |
| Dashy | 高可定制的仪表盘 | 功能很全，但配置项多得像给了把瑞士军刀 |
| Heimdall | 老牌 Web UI，图形化配置 | 不用写 YAML，但界面和扩展生态都显老 |
| Homarr | 主打 Plex / \*arr 影音套件集成 | 不是我的菜 |
| **Homepage** | **YAML 声明式 + Docker 集成 + 服务 widget** | ✅ 最终选择 |

我最终落在 Homepage 上，理由三条：

1. **声明式 YAML**：整个面板就是几个 `yaml` 文件，能进 Git 版本管理、能 diff、能脚本化改，跟我的运维习惯完全一致。
2. **原生 Docker 集成**：挂上 `docker.sock`，每个导航卡片能直接显示对应容器的**在线状态 + CPU / 内存**，点开还能展开网络与流量——这就不只是「入口」，还是一面活的状态墙。
3. **颜值在线**：主题、背景、玻璃拟态、几十个服务 widget（天气、日历、系统资源、第三方服务状态……），成品就是现代仪表盘该有的样子。

## 部署：一个 compose 就起来了

Homepage 是 Next.js 应用，官方镜像 `ghcr.io/gethomepage/homepage`，容器内端口 3000，配置目录挂载即可：

```yaml
# compose.yml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:3000"          # 只绑本地，交给 Caddy 反代
    volumes:
      - ./config:/app/config           # yaml 配置目录
      - /var/run/docker.sock:/var/run/docker.sock:ro   # docker 集成（只读）
```

```bash
docker compose up -d
```

> `docker.sock` 挂载是可选的：不挂，面板就只是静态导航；挂了（一定要 `ro` 只读），卡片才有容器状态和资源统计。

再套用系列里那套 Caddy + MagicDNS，给它安排一个免端口域名：

```caddy
https://home.homemesh.internal {
    bind 100.64.12.1
    tls /certs/server.crt /certs/server.key
    reverse_proxy 127.0.0.1:3000
}
```

浏览器打开 `https://home.homemesh.internal`，就能看到一面空面板了，接下来全靠 YAML 填内容。

## 配置：一切皆 YAML

配置目录下最核心的是三个文件：

| 文件 | 干什么 |
| --- | --- |
| `services.yaml` | 分组 + 服务导航项（href、图标、描述、Docker 集成） |
| `settings.yaml` | 站点标题、主题、背景、布局、语言 |
| `widgets.yaml` | 顶部的信息组件（搜索框、时钟、天气、系统资源…） |

一个 `services.yaml` 长这样：

```yaml
- 系统 · 运维:
    - 导航面板:
        icon: /icons/homepage.png
        href: https://home.homemesh.internal
        description: 服务工作台
        server: local
        container: homepage
    - 服务器监控:
        icon: /icons/komari.png
        href: https://komari.homemesh.internal
        description: CPU / 内存 / 磁盘
        server: local
        container: komari

- 数据 · 存储:
    - 网盘:
        icon: /icons/dufs.png
        href: https://dufs.homemesh.internal
        server: local
        container: dufs
```

看到 `server` + `container` 这两行没？这就是 Docker 集成——`server` 对应 `docker.yaml` 里配置的 Docker 实例名（本机 socket 即可），`container` 对应容器名。配好之后，卡片右边就会出现一枚红绿状态点，点开还能看 CPU / 内存 / 网络。把家里十几个 Web 服务按「AI / 数据 / 媒体 / 生活 / 运维」分组排进去，工作台就成了。

`settings.yaml` 管外观：

```yaml
title: Homelab
description: 个人工作台 · 服务导航
theme: dark
background: /images/bg.jpg     # 自定义背景
cardBlur: sm                   # 卡片玻璃拟态
language: zh-Hans
```

## 让它好看：背景 + 官方图标本地自托管

默认状态的 Homepage 有点素，两个小动作就能把颜值拉满。

**① 加背景 + 玻璃拟态。** 我用 ImageMagick 生成了一张 2560×1440 的暗色极光渐变底图（靛蓝 / 青 / 紫三团柔光 + 大半径模糊），挂进容器后设 `background: /images/bg.jpg`，再叠 `cardBlur: sm` 让卡片有毛玻璃质感、标题风格换成 `clean`。成品就是封面和配图的样子。

**② 图标用官方 logo，且本地自托管。** 这是最容易忽略、也最影响观感的一点：Homepage 默认让浏览器去**远端 CDN 现拉图标**（`si-*` / `mdi-*` / 裸名牌）。要是你的设备网络到不了那个 CDN（国内很常见），所有图标就退化成丑丑的首字母头像。解法是把图标下载下来自己伺服：

```bash
# 官方应用 logo：从 dashboard-icons 仓库拿，放进 ./icons/
curl -o ./icons/<名>.png \
  "https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/png/<名>.png"

# 自研/小众服务：用 Material Design 图标转成白色 PNG
convert -background none -density 1200 /tmp/<名>.svg -resize 128x128 \
  -channel RGB -negate +channel png32:./icons/<名>.png
```

compose 里补两条只读挂载：

```yaml
    volumes:
      - ./icons:/app/public/icons:ro
      - ./images:/app/public/images:ro
```

然后 `services.yaml` 的 `icon` 全部写成 `/icons/<名>.png` 指向本地文件——图标从此稳定显示，再也不受 CDN 脸色影响。

## 两个部署时踩的坑

**① 页面 400「Host validation failed」。** 新版 Homepage 会校验 HTTP Host 头，Host 不在白名单就直接 400，裸 IP 或新域名访问时最容易撞上。解法是在 compose 里加环境变量，把允许的 `host:port` 都列进去：

```yaml
    environment:
      - HOMEPAGE_ALLOWED_HOSTS=127.0.0.1:3000,100.64.12.1:3000,home.homemesh.internal
```

> 后面每加一个入口域名 / 端口，记得同步补进白名单再重启，否则又是一个 400。

**② healthcheck 别用 busybox `wget`。** 容器健康检查若写成 `wget -q -O /dev/null http://127.0.0.1:3000/`，会假健康——镜像里的 busybox `wget` 对 4xx / 5xx 也返回退出码 0，页面明明 400 它照样报 healthy。换成 Node 严格校验 `r.ok` 才是真的：

```yaml
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:3000/').then(r=>{if(!r.ok)process.exit(1)}).catch(()=>process.exit(1))"]
```

## 安全上的一点交代

Homepage 本身不鉴权、不登录，它只是「入口页 + 状态墙」，所以两点务必做：

1. **只绑内网 + Caddy 反代**：跟系列其他服务一样，容器端口只绑 `127.0.0.1`，外面一律走 `https://home.homemesh.internal`（HTTPS + 内网可达），不暴露公网。
2. **`docker.sock` 只读挂载**：读容器状态够用就行，别给面板写 Docker API 的能力。

## 收尾

把 Homepage 摆到浏览器首页，家里这套基础设施终于有了个「面子工程」：打开就是一张卡片墙，服务分类清晰、状态红绿一眼可辨，点谁进谁。配合前几篇，整条链路是：

**Tailscale 组网连得上 → Caddy 域名用得到 → Sunshine 远程桌面控得住 → Komari 面板看得清 → Homepage 工作台找得着。**

如果服务还不多，就从 Homepage 开始搭；等它长成几十个入口，你会庆幸当初选的是个 YAML 就能维护的导航面板。

相关链接：

- [gethomepage/homepage](https://github.com/gethomepage/homepage)（主仓库）
- [官方文档](https://gethomepage.dev/)（配置项权威参考）
- [homarr-labs/dashboard-icons](https://github.com/homarr-labs/dashboard-icons)（自托管应用图标库）