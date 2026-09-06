+++
title = 'Tailscale 组网下的高性能远程桌面：Sunshine + Moonlight + DP 诱骗器实战'
date = '2026-09-05T15:00:00+08:00'
draft = false
tags = ['Tailscale', 'Headscale', 'Sunshine', 'Moonlight', 'RustDesk', '远程桌面', 'Self-Hosted', '教程']
description = '给没有显示器的主机装上远程桌面：DP 诱骗器伪装一块屏幕，Sunshine 用 AMD 核显硬件编码推流，Moonlight 客户端经 Tailscale 点对点直连；对比 RustDesk 的软编高 CPU，并附自建 RustDesk 中继服务器方案。'

[cover]
  image = 'https://s3.blog.zeroicey.me/covers/tailscale-sunshine-moonlight-remote-desktop.jpg'
+++

上一篇《从零自建 Tailscale 网络》把家里和云端的机器都拉进了同一个虚拟局域网。网里的每一台设备现在都能互相点到为止地址了——但「能连」和「能像坐在机箱前一样操作」是两回事。

我有一台「运维中心」：一台没接任何显示器的 Arch Linux 服务器，上面跑着 Docker、Samba、SSH，塞在角落里常年不亮屏。它其实是有图形界面的（为了偶尔点进去看看、操作些带 GUI 的工具），只是——没有屏。这就是典型的「无头主机」（headless）。

很长一段时间，它的远程控制靠 **RustDesk**。能用，但有两个一直让我不爽的地方：被控时 **CPU 占用很高**，画面延迟也一般。

后来排查清楚了原因：RustDesk 在 Linux + AMD 核显上基本走**软件编码**——它这套软件对 NVIDIA（NVENC）、Intel（QSV）的硬件加速支持不错，唯独对 AMD 很弱。而我这台机器恰好是 AMD 核显（Radeon 780M），于是每次被连，CPU 都吭哧吭哧地软编视频。

既然 Tailscale 已经把传输层摆平了（点对点直连、自动打洞、还有 ACL），那我要的其实只剩一句话：**换一个会用显卡硬件编码的被控端**。答案就是给游戏串流设计的 **Sunshine + Moonlight**——拿来当远程桌面，意外地好。

这篇文章把完整方案记下来，分成三层：

1. **硬件层**：DP 诱骗器——怎么在没有显示器的情况下让显卡「相信」自己接了块屏；
2. **软件层**：Sunshine（被控端）+ Moonlight（客户端）的安装、配置与调参；
3. **对比层**：为什么 RustDesk 还在用、它自建的中继服务器（hbbs/hbbr）怎么搭。

> ⚠️ **替换说明**：与上一篇一致，文中所有 IP 都用 `100.64.12.x` 这段示例地址，主机名、用户名都是虚构的。照着做时替换成你自己的真实值。

## 先看全局：一条链路，三个角色

```
                    ┌─────────────────────────────────────────┐
                    │  core（被控端，无显示器的主服务器）      │
                    │    Arch Linux · AMD Radeon 780M 核显     │
                    │    ┌──────────────┐                      │
                    │    │   DP 诱骗器  │  ← 伪装成一块 1080p 屏 │
                    │    └──────┬───────┘                      │
                    │           │ DisplayPort                   │
                    │    ┌──────▼───────┐                      │
                    │    │ lightdm+bspwm│  ← 极简图形会话       │
                    │    └──────┬───────┘                      │
                    │    ┌──────▼───────┐                      │
                    │    │   Sunshine   │  ← 抓屏 + 硬编推流    │
                    │    └──────┬───────┘                      │
                    └───────────┼──────────────────────────────┘
                                │ Tailscale 点对点（100.64.12.1）
     ─────────── 虚拟局域网 ──────┼──────────────────────────────
              │                  │                  │
        ┌─────▼─────┐      ┌─────▼─────┐      ┌─────▼─────┐
        │  macbook  │      │   winpc   │      │   phone   │
        │  100.64.  │      │  100.64.  │      │  100.64.  │
        │   12.3    │      │   12.5    │      │   12.7    │
        │  Moonlight│      │  Moonlight│      │  Moonlight│
        └───────────┘      └───────────┘      └───────────┘
```

链路很简单：**Sunshine 在被控端抓屏并用核显硬编成视频流 → 经 Tailscale 点对点推到客户端 → Moonlight 解码显示，并把键鼠操作回传**。没有公网端口、没有中继服务器，全程只在本篇上一篇搭好的那个虚拟局域网里跑。

