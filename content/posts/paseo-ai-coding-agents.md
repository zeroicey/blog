+++
title = '把编程 AI 装进口袋：Paseo 自托管总控台实战'
date = '2026-09-06T22:00:00+08:00'
draft = false
tags = ['Paseo', 'AI Agent', 'Tailscale', '自托管', 'Claude Code', 'Codex', '教程']
description = 'Claude Code、Codex、Pi 这些命令行编程 agent，能不能不在终端前守着？Paseo 把它们统一到一个界面，agent 跑在自己的服务器上，手机随时遥控、追加指令。附 headless 部署与三个真踩过的坑。'
[cover]
  image = 'https://s3.blog.zeroicey.me/covers/paseo-ai-coding-agents.jpg'
+++

前面几篇把「基础设施」的活儿干得差不多了：把家里和云端的机器拉进同一个 Tailscale 虚拟局域网、给无头主机配上能远程操作的桌面、用 Caddy + MagicDNS 让每个服务都有免端口的 HTTPS 域名，还补上了服务器监控和 GitHub 加速。

但「连得上、管得住、看得见」之后，有一件事我一直觉得别扭——**真正用来写代码的那批 AI 工具，始终被困在终端里**。

Claude Code、Codex、IBM OpenCode、Pi……这些命令行编程 agent 几乎已经成了我的主力。它们能在你的仓库里读懂代码、改文件、跑测试、一条龙地完成任务。但它们有个共同的痛点：**一开跑，你就得守在终端前**——任务跑到一半想加个要求？想看进度？想换个模型重来？都得回到那台机器、那个终端窗口。

出门在外或者躺沙发上时，我能不能掏出手机，看看那个正在后台改代码的 agent 跑到哪了，顺手补一句「顺便把测试也补了」？

