+++
title = '轻量服务器监控：Komari + Tailscale 一眼看清多台设备的 CPU / 内存 / 磁盘'
date = '2026-09-06T21:00:00+08:00'
draft = false
tags = ['Komari', 'Tailscale', 'Headscale', '监控', '自托管', '服务器', '教程']
description = 'Uptime Kuma 只能看「通不通」，Cockpit 只能看单台 Linux。用 Komari 的 server + agent 架构，一个面板看清全家设备的 CPU / 内存 / 磁盘 / 网络占用和逐秒历史曲线，agent 一个二进制全平台装。'
[cover]
  image = 'https://s3.blog.zeroicey.me/covers/komari-server-resource-monitoring.jpg'
+++

前面几篇把「连得上」和「用得到」都做完了：第一篇把家里和云端的机器拉进同一个 Tailscale 虚拟局域网并配好 MagicDNS；第二篇让无头主机也能像坐在机箱前一样被远程操作；第三篇用 Caddy 给每个服务安排了免端口的 HTTPS 域名。

但「连得上、用得到」之后，还有一个更基础的问题一直没解决——**这些机器「过得怎么样」？** 主控机 CPU 有没有被某个容器悄悄吃满？Mac 的内存是不是又快见底？香橙派那块 TF 卡还剩多少空间？哪台设备的负载最近一小时在往上蹿？

浏览器里能连上服务，但机器本身的健康状况，我还是两眼一抹黑。

## 为什么之前一直没找到合适的

为了「看清每台机器用了多少资源」，我陆陆续续试过几个方案，都不太对味：

| 工具 | 能做什么 | 缺什么 |
| --- | --- | --- |
| Uptime Kuma | ping、端口探活、HTTP 状态码、宕机告警 | 它只管「通不通」，**完全没有 CPU / 内存 / 磁盘**这类资源数据 |
| Cockpit | 单台 Linux 的磁盘、内存、服务、温度 | 一次只看一台，且只支持 Linux |
| glances / btop | 单机实时，终端里很好看 | 没有统一面板，多台设备要一个个 SSH 上去看 |

