+++
title = 'Tailscale 组网进阶：Caddy + MagicDNS 让内网服务免端口、免记 IP'
date = '2026-09-06T12:30:00+08:00'
draft = false
tags = ['Tailscale', 'Headscale', 'Caddy', '反向代理', 'MagicDNS', '自托管', '教程', '内网穿透']
description = 'MagicDNS 只能把域名解析到主机，访问服务还得在域名后面加端口。用 Caddy 反向代理 + 复用组网 CA 签一张通配符证书 + 两条 extra_records，给每个服务一个免端口的 HTTPS 好记域名，全程绿锁、零客户端配置。'

[cover]
  image = 'https://s3.blog.zeroicey.me/covers/tailscale-caddy-reverse-proxy-v2.jpg'
+++

前两篇把「连得上」这件事做完了：第一篇把家里和云端的机器拉进同一个 Tailscale 虚拟局域网，还配好了 MagicDNS——现在输入 `core.homemesh.internal` 就能指到主服务器；第二篇让这台没接屏的主服务器像坐在机箱前一样被远程操作。

但「能连上」之后，日常使用里还藏着一个高频的小烦恼：**MagicDNS 只到主机这一层，访问服务还是得带端口**。

以我家那台 `core` 为例，上面用 Docker 跑着十几个服务。它们各自监听不同的端口，于是访问它们就变成了这样：

```text
core.homemesh.internal:5000    ← 网盘（dufs）
core.homemesh.internal:4533    ← 音乐（Navidrome）
core.homemesh.internal:5678    ← 自动化（n8n）
core.homemesh.internal:8090    ← 下载（qBittorrent）
core.homemesh.internal:8083    ← 书库（Calibre-Web）
```

端口零散、没有规律、一个都记不住。手机浏览器上输入 `https://core.homemesh.internal:8083` 这种长串，简直是折磨。

本篇解决的就是它：**用 Caddy 反向代理 + 一张通配符证书 + 几条 MagicDNS 自定义记录，给每个服务一个「服务名.主机名.内部域」的好记域名**——浏览器只输入域名、不带端口，还全程 HTTPS 绿锁。

```text
dufs.core.homemesh.internal        https:// 直达，不用 :5000
navidrome.core.homemesh.internal   https:// 直达，不用 :4533
```

> ⚠️ **替换说明**：与前两篇一致，文中所有 IP 用 `100.64.12.x` 示例段，主机名（`core` / `relay-1`）、内部域（`homemesh.internal`）均为虚构。照着做时换成你自己的真实值。

## 先看全局：域名是怎么「免端口」的

这里有个很容易想岔的地方：**反代不是把端口藏起来了，而是让服务「看起来」都在 443（HTTPS 默认端口）后面**。浏览器访问任何 `https://xxx`，网络层走的都是 443，真正决定「打到哪个服务」的是 TLS 握手里的 SNI / Host 头——Caddy 就是靠它来分流的。所以整件事要三块一起凑：

```
        macbook / phone / pad（客户端）
             │  https://dufs.core.homemesh.internal
             │  （不输端口，浏览器直接绿锁）
             ▼
   ┌─────────────────────────────────┐
   │  core · 100.64.12.1             │
   │                                 │
   │  Caddy  :443（https）· :80（跳转）│ ← ① 按 SNI/Host 分流
   │   ┌─────┐  ┌─────┐  ┌─────┐    │
   │   │dufs │  │navi │  │ n8n │    │ ← ② 转发到本地端口
   │   └─┬───┘  └─┬───┘  └─┬───┘    │
   │    :5000    :4533    :5678      │
   └─────────────────────────────────┘
             ▲
   headscale `dns.extra_records`     ← ③ MagicDNS 把 *.core.homemesh.internal 指到 core
```

三块的职责：

| 组件 | 做什么 |
| --- | --- |
| 证书 | `*.core.homemesh.internal` 通配符，**复用第一篇那把组网 CA** 签发 |
| DNS | headscale `extra_records` 自定义静态记录，把每个服务域名指到 `core` 的 Tailscale 地址 |
| 反代 | Caddy 监听 `core` 的 443，按主机名把请求转发到本机对应端口 |