| 节点 | 角色 | 系统 | Tailscale 地址（示例） |
| --- | --- | --- | --- |
| core | 被控端（Sunshine 服务端） | Arch Linux + AMD 核显 | 100.64.12.1 |
| macbook | 客户端（Moonlight） | macOS | 100.64.12.3 |
| winpc | 客户端（Moonlight） | Windows 11 | 100.64.12.5 |
| phone | 客户端（Moonlight） | iOS | 100.64.12.7 |

## 硬件篇：DP 诱骗器，让显卡「以为」有块屏

### 为什么需要它

显卡的输出口（HDMI / DP）有个特性：它要通过一条 **EDID** 信息来「认识」显示器——分辨率、刷新率、厂商、能力都在里面。没有接显示器时，系统读不到 EDID，图形会话可能起不来，或者起来后分辨率异常（比如被困在 640×480），远程一看就是灾难。

对一台「有图形界面但永远不接屏」的主机，最稳的解法不是去折腾 X11 的虚拟输出，而是花几十块钱买一个 **DP 诱骗器（DisplayPort Dummy Plug / HDMI Dummy Plug）**：一个拇指大小的插头，内置一小片 EEPROM 存着一份 EDID，插到显卡口上，系统就以为接了一块真显示器。

本文用的是一块 1080p 的 DP 诱骗器，插上后 `xrandr` 看到的是：

```
DisplayPort-0 connected primary 1920x1080+0+0
   1920x1080     60.00*+  59.94  ...   ← EDID 主模式是 1080p@60
```

### 两个实操建议

1. **分辨率锁在 1080p 就够**。有些诱骗器 EDID 能上探到 4K，但远程控制的编码负担是跟着分辨率翻倍涨的，1080p 是办公远程最舒服的甜点，别为了「清晰」去调 4K。
2. **保留它，别想着用虚拟屏替代**。Sunshine 自己带虚拟显示器方案，但在 AMD 下配置更折腾，不如一块物理诱骗器来得可靠省心——它是整套「无头主机」的根基。

配合一个极简的图形栈，这台机器彻底变成「没有屏、但有桌面」的形态：

```
lightdm（自动登录） → Xorg :0 → bspwm 平铺窗口管理器 + polybar 状态栏
```

`lightdm` 配自动登录，是为了开机后不用人肉登录，Sunshine 直接就能抓到一个完整的桌面会话。相关配置贴在 `lightdm.conf`：

```ini
[Seat:*]
autologin-user=你的用户名
autologin-session=bspwm
autologin-user-timeout=0
```

## 被控端：安装与配置 Sunshine

### Step 1：确认显卡硬件编码可用

这是整个方案成立的前提——先验证你的显卡支持硬件编码（H.264/HEVC/AV1 里至少一两样）。装 `libva-utils`（提供 `vainfo`）实测：

```bash
sudo pacman -S libva-utils    # Arch；Debian/Ubuntu 是 sudo apt install vainfo
vainfo
```

看输出里有没有 `VAEntrypointEncSlice`（编码入口）。AMD 780M 核显的输出长这样，三样全齐：

```
VAProfileH264High               : VAEntrypointVLD
VAProfileH264High               : VAEntrypointEncSlice   ← H.264 硬编 ✓
VAProfileHEVCMain               : VAEntrypointEncSlice   ← HEVC 硬编 ✓
VAProfileAV1Profile0            : VAEntrypointEncSlice   ← AV1 硬编 ✓
```

只要不是只有 `VLD`（那只是解码）、没有 `EncSlice`，就说明能硬编。Sunshine 实际用的是更新一点的 Vulkan Video 编码（`h264_vulkan` / `hevc_vulkan` / `av1_vulkan`），效果一样，AMD 上它就是走这套。

### Step 2：安装 Sunshine

**坑来了**：Sunshine 在 Arch 官方仓库里没有，AUR 里的 `sunshine`/`sunshine-bin` 也常常让人等得抓狂（源码编译巨慢，AUR 元数据同步也慢）。最省事的路线是直接下 **GitHub 官方 release 的预编译包**：

```bash
# 1) 先查最新版本号
#    浏览器打开 https://github.com/LizardByte/Sunshine/releases/latest
#    或（下面这条命令国内直连比走代理更通畅，原因见踩坑记录）：
#    curl --noproxy '*' -s https://api.github.com/repos/LizardByte/Sunshine/releases/latest

# 2) Arch：下 .pkg.tar.zst 后本地装
sudo pacman -U sunshine-<版本>-x86_64.pkg.tar.zst

#    Debian/Ubuntu 用对应 .deb：
#    sudo dpkg -i sunshine-ubuntu-24.04-amd64.deb（缺依赖再 sudo apt-get install -f）
#    Fedora 用 .rpm、或直接装 Flatpak 版 org.lizardbyte.app.Sunshine
```