我要的其实就三件事：**一个面板、覆盖全部设备、能看 CPU / 内存 / 磁盘 / 网络的历史曲线**。找了一圈，最后落在 [Komari](https://github.com/komari-monitor/komari) 上——一个轻量自托管的服务器监控，正好是「server + agent」两段式架构，跟我这套 Tailscale 组网天然契合。

## Komari 是什么

一句话：**主控机跑一个 server（单容器），每台被测设备装一个 agent（Go 单二进制），agent 逐秒把 CPU / 内存 / 磁盘 / 网络 / 负载 / 连接数上报给 server，server 出面板和图表。**

官方长这样（主面板，一眼看到所有机器的实时占用）：

![Komari 主面板](https://s3.blog.zeroicey.me/posts/komari-server-resource-monitoring/home-dashboard.jpg)

点进任意一台，能看到逐秒的历史曲线：

![Komari 历史图表](https://s3.blog.zeroicey.me/posts/komari-server-resource-monitoring/history-charts.jpg)

架构非常清爽：

```text
  每台设备（agent，一个二进制，常驻）
     │   WebSocket + HTTP 主动上报（连回 server）
     ▼
  ┌─────────────────────────────┐
  │  core · 100.64.12.1          │
  │  komari server（Docker 单容器）│
  │  ├─ 内嵌 Web UI（仪表盘/图表） │
  │  └─ SQLite 持久化（主库+指标库）│
  └──────────────┬──────────────┘
                 │  Caddy 反代（HTTPS 域名，只绑内网）
                 ▼
            你的浏览器
```

> ⚠️ **替换说明**：与前几篇一致，文中主机名（`core`）、内部域（`homemesh.internal`）、Tailscale IP（`100.64.12.x`）均为虚构示例，照着做时换成你自己的真实值。

## 部署：server 就三步

server 用 Docker Compose，官方默认端口 25774，数据放本地卷：

```yaml
# compose.yml
services:
  komari:
    image: ghcr.io/komari-monitor/komari:1.4.3
    container_name: komari
    restart: unless-stopped
    ports:
      - "127.0.0.1:25774:25774"   # 只绑本地，交给 Caddy 反代
    volumes:
      - ./data:/app/data
```

```bash
docker compose up -d
```

第一次启动会停在「初始化向导」，浏览器打开（或直接调 API）填管理员账号 + 站点名 + 数据库 DSN（内置 SQLite 即可）就完事：

```bash
# 浏览器方式：打开 http://core:25774 走向导
# 命令行方式：
curl -X POST http://127.0.0.1:25774/api/install/complete \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"<强密码>","sitename":"homelab","metric_dsn":"file:/app/data/metrics.db?mode=rwc&_txlock=immediate"}'
```

> `metric_dsn` 是「指标库」的连接串，单机自用填这个 SQLite 路径就够；密码要求同时含大小写和数字。

然后把端口藏到之前那套 Caddy 入口后面，拿到一个免端口域名：

```caddy
https://komari.homemesh.internal {
    bind 100.64.12.1
    tls /certs/server.crt /certs/server.key
    reverse_proxy 127.0.0.1:25774
}
```

加一条 MagicDNS `extra_records`（`komari.homemesh.internal` → `100.64.12.1`），浏览器就能用 `https://komari.homemesh.internal` 打开面板了。到这里，server 就绪，但面板还空空如也——**数据得靠 agent 喂进来**。

## 装 agent：一个二进制全平台

agent 在 [komari-agent releases](https://github.com/komari-monitor/komari-agent/releases) 下，按平台拿二进制（`komari-agent-<os>-<arch>`）。先在面板的「节点」页建一台设备，拿到它的 token（token = 这台设备的身份），然后每台机器用同一句命令跑起来：

```bash
# Linux（systemd 常驻）
komari-agent \
  -e https://komari.homemesh.internal \
  -t <该设备的 token> \
  --disable-web-ssh --disable-auto-update
```

几个平台我都是这么装的：

- **Linux**：二进制放 `/usr/local/bin/komari-agent`，套一个 systemd unit，`enable --now` 常驻。
- **macOS**：同一个二进制，套一个 LaunchAgent plist，用户态跑起来即可（无需 sudo）。
- **Windows**：计划任务（`schtasks`）跑，设成开机以 SYSTEM 启动。

> 为什么 Windows 用计划任务而不是 `sc` 注册成服务？因为 agent 是控制台程序，不响应 Windows SCM 的握手，直接 `sc create` 会启动失败；用计划任务最简单可靠。

三台都跑起来后，回面板刷新，设备就一台台冒出来，开始逐秒上报了。

## 两个实操时踩的坑

**① 「地区 / 国旗」徽标不太对，多半是代理或墙的锅。** 面板上给每台设备标的地区，是 server 拿 agent 上报的「公网出口 IP」查 GeoIP 得到的。而 agent 探测公网 IP 的方式是去访问几个公网站点（`visa.cn`、`toutiao.com`、`ip.sb` 等）看返回里自己的 IP。问题就在这——**机器开了代理 / VPN，或者这几个站点连不通时，探测会失败或拿到代理节点的 IP**，于是地区要么空、要么显示成代理所在国。这只是个展示字段，不影响任何监控数据；想让它显示正确，给 agent 加 `--custom-ipv4` 手动指定一个真实出口 IP 即可。

**② agent 和 server 同居一台时，别让 agent 走公网域名回环。** agent 通过 WebSocket 连回 server。Caddy 这类反代对 WebSocket 的 `Upgrade` 头是原生透传的，跨机走域名完全没问题；但**同一台机器**上的 agent 去连 `https://自己的域名` 时，流量会绕一圈 tailscale0 回环，我的环境里就遇到 WebSocket 升级后流中断、客户端收到 200 而非 101 的怪现象。解法很简单：同机 agent 直接 `-e http://127.0.0.1:25774`，跨机才走域名。

## 一点安全上的建议

Komari 自带网页终端 / 远程执行的能力，但如果你只是拿它当「看板」用，建议统一给所有 agent 加两个开关：

- `--disable-web-ssh`：**关掉远程终端和远程执行**，面板变成纯监控，谁拿到面板也控制不了你的机器；
- `--disable-auto-update`：升级节奏收归自己，不让 agent 半夜自动更新版本。

再配合「server 只绑内网 + Caddy 反代只开 HTTPS」这套，监控数据和机器控制面就都留在自己的 Tailscale 局域网里了。

## 收尾

装上 Komari 之后，家里那几台机器终于有了一个统一的「仪表盘」：主控机、MacBook、游戏 PC、香橙派、云服务器，每台的 CPU / 内存 / 磁盘 / 网络占用一目了然，还能回看历史曲线，谁在偷偷吃资源、谁的盘快满了，一眼就有数。

和前三篇串起来，完整的家用基础设施就是：**Tailscale 组网连得上 → Caddy 域名用得到 → Sunshine 远程桌面控得住 → Komari 面板看得清**。

相关链接：

- [Komari 主仓库](https://github.com/komari-monitor/komari)（server + Web UI）
- [komari-agent](https://github.com/komari-monitor/komari-agent)（各平台 agent）
- [官方文档](https://www.komari.wiki/)