其中最妙的是证书那一环——因为它是让「绿锁」**零客户端成本**的关键，下面每个部分展开讲。

## Step 1：签一张通配符证书（复用组网 CA，零配置绿锁的关键）

先想清楚一件事：为什么这里能「直接绿锁、不用再给任何设备装证书」？

因为**第一篇里，所有设备都已经把 `ca.crt` 装进系统信任库了**（因为要信任自签控制面）。也就是说，那把组网 CA（`HomeMesh Root CA`）现在已经是每台设备都信任的**信任锚**。那么只要新证书也是**这同一把 CA 签发的**，它的证书链就能直通这个锚点——Mac、Windows、iPhone、Android 全都自动信任，什么都不用再动。

反过来，如果这里图省事用 Caddy 自带的 internal CA 另签一张，证书链就不在设备信任锚里了，结果就是你得**再去每台设备装一遍新 CA**，iOS 还要重走一遍「完全信任」开关。吃力不讨好。

所以正经做法只有一条：**把 relay-1 上的 `ca.key` 和 `ca.crt` 拿到 core 上，用它们签一张通配符证书**。

```bash
# ① 从 relay-1 取 CA（ca.key 敏感，scp/ssh 都行，别落到公共目录）
mkdir -p /etc/caddy/ca
scp relay-1:/etc/headscale/ca/ca.key /etc/caddy/ca/
scp relay-1:/etc/headscale/ca/ca.crt /etc/caddy/ca/
chmod 600 /etc/caddy/ca/ca.key

# ② 生成本机服务器私钥 + CSR
openssl genrsa -out /etc/caddy/server.key 2048
openssl req -new -key /etc/caddy/server.key -out /tmp/srv.csr \
  -subj "/C=CN/O=HomeMesh/CN=dufs.core.homemesh.internal"

# ③ 扩展文件：SAN 写通配符，这样一张证书覆盖所有服务域名
cat > /tmp/srv.cnf <<'EOF'
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = DNS:*.core.homemesh.internal
EOF

# ④ 用组网 CA 签发，有效期 397 天（Apple 限制，见下方坑）
openssl x509 -req -in /tmp/srv.csr -CA /etc/caddy/ca/ca.crt -CAkey /etc/caddy/ca/ca.key \
  -CAcreateserial -out /etc/caddy/server.crt -days 397 -sha256 -extfile /tmp/srv.cnf

# ⑤ 校验：证书链必须通到 HomeMesh Root CA
openssl verify -CAfile /etc/caddy/ca/ca.crt /etc/caddy/server.crt
# server.crt: OK
```

几个要点：

- **SAN 一定是 `*.core.homemesh.internal` 通配符**，这样以后加服务不用再签证书，一张吃遍 `dufs.`、`navidrome.`、`n8n.`…… 所有前缀。
- `CN` 随便填一个就行，现代浏览器认 SAN 不认 CN；写第一个服务域名占位即可。
- 有效期 **397 天**，别贪心签 10 年——Apple 设备强制服务器证书 ≤ 398 天，超了会报 `OtherTrustValidityPeriod`（第一篇踩过同一个坑）。

安全上要诚实交代一句：这做的代价是 **`ca.key` 多了一个落点（从 relay-1 到了 core）**。对家庭自用场景，这个折中换来的是「全网设备零改动的绿锁」；如果你对隔离有洁癖，可以不用组网 CA、改让 Caddy 自带 internal CA 签，代价就是回退到「每台设备装一次新 CA」。怎么取舍看你自己，我在末尾「安全建议」里再展开。

## Step 2：MagicDNS 加自定义记录（让域名解析到 core）

MagicDNS 默认只把主机的名字解析出来——`core.homemesh.internal` 指向 `100.64.12.1`，但它不认识 `dufs.core.homemesh.internal`。要让它认识，就得用 headscale 的 **`extra_records`**（静态自定义记录）。

在 relay-1 的 `/etc/headscale/config.yaml` 里：