官方 release 会带一个 `sunshine.AppImage`，想零安装直接跑的也能用，但做服务还是包管理干净。

下载环节国内容易踩坑，具体见文末「坑 1」「坑 2」。

### Step 3：设置管理凭据 + 启动

```bash
# 设置 Web 管理面板的用户名/密码（后续调参靠它）
sunshine --creds <用户名> <密码>

# 启动（用户级 systemd，跟随图形会话自动起）
systemctl --user enable --now sunshine
systemctl --user status sunshine
```

注意：Sunshine 是**用户级服务**，要跑在你的图形会话里才能抓到屏幕（它抓的是 `DISPLAY=:0` 的 X11 桌面）。启动后看日志确认两件事：抓到显示器、启用了硬编：

```bash
journalctl --user -u sunshine -n 50
# 期望出现：
#   Detected display: DisplayPort-0 ... connected: true
#   Found HEVC encoder: hevc_vulkan [vulkan]
#   Found AV1 encoder: av1_vulkan [vulkan]
```

### Step 4：Sunshine 用到的端口

| 端口 | 协议 | 用途 |
| --- | --- | --- |
| 47984 | TCP | HTTP（发现/配对） |
| 47989 | TCP | HTTPS / GameStream 主端口 |
| 47990 | TCP | Web 管理 UI |
| 48010 | TCP | RTSP / 主连接 |
| 47998–48000 | UDP | 视频流（RTP） |
| 48002 | UDP | 控制流 |

⚠️ **在 deny-by-default 的 Tailscale ACL 里**（上一篇 Step 3.4 那种「未匹配一律拒绝」的策略），一定要把上面的端口加进「客户端 → core」的白名单，否则客户端会和 hpcore 一样被静默拒绝，连握手都到不了。加完之后重新 `headscale policy set` 一下：

```hujson
{
  "src": ["100.64.12.3", "100.64.12.5", "100.64.12.7"],
  "dst": ["100.64.12.1"],
  "ip": ["47984", "47989", "47990", "48010", "47998-48000", "48002"]
}
```

### Step 5：Web 管理面板怎么访问

Sunshine 的 Web UI（`47990`）默认有个安全限制：只允许**局域网/本机**来源访问。Tailscale 的地址属于 CGNAT 段（`100.64.0.0/10`），会被它当成「公网」拒掉（表现为 403）。两个办法：

1. 被控端本机直接开 `https://127.0.0.1:47990`；
2. 远程调参走 SSH 隧道（推荐）：

```bash
ssh -L 47990:localhost:47990 core
# 然后浏览器打开 https://localhost:47990
```

对日常「偶尔点进去用一下」来说，其实可以不碰 Web UI——默认的硬编 + 30fps 已经很够用，参数主要是在客户端 Moonlight 那边调（下一节）。

## 客户端：Moonlight 的设置

Moonlight 是官方客户端，各端都有：

| 端 | 获取方式 |
| --- | --- |
| Windows | 微软商店 / GitHub `moonlight-stream/moonlight-qt` |
| macOS | App Store / moonlight-qt |
| iOS | App Store「Moonlight Game Streaming」 |
| Android | Play / GitHub moonlight-android |

### 连接与配对

1. 客户端「Add Host / 添加主机」→ IP 填被控端的 Tailscale 地址 `100.64.12.1`，端口留默认（47989）。
2. 首次连接要**配对**：Sunshine 生成一个 4 位 PIN，客户端输入即可。无头主机看不到屏幕上的 PIN 提示，去被控端日志里抓：

```bash
journalctl --user -u sunshine -n 50 | grep -i pin
# Enter the following PIN on the Moonlight client: XXXX
```

### 参数怎么调（不打游戏，只做远程办公）

这是最容易纠结的地方。Moonlight 的默认预设是奔着「游戏高帧率」去的，对远程办公纯属浪费。我实测下来这套组合最舒服的一组是：