能。答案是一个叫 [Paseo](https://github.com/getpaseo/paseo) 的开源项目。

## Paseo 是什么

一句话：**Paseo 是一个自托管的「AI 编程 agent 总控台」。它把 Claude Code、Codex、GitHub Copilot、OpenCode、Pi 这些命令行 agent 统一到一个界面里，agent 仍然跑在你自己的机器上，而你用手机、桌面、浏览器或 CLI 随时随地遥控它们。**

它的几个卖点，恰好都踩在这系列文章一贯的「自托管」价值观上：

- **自托管**：agent 跑在你自己的机器上，用你自己的工具、配置、skills，数据不出本机。
- **多 provider**：Claude Code / Codex / Copilot / OpenCode / Pi 一个界面全收，按活儿挑最合适的模型。
- **跨设备**：桌面开工、手机盯进度、终端写脚本，无缝切换。
- **隐私优先**：没有遥测、没有追踪、不强制登录。

## 架构：一个 daemon，一堆客户端

Paseo 的架构非常清爽。核心是本机一个叫 **daemon** 的后台进程，负责真正去启动、管理那些 agent 进程；手机 App、桌面 App、网页、CLI 都是它的「遥控器」，通过网络连上去：

```text
  你的终端 agent（Claude Code / Codex / Pi ……）
        │  由 daemon 启动、喂参数、收输出
        ▼
  ┌──────────────────────────────┐
  │  core · paseo daemon          │
  │  （监听 6767，密码保护）        │
  └──────────┬───────────────────┘
             │  HTTPS / Tailscale / 本机
     ┌───────┼─────────┬──────────┐
     ▼       ▼         ▼          ▼
   手机App  桌面App    网页 UI     CLI
```

daemon 还开放 WebSocket API，第三方可以写脚本、做集成（比如 issue 机器人、看板、编排服务）。

> ⚠️ **替换说明**：与前几篇一致，文中主机名（`core`）、内部域（`homemesh.internal`）、Tailscale IP（`100.64.12.x`）均为虚构示例，照着做时换成你自己的真实值。Paseo 本身是开源项目，`6767` 是它的默认端口、`paseo` 是命令名，这些是公开事实。

## 部署：headless 服务器三步

我把它跑在一台常开的「主机」上（沿用这系列里的 `core`）。官方对「服务器 / 远程机」这类没图形界面的场景，推荐用 CLI 方式（而不是当桌面应用来装）：

```bash
npm install -g @getpaseo/cli
paseo daemon start
```

但这只是「前台跑一下」，重启就没了。要长期常驻，交给 systemd 更省心：

```ini
[Unit]
Description=Paseo daemon
After=network-online.target

[Service]
Type=simple
EnvironmentFile=/etc/paseo/paseo.env   # 里面放 PASEO_PASSWORD=…
ExecStart=/usr/local/bin/paseo daemon start --foreground --listen 100.64.12.1:6767 --web-ui
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

三个要点：`--foreground` 让 systemd 接管前台进程；`--listen` 绑定到 Tailscale 网卡的地址，只在组网里可达、不暴露公网；`--web-ui` 打开自带的网页控制台。密码放在独立的 env 文件里，别写进命令行。

前置条件是**机器上至少装好一个 agent CLI 并且已经登录**——比如 Claude Code 或 Codex。Paseo 自己不带 agent，它只是「调度员」。

## 手机怎么连

手机上不需要装什么服务器组件，就是个普通 App。我走的是 Tailscale 直连（这系列的「老熟路」）：

1. 手机装好 Tailscale，能 `ping` 通主机；
2. 打开 Paseo App → **Direct connection**（直连）；
3. 填 `100.64.12.1:6767`，再输 daemon 密码。

连上之后，你就能在手机上看到所有正在跑的 agent，点进去是实时滚动的终端输出，还能直接发新任务、给正在跑的任务追加一句「顺便把测试也补了」。App 还支持语音下任务，走路遛弯也能派活。

## 三个真踩过的坑

部署看着简单，但真上车还是被几个小细节绊了一下。都是翻源码、看日志才定位的，写下来帮你省时间：

### 1. daemon 明明起来了，CLI 就是连不上

现象：`paseo daemon start --listen 100.64.12.1:6767` 后，日志清清楚楚写着 `Server listening on http://100.64.12.1:6767`，但紧接着 `paseo ls` 报：

```text
Cannot reach the daemon at localhost:6767: Transport closed (code 1006)
```

**原因**：`--listen` 这个参数只影响「这次运行时」，并不写回持久化的 `config.json`。于是 daemon 听着 `100.64.12.1:6767`，而 CLI 读完 `config.json` 里那个默认的 `127.0.0.1:6767` 之后，跑去连 `localhost:6767`——那里根本没人听，握手直接被关。

**解法**：把目标监听地址直接写进 `~/.paseo/config.json` 的 `daemon.listen` 字段，跟启动参数保持一致。

### 2. systemd 里 agent 起不来，因为 node 版本「漂」了

现象：daemon 起来了，但让它启动 agent 时失败，`paseo provider diagnostic` 显示找不到 agent 的二进制。

**原因**：systemd 的 *user* 单元默认 `PATH=/usr/local/bin:/usr/bin`。如果你（跟我一样）用 nvm / fnm / asdf 这类版本管理器装 node，全局安装的 `paseo`、`pi` 这些二进制其实不在这个 PATH 里；更坑的是系统里若还有个 `/usr/bin/node`，它的版本很可能跟你日常用的（agent 按这个版本装的）对不上，两个版本打架。

**解法**：在 unit 里显式写 `Environment=PATH=...`，把你版本管理器的 node 目录和全局 bin 目录排到最前：

```ini
Environment=PATH=/home/you/.local/share/fnm/node-versions/v22/bin:/home/you/.npm-global/bin:/usr/local/bin:/usr/bin
```

部署完自检一句：`paseo provider diagnostic <provider>`，看到 `Status: Ready` 才算真就绪。

### 3. 开了 deny-by-default 的 ACL，客户端静默连不上

现象：手机端怎么填都连不上，但主机本机 curl 一切正常。

**原因**：这系列的组网里我配了 deny-by-default 的 ACL——个人设备到主机只放行一个端口白名单。新服务的端口不在名单里，就会被静默拒绝（不是超时，就是连不上，很迷惑）。

**解法**：在 ACL 策略里，把 6767 加进「个人设备 → 主机」的端口白名单，`policy set` 生效后客户端重连即可。这也提醒自己：**以后每加一个自托管服务，除了部署、防火墙，还要顺手在 ACL 里放行它的端口**——三处缺一不可。

## 小结

Paseo 刚好补上了我这套「自托管全家桶」缺的那一块：**让 AI 编程 agent 也变成可以随时随手遥控的服务**。它的价值不是又造了一个 agent（agent 还是 Claude Code、Codex、Pi 它们），而是把这些散落在一个个终端窗口里的智能，收进一个统一、可远程、可脚本化的界面。

如果你也和我一样，习惯把东西跑在自己的机器上、讨厌被某个终端窗口「拴住」，值得一试。`npm install -g @getpaseo/cli`，五分钟就能跑起来；配合 Tailscale，把家里的主机变成你口袋里的 AI 编程工作台。