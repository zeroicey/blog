+++
title = '从零自建 Tailscale 网络：Headscale + 免域名自签证书实战'
date = '2026-09-05T12:00:00+08:00'
draft = false
tags = ['Tailscale', 'Headscale', 'VPN', 'Self-Hosted', '组网', '教程']
description = '没有备案域名也能自建 Tailscale 网络：用 Headscale 自托管控制面 + 嵌入式 DERP，以公网 IP + 自签 CA 免域名组网，并安装 Headplane 管理面板，附完整命令与踩坑记录。'

[cover]
  image = 'covers/self-host-tailscale-headscale-no-domain.jpg'
+++

Tailscale 是近几年最好用的组网工具之一：把分布在不同网络的设备（家里的主机、笔记本、手机、公司/海外云服务器）拉进同一个虚拟局域网，点对点直连、自动打洞、内置 ACL 和 MagicDNS。但它默认连的是官方协调服务器（control plane），控制面不在自己手里。

想自托管控制面，标准答案是 [Headscale](https://github.com/juanfont/headscale)——一个 Tailscale 控制面的开源实现。可问题来了：**几乎所有 Headscale 教程都默认你有域名 + 备案 + 公网 HTTPS**。在很多地区（比如国内服务器），域名备案麻烦、周期长，甚至因为各种原因下不来。

这篇文章记录的是一条**完全没有域名**的路线，也是我自己这套网络实际采用的方案：

- **控制面**：用公网 IP + 自签 CA 直连，绕开域名和备案
- **DERP 中继**：用 headscale 内置的「嵌入式 DERP」，复用同一张证书
- **MagicDNS**：用内部命名空间（`xxx.internal`），不碰公网 DNS，也不需要备案
- **管理面板**：装一个 Headplane，用 Web UI 管节点、用户和 ACL

全文按四步铺开：**① 搭 DERP 服务器 → ② 各节点装 Tailscale 接入 → ③ 配置 Headscale → ④ 装 Headplane 面板**。每一步都给了可直接照抄的命令，文末把踩过的坑一起列了。

> ⚠️ **替换说明**：本文所有公网 IP（`203.0.113.10`）都是 RFC 5737 保留的示例地址，主机名、内网 IP、用户名也都是虚构的。照着做时请替换成你自己的真实地址。

## 最终拓扑

先看全局，后面每一步都是在往这张图里填东西：

```
                公网 VPS（控制面 + 嵌入式 DERP 同机）
        ┌────────────────────────────────────────────────┐
        │  relay-1 · Ubuntu 24.04                        │
        │    Headscale 0.29.3      ← tcp 443             │
        │    Embedded DERP         ← tcp 443 + udp 3478 │
        │    自签 CA，证书 SAN = 203.0.113.10            │
        └───────────────────┬────────────────────────────┘
                            │ 公网 IP 203.0.113.10
   ──────────── 互联网 ──────┼──────────────────────────────
        │            │             │            │
   ┌────▼───┐   ┌────▼───┐   ┌─────▼───┐   ┌────▼────┐
   │ core    │   │ macbook│   │  winpc  │   │ cloud-eu│
   │100.64.  │   │100.64. │   │100.64.  │   │100.64.  │
   │ 12.1    │   │ 12.3   │   │ 12.5    │   │ 12.2    │
   └─────────┘   └────────┘   └─────────┘   └─────────┘
   （另有 workcloud / nas / phone / pad 四台，稍后补齐）
```

虚拟网段用 headscale 默认的 CGNAT 段 `100.64.0.0/10`，节点地址由控制面从这个池子里分配（本文示例统一用 `100.64.12.x`，实际以 `tailscale status` 显示为准）。

| 节点 | 角色 | 系统 | Tailscale 地址（示例） |
| --- | --- | --- | --- |
| relay-1 | 控制面 + DERP（公网 VPS） | Ubuntu 24.04 | 非节点 |
| core | 家里的主服务器 | Arch Linux | 100.64.12.1 |
| macbook | 日常笔记本 | macOS | 100.64.12.3 |
| winpc | 游戏/主力机 | Windows 11 | 100.64.12.5 |
| cloud-eu | 海外云服务器 | Linux | 100.64.12.2 |
| workcloud | 公司云服务器 | Linux | 100.64.12.4 |
| nas | 内网另一台主机 | Arch Linux | 100.64.12.6 |
| phone | 手机 | iOS | 100.64.12.7 |
| pad | 平板 | Android（HyperOS） | 100.64.12.8 |

## 前置准备：一台有公网 IP 的服务器

整个方案需要一个「锚点」：一台**有固定公网 IP 的云服务器**（任何厂商的最小配置都够），它就是控制面和 DERP 的家。这台机器后续记作 `relay-1`，公网 IP 示例为 `203.0.113.10`。

需要提前做的两件事：

1. **开安全组 / 防火墙端口**（公网入方向）：

| 端口 | 协议 | 用途 |
| --- | --- | --- |
| 443 | TCP | headscale 控制面 + DERP（复用同一个端口） |
| 3478 | UDP | STUN（NAT 打洞 / DERP 发现） |

2. **确认服务器内防火墙**：如果开了 iptables/nftables，保证以上端口放行；最简单是 INPUT 默认 ACCEPT（云厂商安全组兜底）。

> 为什么 DERP 和控制面要放同一台？因为 headscale 的嵌入式 DERP **复用控制面那张 TLS 证书**，客户端只要信任了你的 CA，控制面和 DERP 的证书验证就一起过了，省去额外证书和端口。这也是「免域名」方案能成立的关键之一。

---

## Step 1：搭建 DERP 服务器（顺带把控制面立起来）

先澄清一个概念：这里说的「DERP 服务器」不是单独跑一个 derper 程序，而是 **headscale 自带的嵌入式 DERP**。它和控制面是同一个进程、同一个端口，开箱即用，配置只在 `config.yaml` 里改几行。所以这一步实际上 = **装 headscale + 生成自签证书 + 启用嵌入式 DERP**。

### 1.1 安装 headscale

以 relay-1（Ubuntu 24.04）为例：

```bash
# root 执行
VERSION=0.29.3
wget -O headscale.deb \
  "https://github.com/juanfont/headscale/releases/download/v${VERSION}/headscale_${VERSION}_linux_amd64.deb"

# 如果 GitHub release 直连慢/超时（国内服务器常见），走镜像加速：
# wget -O headscale.deb \
#   "https://ghfast.top/https://github.com/juanfont/headscale/releases/download/v${VERSION}/headscale_${VERSION}_linux_amd64.deb"

dpkg -i headscale.deb
systemctl enable headscale
```

DEB 包会自动创建 `headscale` 系统用户和 `/etc/headscale`、`/var/lib/headscale` 目录。

### 1.2 生成自签证书（「免域名」的核心）

没有域名、没有 Let's Encrypt，就用一张**自签证书**来给控制面做 TLS——只要把**公网 IP 写进证书的 SAN**，客户端就能用 `https://<IP>` 直连。

```bash
mkdir -p /etc/headscale/ca && cd /etc/headscale/ca

# ① CA 私钥 + 自签根证书（根证书可以签长得久一点）
openssl ecparam -genkey -name prime256v1 -out ca.key
openssl req -new -x509 -key ca.key -sha256 -days 3650 -out ca.crt \
  -subj "/CN=HomeMesh Root CA"

# ② 服务器私钥 + CSR
openssl ecparam -genkey -name prime256v1 -out server.key
openssl req -new -key server.key -out server.csr -subj "/CN=relay-1"

# ③ 扩展文件：SAN 写公网 IP（多端点就把多个 IP/域名都写进来）
cat > extfile.cnf <<'EOF'
subjectAltName = IP:203.0.113.10
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
EOF

# ④ 用 CA 签服务器证书
#    ⚠️ 有效期必须 ≤ 397 天：Apple 设备强制 398 天上限，签 10 年会被 macOS/iOS 拒绝
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -days 397 -sha256 -extfile extfile.cnf

# ⑤ 权限：CA 私钥只归 root；服务器私钥要给 headscale 用户读
chown root:root ca.key && chmod 600 ca.key
chgrp headscale server.key server.crt && chmod 640 server.key server.crt
```

生成后你有三样要分发/留存的产物：

- `ca.crt` —— 发到**每一台客户端**装进系统信任库（后面 Step 2 反复用）
- `server.crt` / `server.key` —— 只放在 relay-1，给 headscale 用
- `ca.key` —— 只留在 relay-1，**绝不能外传**

> **如果将来有域名了怎么办？** 那就简单多了：不要自签，改在 `config.yaml` 里填 `tls_letsencrypt_hostname` + `acme_email`，让 headscale 自己申请 Let's Encrypt 证书，客户端也就**不用**手动装 CA。自签方案是「没有域名」时的替代路线。

### 1.3 配置 config.yaml 并启用嵌入式 DERP

编辑 `/etc/headscale/config.yaml`，关键字段如下（完整字段见官方 config-example，这里只列与本方案强相关的）：

```yaml
# ── 控制面地址：直接用公网 IP，这就是免域名的关键 ──
server_url: https://203.0.113.10
listen_addr: 0.0.0.0:443

# ── 自签证书路径（Step 1.2 生成的）──
tls_cert_path: /etc/headscale/ca/server.crt
tls_key_path: /etc/headscale/ca/server.key

# ── 虚拟网地址池：headscale 默认 CGNAT 段 ──
prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48
  allocation: sequential

# ── 嵌入式 DERP（Step 1 的主角）──
derp:
  server:
    enabled: true
    region_id: 999
    region_code: "homemesh"
    region_name: "HomeMesh Embedded DERP"
    verify_clients: true
    stun_listen_addr: "0.0.0.0:3478"          # STUN，配合 3478/udp 安全组
    private_key_path: /var/lib/headscale/derp_server_private.key
    automatically_add_embedded_derp_region: true
    ipv4: 203.0.113.10                          # 告诉客户端 DERP 的公网地址
  urls:
    - https://controlplane.tailscale.com/derpmap/default   # 同时保留官方 DERP 作兜底
  paths: []
  auto_update_enabled: true
  update_frequency: 3h

# ── 数据库：起步用 sqlite 最省事 ──
database:
  type: sqlite
  sqlite:
    path: /var/lib/headscale/db.sqlite

# ── DNS / MagicDNS：内部命名空间，不碰公网 DNS、无需备案 ──
dns:
  magic_dns: true
  base_domain: homemesh.internal
  override_local_dns: false          # ⚠️ 保持 false，见文末「坑 8」
  nameservers:
    global:
      - 1.1.1.1
      - 1.0.0.1

# ── ACL 存进数据库，方便用 Headplane 面板改 ──
policy:
  mode: database

log:
  level: info
```

几个要点说清楚：

- `server_url` 是**控制面的对外地址**，客户端 `tailscale up` 时登录的就是它。这里直接写 `https://公网IP`，无需域名。
- `derp.server.enabled: true` 打开嵌入式 DERP；`region_code` 是自定义标识，`ipv4` 填公网 IP。
- `base_domain` 只是 tailnet 的**内部命名空间**，客户端用内置 resolver（`100.100.100.100`）解析，完全不经过公网 DNS。
- `override_local_dns: false` 让 MagicDNS 只接管 `*.homemesh.internal`，公网域名各节点照常用本地 DNS 解析。

### 1.4 启动并验证控制面 + DERP

```bash
# 改完配置用 restart（不要用 enable --now，DEB 安装后服务已经在跑了）
systemctl restart headscale
systemctl status headscale

# 验证 443 控制面 + 自签证书
curl -sk https://203.0.113.10/ | head -n 5

# 验证 STUN 端口在监听
ss -ulnp | grep 3478

# 常看日志
journalctl -u headscale -f
```

`curl` 用 `-k` 是因为自签证书，能正常返回内容就说明 TLS 起来了。至此，**控制面 + DERP + STUN 已经就绪**，可以开始拉节点入网了。

---

## Step 2：各节点安装 Tailscale 并接入

这一步在每台设备上重复做四件事：**装 Tailscale → 装 CA → 重启 tailscaled → `tailscale up` 接入**。顺序不能反，尤其是「装 CA 之后必须重启 tailscaled」——tailscaled 是 Go 程序，系统 CA 池在**进程启动时加载一次**，不重启它就不认识你的自签 CA，会报 `x509: certificate signed by unknown authority`。

### 2.1 Linux（core / cloud-eu / workcloud / nas）

以 Debian/Ubuntu 为例：

```bash
# ① 装 Tailscale 客户端
curl -fsSL https://tailscale.com/install.sh | sh
# Arch Linux 则：
# sudo pacman -S tailscale

# ② 装自签 CA（把 relay-1 的 ca.crt 先拷到本机）
#    Debian/Ubuntu：
sudo cp ~/ca.crt /usr/local/share/ca-certificates/homemesh-ca.crt
sudo update-ca-certificates
#    Arch Linux：
# sudo cp ~/ca.crt /etc/ca-certificates/trust-source/anchors/homemesh-ca.crt
# sudo trust extract-compat

# ③ 重启 tailscaled —— 必需！让 CA 池重新加载
sudo systemctl restart tailscaled

# ④ 接入（--authkey 在 Step 3.1 会讲怎么生成）
sudo tailscale up \
  --login-server=https://203.0.113.10 \
  --authkey=<PREAUTH_KEY> \
  --hostname=core
```

### 2.2 macOS

```bash
# ① 用 brew 装 CLI 版（不要用 App Store 版——它锁死官方控制面，没有 --login-server）
brew install tailscale

# ② 把 CA 加进系统钥匙串（放进 System 钥匙串即被当作信任锚点 AnchorTrusted）
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain ~/ca.crt

# ②' 验证系统现在信任这张 CA（不再报证书错误即成功）
security verify-cert -c ~/ca.crt
curl -I https://203.0.113.10

# ③ 重启 tailscaled（brew 服务）——必需，重载 CA 池
sudo brew services restart tailscale
# 或：sudo launchctl kickstart -k system/homebrew.mxcl.tailscale

# ④ 接入
sudo tailscale up \
  --login-server=https://203.0.113.10 \
  --authkey=<PREAUTH_KEY> \
  --hostname=macbook
```

> macOS 有个专属坑：**服务器证书有效期必须 ≤ 397 天**。之前签 10 年证书，Linux 客户端一切正常，macOS 却反复 `Trust evaluate failure: [leaf OtherTrustValidityPeriod]`。根因是 Apple 强制 398 天上限，所以 Step 1.2 里 `-days 397`。CA 本身可以签 10 年，只要服务器（leaf）证书短就没事。

### 2.3 Windows

1. 从官网下载官方 `.msi` 安装。
2. 导入 CA 到「受信任的根证书颁发机构」：

```powershell
# 管理员 PowerShell
Import-Certificate -FilePath .\ca.crt -CertStoreLocation Cert:\LocalMachine\Root
Restart-Service Tailscale
```

3. 连接：托盘图标 → 登录 → 右上角 **•••** → **Use custom coordination server** → 填 `https://203.0.113.10`。或者命令行：

```powershell
tailscale up --login-server=https://203.0.113.10 --authkey=<PREAUTH_KEY> --hostname=winpc
```

### 2.4 iOS

官方 App Store 版 Tailscale **已原生支持自定义协调服务器**，无需第三方 App：

1. 把 `ca.crt` 弄到手机上（AirDrop / 下载链接 / 临时 HTTP 服务都行），用 Safari 打开并安装描述文件。
2. 打开**完全信任**（容易漏的步骤）：设置 → 通用 → 关于本机 → **证书信任设置** → 找到你的 CA → 打开开关。
3. 验证：Safari 重开 `https://203.0.113.10`（先划掉旧标签页防缓存），不再弹「此连接非私人连接」= 系统层信任 OK。
4. 重开 Tailscale App → Log in → ••• → **Use custom coordination server** → `https://203.0.113.10` → 认证页选用户。

> ⚠️ **「装了描述文件」≠「信任了证书」**。漏开完全信任时，Safari 一直报「此连接非私人连接」，控制面日志会刷 `tls: bad certificate`，登录反复失败。详见文末「坑 7」。

### 2.5 Android

官方 Play / F-Droid 版 Tailscale **同样原生支持自定义协调服务器**，无需第三方 App：

1. 把 `ca.crt` 弄到平板上（下载链接 / 临时 HTTP 服务都行）。
2. **装成「CA 证书」**（关键，别点错类型）：设置 → 密码与安全 → 系统安全 → 加密与凭据 → 安装证书 → **CA 证书** → 选下载的 `ca.crt`。（各品牌路径略有差异，认准「CA 证书」四个字。）
3. 验证装对：设置 → … → 加密与凭据 → 信任的凭据 → 「用户」标签页，应能看到你的 CA（如 `HomeMesh Root CA`）；看不到就是装错类型。
4. 打开 Tailscale App → 右上角设置/头像 → Accounts → 右上角 ⋮ → **Use an alternate server** → 填 `https://203.0.113.10`。
5. 再点 ⋮ → **Use an auth key** → 粘贴 preauth key。
6. 回主界面 Log in / Connect。

> ⚠️ **Android 的坑和 iOS 不一样**：iOS 是「装描述文件 → 开完全信任开关」；Android 没有那个开关，**信任取决于证书装成哪一类**——必须装成「CA 证书」（进入系统信任锚），不能装成「VPN 和应用」证书（那只是应用/私钥凭据，App 不认，控制面日志同样刷 `tls: bad certificate`）。点文件名直接安装很容易默认装成后者。详见文末「坑 9」。

> ⚠️ 另外 Android 官方 App 对自签证书还有一个**上游已知 bug**：即使 CA 装对，DERP 中继连接可能仍不信任自签 CA，健康提示 `no-derp-connection` / `tls-connection-failed`。实际影响：平板与其它节点**同局域网时直连正常**，但出门用流量/别处 Wi-Fi、要经 DERP 跨网时可能不通。要移动端随时随地可用，建议给控制面换一张公网可信证书（Let's Encrypt / 域名）替代自签。

### 2.6 验证接入

```bash
tailscale status                 # 看节点和在线状态
tailscale ip -4                  # 本机分配的 Tailscale 地址

tailscale ping 100.64.12.2       # 看路径：p2p 为直连，via <地址> 为走 DERP 中转
getent hosts core.homemesh.internal   # 验证 MagicDNS
```

全部节点接入后，`relay-1` 上 `headscale nodes list` 应该能看到每一台，且已分配示例地址 `100.64.12.x`。

---

## Step 3：配置 Headscale（不走官方面板）

到这一步，网已经通了。接下来是把它「配置成自己想要的形状」：用户、预授权密钥、MagicDNS、ACL。所有操作都在 relay-1 上用 `headscale` CLI 完成——这也正是「不走官方协调服务器」的意义：一切由你掌控。

### 3.1 用户与预授权密钥（preauthkey）

```bash
# 建用户（节点都归属到这个 user 名下）
sudo headscale users create sam
sudo headscale users list
# id | name
# 1  | sam

# 生成预授权密钥——节点接入时把它当 --authkey 用
# ⚠️ 0.29.x 里 preauthkeys create 的 --user 要「数字 ID」（users list 里的 id 列），
#    填名字会报 strconv.ParseUint 错误
sudo headscale preauthkeys create --user 1 --expiration 24h
# 输出里那串 key.xxx = <PREAUTH_KEY>，复制到 Step 2 用

sudo headscale preauthkeys list --user 1
```

### 3.2 节点管理

```bash
sudo headscale nodes list          # 看在线状态 / 分配 IP
sudo headscale nodes delete <id>   # 移除节点
sudo headscale nodes expire <id>   # 使节点过期

# 改完 config.yaml 一定 restart
sudo systemctl restart headscale
sudo journalctl -u headscale -f
```

### 3.3 配置 MagicDNS

MagicDNS 是 headscale 内置的私有 DNS：让每台设备用**内部域名**互访，不用死记 IP。它的解析不经过公网 DNS、也不需要备案，所以免域名方案照常能用。

配置全在 `config.yaml` 的 `dns` 段（Step 1.3 已给），关键字段逐条说：

```yaml
dns:
  magic_dns: true                 # 总开关：打开内置 DNS
  base_domain: homemesh.internal  # 内部命名空间，节点域名 = <hostname>.<base_domain>
  override_local_dns: false       # ⚠️ 只接管 .internal，公网域名仍走本地 DNS（见坑 8）
  nameservers:
    global:                       # 兜底上游（内部记录由在线节点表应答，一般不命中这里）
      - 1.1.1.1
      - 1.0.0.1
    split: {}                     # 可选：按域名前缀分流到指定 DNS
  search_domains: []              # 可选：搜索域，如 [homemesh.internal]
  extra_records: []               # 可选：手写静态 A/AAAA 记录
```

工作机制（配合理解）：

- 客户端连上控制面后会拿到一个内置 resolver 地址 `100.100.100.100`，`*.homemesh.internal` 的查询被路由给它，由 headscale 依据**当前在线节点表**应答。
- 节点域名 = `--hostname 的值 + base_domain`。例如 `--hostname=core` → `core.homemesh.internal`，所以 hostname 别用空格/特殊字符。
- 想加一条不占节点的纯静态记录，用 `extra_records`：

```yaml
dns:
  extra_records:
    - name: "dash.homemesh.internal"
      type: "A"
      value: "100.64.12.1"
```

改完 config 要 `systemctl restart headscale`；个别客户端没拿到新记录时，重启它的 tailscaled 强拉。验证：

```bash
# 任意节点上
getent hosts core.homemesh.internal    # → 100.64.12.1
curl http://core.homemesh.internal:8080/
nslookup dash.homemesh.internal        # 自定义记录
```

### 3.4 ACL（默认拒绝，按需放行）

headscale 的 ACL 从 0.29 起用 `grants` 格式，**未匹配一律拒绝**（deny-by-default），是最小权限的好基础。每条 grant 三个字段：

- `src`：来源地址（组）
- `dst`：目标地址（组）
- `ip`：放行的目标端口，或 `*` 表示全部

只要「来源 × 目标 × 端口」不被任何一条 grant 命中，连接就被拒。若 ACL 存数据库（`policy.mode: database`），还能直接在 Headplane 面板里改。

写一个策略文件 `/etc/headscale/acl-policy.hujson`：

```hujson
{
  "grants": [
    // 1) 服务器之间互访：全放行
    {
      "src": ["100.64.12.1", "100.64.12.2", "100.64.12.4", "100.64.12.6"],
      "dst": ["100.64.12.1", "100.64.12.2", "100.64.12.4", "100.64.12.6"],
      "ip": ["*"]
    },
    // 2) 个人设备（笔记本/台式机/手机/平板）之间互访：全放行
    {
      "src": ["100.64.12.3", "100.64.12.5", "100.64.12.7", "100.64.12.8"],
      "dst": ["100.64.12.3", "100.64.12.5", "100.64.12.7", "100.64.12.8"],
      "ip": ["*"]
    },
    // 3) 主服务器 core → 个人设备：全放行（方便远程排障）
    {
      "src": ["100.64.12.1"],
      "dst": ["100.64.12.3", "100.64.12.5", "100.64.12.7", "100.64.12.8"],
      "ip": ["*"]
    },
    // 4) 个人设备 → core：只放行白名单端口（示例，按需增删）
    {
      "src": ["100.64.12.3", "100.64.12.5", "100.64.12.7", "100.64.12.8"],
      "dst": ["100.64.12.1"],
      "ip": ["22", "443", "3000", "8080", "25565"]
    },
    // 5) 个人设备 → nas：只放行 SSH
    {
      "src": ["100.64.12.3", "100.64.12.5", "100.64.12.7", "100.64.12.8"],
      "dst": ["100.64.12.6"],
      "ip": ["22"]
    }
  ]
}
```

先干跑校验再写入，最后查看生效结果：

```bash
# 先校验语法/结构，无误再 set
sudo headscale policy check -f /etc/headscale/acl-policy.hujson
sudo headscale policy set -f /etc/headscale/acl-policy.hujson
sudo headscale policy get      # 查看当前生效策略
```

> 思路：**服务器组之间、个人设备组之间互通；系统主服务器对其他服务器有完全控制权；个人设备访问主服务器的服务时走端口白名单**。未列出的（比如数据库 5432）默认全拒。

---

## Step 4：安装 Headplane（自己的 Web 面板）

headscale 本身只有 CLI，没有图形界面。日常点几下就能看节点、删设备、发 preauthkey、改 ACL 的需求，交给 **Headplane** 解决——它通过 headscale 的 HTTP API 来管理，不需要和 headscale 装在同一台机器上。

我把它跑在主服务器 `core` 的 Docker 里（`core` 在 Tailscale 网内，面板走内网访问，不暴露公网）。

### 4.1 准备 headscale API key

Headplane 登录靠 headscale 的 API key，在 relay-1 上生成：

```bash
sudo headscale apikeys create --expiration 720h
# 输出：<prefix>.<secret>，secret 只在这次打印，复制整行去 Headplane 登录用
sudo headscale apikeys list       # 之后只能看到前缀（掩码），丢了就重新 create 一把
```

### 4.2 部署 Headplane（Docker Compose）

在 core 上建目录 `/opt/headplane`，写 `compose.yaml`：

```yaml
services:
  headplane:
    image: ghcr.io/tale/headplane:0.7.1
    container_name: headplane
    restart: unless-stopped
    environment:
      # headscale 用的是自签 CA，给容器的 Node 出站请求喂 CA
      NODE_EXTRA_CA_CERTS: /etc/headplane/ca.crt
      TZ: Asia/Shanghai
    ports:
      # 只绑本机 + Tailscale 网，不暴露公网
      - "127.0.0.1:8088:3000"
      - "100.64.12.1:8088:3000"
    volumes:
      - ./config.yaml:/etc/headplane/config.yaml:ro
      - /etc/ca-certificates/trust-source/anchors/homemesh-ca.crt:/etc/headplane/ca.crt:ro
      - /data/headplane:/var/lib/headplane
```

配套 `config.yaml`（和 compose 同目录）：

```yaml
server:
  host: "0.0.0.0"
  port: 3000
  # 走 Tailscale 网访问，让 Mac/手机也能开面板
  base_url: "http://100.64.12.1:8088"
  # 32 字符，openssl rand -hex 16 生成
  cookie_secret: "<paste>"
  # 纯 HTTP（Tailscale 已加密传输），必须 false，否则浏览器拒发 cookie
  cookie_secure: false
  data_path: "/var/lib/headplane"

headscale:
  # headscale 的 HTTP API（控制面同端口 443，自签 TLS）
  # 不自带 tls_cert_path，改为用 NODE_EXTRA_CA_CERTS 信任整个自签 CA（见 compose.yaml）
  url: "https://203.0.113.10"
```

启动并验证：

```bash
cd /opt/headplane
openssl rand -hex 16   # 生成 cookie_secret，填进 config.yaml
docker compose up -d
docker compose logs -f
```

### 4.3 访问面板

浏览器打开 `http://100.64.12.1:8088/admin`（任意 Tailscale 网内设备，手机也行），登录框里**粘贴 headscale API key**，进去后就能图形化管理用户、节点、preauthkey、API key 和 ACL。

---

## 踩坑记录（都是真踩过的）

按出现频率排序，排第一的最容易中招：

1. **装完 CA 没重启 tailscaled → `x509 unknown authority`**。Go 进程的 CA 池启动时加载一次；`openssl s_client` 验证却正常，极容易误导。口诀：**装 CA → 重启 tailscaled → up**。
2. **headscale 0.29 `preauthkeys create --user` 要数字 ID**。`users create` 收名字，`preauthkeys create` 收 ID，填名字报 `ParseUint`。用 `users list` 查 ID。
3. **DEB 装完服务已在跑，`enable --now` 不重启**。postinst 已用默认配置起了服务，改完 config 后必须 `systemctl restart headscale`。
4. **权限问题**：`/var/lib/headscale` 里 `derp_server_private.key`/`noise_private.key` 属主要给 headscale 用户；`server.key` 也要 `chgrp headscale && chmod 640`。别用 root 前台跑 `headscale serve`，会把状态文件生成成 root 属主。
5. **国内服务器直连 GitHub 下 DEB 失败**。release 大文件走 `objects.githubusercontent.com` 常被阻断，加 `ghfast.top/` 前缀走镜像。
6. **macOS 拒绝超长有效期证书**（`OtherTrustValidityPeriod`）：Apple 强制 ≤398 天，服务器证书签 397 天，CA 可以长。
7. **iOS 装了描述文件 ≠ 信任**：还要去「证书信任设置」打开完全信任，否则 `tls: bad certificate`，登录反复失败。排查手机接入先看控制面日志里有没有这行。
8. **不要开 `override_local_dns: true`**：它会把全网公网解析都甩给配置的 nameservers（如 1.1.1.1），部分网络环境下极慢甚至超时，表现为浏览器卡顿。保持 `false`，让 MagicDNS 只接管 `.internal`。
9. **Android 装 CA 要选「CA 证书」类型，且 DERP 对自签证书有已知 bug**：Android 没有 iOS 的完全信任开关，信任取决于证书类型——「CA 证书」进信任锚、「VPN 和应用」证书不进，装错类型控制面日志照刷 `tls: bad certificate`；即便装对，DERP 中继仍可能不信任自签 CA（`no-derp-connection`），同局域网直连不受影响，跨网时建议换公网可信证书。

## 安全建议

- `ca.key` 只留在 relay-1（root 600），分发出去的只有 `ca.crt`。
- 服务器证书有效期 ≤397 天（Apple 限制），到期前重签 + `systemctl restart headscale`。
- ACL 用 **deny-by-default**，只放行需要的端口。
- relay-1 安全组只开 443/tcp + 3478/udp，其余端口对外全关。
- API key 定期轮换（`apikeys create` 新发、`apikeys expire` 作废旧）。

## 小结

整套方案的核心就一句话：**用 headscale 把 Tailscale 的控制面收回自己手里，再用「公网 IP + 自签 CA」干掉域名/备案这个前置门槛**。DERP 交给嵌入式实现复用同一张证书，MagicDNS 用内部命名空间，最后配一个 headplane 面板让日常管理可视化。

成本只有一台最小配置的公网 VPS。换来的是：跨网段设备点对点直连、MagicDNS 内部域名、deny-by-default 的 ACL、以及完全不受官方控制服务器约束的自由度。希望这篇记录对自建组网的你有帮助。