```yaml
dns:
  magic_dns: true
  base_domain: homemesh.internal
  override_local_dns: false
  extra_records:
    - name: "dufs.core.homemesh.internal"
      type: "A"
      value: "100.64.12.1"
    - name: "navidrome.core.homemesh.internal"
      type: "A"
      value: "100.64.12.1"
    # 其余服务照此追加……
```

改完重启让配置生效，然后在任意节点验证：

```bash
systemctl restart headscale

# 任意节点上
getent hosts dufs.core.homemesh.internal
# 100.64.12.1   dufs.core.homemesh.internal
```

> 目前 `extra_records` 支持 A / AAAA 记录，够用了。看 `base_domain` 那条：它只是 tailnet 内部命名空间，解析由内置 resolver（`100.100.100.100`）应答，**完全不经过公网 DNS、不涉及备案**——和第一篇的免域名方案是同一套逻辑。

## Step 3：部署 Caddy，按主机名分流

Caddy 我跑在 `core` 的 Docker 里，用 **host 网络**——目的是让它直接绑 Tailscale 地址、并能直接访问本机 `127.0.0.1` 上的其它服务，少一层网络穿透的不确定。

`compose.yaml`：

```yaml
services:
  caddy:
    image: caddy:2-alpine
    container_name: caddy
    restart: unless-stopped
    network_mode: host            # 直接绑宿主网络，反代 127.0.0.1 直达本地服务
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - /etc/caddy:/certs:ro      # server.crt / server.key（挂只读）
    healthcheck:
      test: ["CMD", "wget", "-q", "-O", "/dev/null", "http://127.0.0.1:2019/config/"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
```

`Caddyfile`（关键在 `bind 100.64.12.1` + 显式证书）：

```caddy
{
    auto_https off                 # 关闭自动签发，用我们自己的证书
}

https://dufs.core.homemesh.internal {
    bind 100.64.12.1               # 只绑 Tailscale 地址
    tls /certs/server.crt /certs/server.key
    reverse_proxy 127.0.0.1:5000
}

https://navidrome.core.homemesh.internal {
    bind 100.64.12.1
    tls /certs/server.crt /certs/server.key
    reverse_proxy 127.0.0.1:4533
}

# ……其余服务照此追加，每个服务一个 block……

# 统一：http(80) 301 跳 https，浏览器直输域名不带协议也对
http://*.core.homemesh.internal {
    bind 100.64.12.1
    redir https://{host}{uri} permanent
}
```

启动后验证：

```bash
docker compose up -d
curl -s --resolve dufs.core.homemesh.internal:443:100.64.12.1 \
     --cacert /etc/caddy/ca/ca.crt \
     -o /dev/null -w '%{http_code}\n' https://dufs.core.homemesh.internal/
# 401（dufs 匿名未登录，正常；能拿到 HTTP 响应说明反代链路已通）
```

- `auto_https off` 让 Caddy 别去自动申请证书，因为我们要用显式的 `tls` 指令挂自己的证书。
- **`bind 100.64.12.1` 是最容易漏、也最该写的一行**：host 网络下不加 bind，Caddy 会监听 `0.0.0.0`——等于把这些服务域名暴露到局域网甚至公网口。加上它，443/80 只在 Tailscale 网内。
- 末尾那个 `http://*.core.homemesh.internal` 是通配块，处理所有「浏览器直输域名、默认走 http 80」的情况，统一 301 到 https。

## Step 4：把「全部服务」一口气铺开

按上面的套路一行行加就行。加一个服务 = 三条：`extra_records` 里一条 A 记录 + `Caddyfile` 里一个 block +（可选）ACL 里若原本没放行 443 就补上。我这边实际铺了这么一张表：

| 服务 | 域名 | 后端端口 |
| --- | --- | --- |
| dufs 网盘 | `dufs.core.homemesh.internal` | 5000 |
| Navidrome 音乐 | `navidrome.core.homemesh.internal` | 4533 |
| qBittorrent 下载 | `qbittorrent.core.homemesh.internal` | 8090 |
| n8n 自动化 | `n8n.core.homemesh.internal` | 5678 |
| Calibre-Web 书库 | `calibre-web.core.homemesh.internal` | 8083 |
| MinIO Console | `minio.core.homemesh.internal` | 9001 |
| Pocket-ID 认证 | `pocket-id.core.homemesh.internal` | 1411 |