| 参数 | 推荐值 | 说明 |
| --- | --- | --- |
| 分辨率 Resolution | **1920×1080** | 与被控端诱骗器一致，别选 4K |
| 帧率 Frame rate | **30 FPS** | 办公 30fps 流畅够用，60fps 只对游戏有意义，还翻倍负担 |
| 码率 Bitrate | **10 Mbps**（HEVC）/ 20 Mbps（H.264） | 1080p 静态画面的甜点区，再高是浪费 |
| 视频编码 Codec | **HEVC (H.265)** | 服务端已硬编 HEVC，同码率比 H.264 清晰；选「自动」也行 |
| 硬件解码 | 开（默认即开） | Mac/iOS 走 VideoToolbox，Windows 走 D3D11VA，客户端也不吃 CPU |

一句话：**1080p + 30fps + HEVC + 10 Mbps**。觉得鼠标发飘时再往 60fps/20 Mbps 加，但日常办公那套已经足够，而且最省。

> 有个真实案例：同一台机器，有人把其中一台客户端的码率拉到了 77 Mbps，另一台是 7 Mbps，画面观感几乎没差——但前者纯是在给编码器和网络加无谓的负担。码率不是越高越好。

## 对比与共存：RustDesk 为什么还在

换到 Sunshine 之后，我并没有把 RustDesk 一锅端，原因值得说清楚——它俩**不是替代关系，是分工关系**。

先看一张对比表：

| 维度 | RustDesk | Sunshine + Moonlight |
| --- | --- | --- |
| 屏幕编码 | 软编为主（AMD 支持弱） | 硬编（Vulkan/VA-API） |
| 被控端 CPU | 高 | 极低 |
| 延迟 / 画质 | 中 | 低 / 好 |
| 网络 | 自建 hbbs/hbbr 中继 | P2P 直连（走 Tailscale） |
| headless 支持 | 一般 | 好（配合 DP 诱骗器） |
| 拿手场景 | 任意设备互连、无硬编设备 | 有硬编的主机做主力被控端 |

**RustDesk 的高 CPU 只在「被连那台是 AMD 核显的 Linux」时才会发作**。反过来，如果被连的是 Windows（NVENC）或 Mac（VideoToolbox），RustDesk 也会走硬件编码，体验并不差。这就解释了一个常见的困惑：

> 「那我 hpbook 连 hpstation（Windows 主力机）用 RustDesk，是不是也吃 AMD 的亏？」——不会。**编码永远发生在『被连那台』自己的客户端里**，用被连那台自己的显卡；中继服务器（hbbs/hbbr）本身不做任何编码、不碰任何显卡，它只是帮忙「互相找到 + 打不通时转发流量」。

所以这套部署里，RustDesk 保留了一个中继服务器，专门服务「其他设备之间互相连」。这个中继也是自建的，跑起来不复杂，下面把部署方式记下来。

### 自建 RustDesk 中继服务器（hbbs / hbbr）

RustDesk 的中继分两个组件：**hbbs**（ID 服务器，负责帮客户端互相发现、交换密钥、打洞）和 **hbbr**（中继服务器，打洞失败时兜底转发流量）。官方提供了 Docker 镜像，一个 compose 搞定：

```yaml
services:
  hbbs:
    container_name: hbbs
    image: rustdesk/rustdesk-server:latest
    command: hbbs -r <中继地址>:21117    # -r 指定 hbbr 的对外地址，客户端打洞失败时走它
    volumes:
      - ./data:/root                   # 数据卷存 id_ed25519 和 sqlite 数据库
    ports:
      - "21115:21115"                  # NAT 探测
      - "21116:21116"                  # TCP：ID 注册
      - "21116:21116/udp"              # UDP：打洞/注册
    depends_on:
      - hbbr
    restart: unless-stopped

  hbbr:
    container_name: hbbr
    image: rustdesk/rustdesk-server:latest
    command: hbbr
    volumes:
      - ./data:/root
    ports:
      - "21117:21117"                  # 中继流量
    restart: unless-stopped
```

启动后，**拿到 hbbs 的公钥**（客户端要照着填，否则连不上）：

```bash
docker compose up -d
docker logs hbbs | grep Key
# Key: l5+...=     ← 复制这段
# 或直接读数据卷里的 id_ed25519.pub
cat data/id_ed25519.pub
```

客户端（每一台要互连的设备）里配置**自定义服务器**：

- **ID 服务器**：填 `<服务器地址>:21116`
- **Key**：粘贴上面那串 hbbs 公钥

配好后，这些设备之间就能靠这台自建 hbbs/hbbr 找到彼此、建立连接了。它绑 `0.0.0.0`（或只绑 Tailscale 地址），走上一篇的 Tailscale 组网访问即可，不必暴露公网。