验证也很省事：对每个域名 curl 一下，看是不是拿到了该服务自己的响应（首页 200 / 跳登录 302 / 未认证 401 都算「链路通了」，因为反代已经把它送到了正确的后端）。

顺带一提架构上的一个小红利：因为所有流量都走 443 反代了，**deny-by-default 的 ACL 里「客户端 → core」其实只需要放行一个 443**，那些五花八门的服务端口（5000、4533、5678……）不必再逐个暴露——最小权限更干净了。（是否收窄旧白名单看你习惯，两者能并存。）

## 踩坑记录（都是真踩过的）

1. **别另起炉灶签新 CA**。第一反应是用 Caddy 自带的 internal CA，结果证书链不在设备信任锚里，等于让所有设备再装一遍 CA——上一篇好不容易装完的 CA 白瞎。**一定要复用组网那把 CA**。
2. **host 网络 + `bind` 是配套的**。忘记 `bind 100.64.12.1` 时 Caddy 监听 `0.0.0.0`，把内部服务域名暴露到局域网/公网。每个 block 都要写 bind。
3. **`caddy reload` 可能不吃新配置**。改了 Caddyfile 后 `caddy reload --config` 有时没真正生效（日志看到的还是旧后端地址）。省心做法是直接 `docker compose restart caddy`，秒级即可，别在 reload 上较劲。
4. **有些服务只绑了 Tailscale 地址、没绑 `127.0.0.1`**。反代后端地址写错了（比如写了 `127.0.0.1:1411`，但该服务只监听 `100.64.12.1:1411`）就会 502。**先 `ss -tlnp` 或 `docker ps` 看服务到底绑的哪个地址，反代目标跟它一致**。
5. **证书有效期 ≤ 397 天**。Apple 强制 398 天上限，签 10 年会只有 Linux 正常、macOS/iOS 全被拒（`OtherTrustValidityPeriod`）。到期前重签 + 重启 Caddy 即可。
6. **别用 `curl -H 'Host: ...'` 测证书**。`-H Host` 不会设置 SNI，Caddy 拿不到匹配证书，会回 `TLS alert internal error`，让你误以为证书坏了。要用 `--resolve 域名:443:IP`，或直接浏览器访问真实域名。

## 安全建议

- **`ca.key` 的落点要想清楚**。复用组网 CA 换取绿锁的同时，`ca.key` 会多一台机器持有。家庭自用可接受；若真要隔离，给 Caddy 单独造一个 intermediate CA，把它交叉签名到组网根 CA——设备照样信任，而 intermediate CA 的私钥可以单独轮换/吊销，不必暴露根私钥。
- **反代只绑 Tailscale 地址**，不 `0.0.0.0`、不开端口转发，服务对公网/局域网保持不可见。
- ACL 维持 deny-by-default，客户端→core 只放 443 一个口；后端那些端口不出现在任何白名单里。
- 证书记个续期提醒（397 天一签），过期症状是浏览器突然告警，重签 + 重启 Caddy 就好。

## 小结

一句话总结本篇：**Tailscale 负责把「网络」打通，Caddy 负责把「服务」变好记**。MagicDNS 给每台主机一个名字，`extra_records` 再给每个服务补一个名字，通配符证书复用组网信任锚解决绿锁，Caddy 在 443 上按主机名把它们一一送到正确的端口。

结果就是：内网服务的访问成本从「记一串 IP 和端口」降到「记住服务名」，手机和电脑浏览器输入 `dufs.core.homemesh.internal` 直接开页面，干净、快、还是绿的。三个组件都是开源、零硬件成本，纯粹是把前面几篇搭好的地基往上再盖了一层。

如果你也有一堆自托管服务在啃端口号的苦，这套「Caddy + MagicDNS + 组网 CA」能一并解决。希望这篇记录对你有帮助。