## 踩坑记录（都是真踩过的）

1. **GitHub 下 Sunshine 走代理反而不通**。很多自建代理对 `github.com` / `objects.githubusercontent.com` 的境外节点不通，curl 直接 15 秒超时，而**直连反而通**。口诀：下载 GitHub 资产先试 `--noproxy '*'` 直连，别默认走代理。
2. **GitHub 大文件直连易断流（curl 报 56）**。release 包下到一半被重置很常见，单线程 curl 反复失败。改用 `aria2c -x 16 -s 16` 多线程分段下载，几秒下完：

   ```bash
   http_proxy= https_proxy= all_proxy= aria2c -x 16 -s 16 -k 1M \
     -o sunshine.pkg.tar.zst "https://github.com/.../sunshine-<版本>-x86_64.pkg.tar.zst"
   ```

3. **Arch 官方仓库没有 sunshine，AUR 又慢**。`pacman -S sunshine` 直接报找不到；AUR 元数据同步国内直连可慢到 60 秒+。用 GitHub release 的 `.pkg.tar.zst` 走 `pacman -U` 最快。
4. **systemd 单元名不是 `sunshine.service`**。官方包的单元叫 `app-dev.lizardbyte.app.Sunshine.service`，直接 `systemctl --user enable sunshine` 会报「单元不存在」。先 `systemctl --user daemon-reload`，再 enable 那个全名（或 reload 后用 alias `sunshine.service`）。
5. **Tailscale 访问 Web UI 被 403**。`origin_web_ui_allowed` 默认 `lan`，Tailscale 的 CGNAT 段被当成公网。别去改宽松它，用 SSH 隧道访问最稳。
6. **deny-by-default ACL 拦掉了 Sunshine**。上一篇配的 ACL 是「未匹配一律拒绝」，Sunshine 那 6 组端口必须显式加进白名单，否则客户端连握手都通不了（表现为一直转圈连不上）。
7. **声音没有：`Couldn't connect to pulseaudio: Access denied`**。Sunshine 抓音频走 PipeWire 的 pulse 兼容层，用户级服务若握手不到 pulse socket 就没声音。纯办公远程控制通常不需要声音，可以无视；要修就往「让 Sunshine 服务能 access 到用户的 pipewire-pulse socket」方向查。
8. **别忘关合成器**。如果被控端跑着 picom 这类 X11 合成器（阴影、透明、动画），远程场景下它是纯开销——本地又没有屏看效果。关掉能再省一点 CPU：把 `~/.config/bspwm/bspwmrc` 里的 `picom -b &` 注释掉即可。
9. **卸载 RustDesk 时别误伤 PipeWire**。Arch 上 `pacman -Rns rustdesk` 会把 rustdesk 依赖的 pipewire 音频栈一并当孤儿删掉（`-s` 级联），而 Sunshine 音频靠它。正确做法是先看依赖再删，或删完记得把 `pipewire pipewire-audio pipewire-pulse alsa-card-profiles` 装回来。

## 安全建议

- Sunshine 全程走 Tailscale，**不暴露公网、不开端口转发**；被控端本机防火墙只对 tailscale0 放行那几个端口。
- ACL 里 Sunshine 端口只放给「你的客户端 IP」，维持最小权限。
- Web UI 密码用 `sunshine --creds` 设一个强口令；Web UI 只走本机或 SSH 隧道，别为图方便放宽 `origin_web_ui_allowed`。
- RustDesk 的 hbbs/hbbr 若只服务组网内设备，绑 Tailscale 地址即可，别绑 `0.0.0.0` 暴露公网。
- 分辨率锁 1080p、码率别超 20 Mbps，既省被控端 CPU，也省带宽和客户端解码负担。

## 小结

整件事的因果链其实很短：**RustDesk 在 AMD 核显上只能软编 → 高 CPU；Tailscale 已经解决了「怎么连」；Sunshine + Moonlight 解决了「用什么编」；DP 诱骗器解决了「没有屏」**。四样拼起来，一台塞在角落、从不亮屏的服务器就有了一个流畅、低占用的桌面。

硬件成本是一块几十块的诱骗器，软件成本为零（都是开源 + 已有组网）。换来的是被控端 CPU 从软编的几十个点降到个位数、画质延迟双提升，客户端在 Mac/Win/手机三处通用。

RustDesk 不必退场——把它当中继服务器留着，服务那些「没有硬编条件」的设备互连，和 Sunshine 各司其职。希望这篇记录对同样要远程控制无头主机的你有帮助。