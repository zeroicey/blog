+++
title = 'Lightweight server monitoring: Komari + Tailscale for CPU / memory / disk across your machines'
date = '2026-09-06T21:00:00+08:00'
draft = false
tags = ['Komari', 'Tailscale', 'Headscale', 'Monitoring', 'Self-Hosted', 'Homelab', 'Tutorial']
description = 'Uptime Kuma only tells you up/down, Cockpit only does one Linux box. Komari’s server + agent design puts CPU / memory / disk / network of every machine on a single dashboard with per-second history charts — one agent binary for Linux, macOS and Windows.'
[cover]
  image = 'https://s3.blog.zeroicey.me/covers/komari-server-resource-monitoring.jpg'
+++

The previous posts finished the "connect" and "use" milestone: the first one pulled all the machines — home and cloud — into a single Tailscale network with MagicDNS; the second made a headless server feel like you're sitting at the keyboard; the third gave every service a portless HTTPS domain via Caddy.

But after "reachable" and "usable" there's an even more basic question still unanswered — **how are these machines actually doing?** Is some container quietly pegging the CPU on the control box? Is the MacBook running out of memory again? How much space is left on the Orange Pi's SD card? Which device's load has been climbing over the last hour?

I can reach the services, but the health of the machines themselves — I was still flying blind.

## Why nothing fit before

I'd tried a few tools to "see how much each machine is using", and none felt right:

| Tool | What it does | What's missing |
| --- | --- | --- |
| Uptime Kuma | ping, port checks, HTTP status, downtime alerts | It only answers "up or down" — **no CPU / memory / disk** data at all |
| Cockpit | disk, memory, services, temperature for one Linux box | One machine at a time, Linux only |
| glances / btop | per-machine realtime in the terminal | No unified dashboard; you SSH into each box one by one |

I really wanted three things: **one dashboard, all the machines, with CPU / memory / disk / network history.** After some digging I landed on [Komari](https://github.com/komari-monitor/komari) — a lightweight self-hosted monitor, built as a "server + agent" split, which fits my Tailscale setup perfectly.

## What Komari is

One line: **a server (single container) on your control host, a small agent (one Go binary) on every box you monitor; the agent pushes CPU / memory / disk / network / load / connections once a second, and the server turns it into a dashboard with charts.**

Here's the official main dashboard — all machines' live usage at a glance:

![Komari main dashboard](https://s3.blog.zeroicey.me/posts/komari-server-resource-monitoring/home-dashboard.jpg)

Click into any machine to see the per-second history:

![Komari history charts](https://s3.blog.zeroicey.me/posts/komari-server-resource-monitoring/history-charts.jpg)

The architecture is tidy:

```text
  each box (agent, a single always-on binary)
     │   pushes over WebSocket + HTTP (toward the server)
     ▼
  ┌─────────────────────────────┐
  │  core · 100.64.12.1          │
  │  komari server (one container)│
  │  ├─ embedded Web UI          │
  │  └─ SQLite (main + metrics)  │
  └──────────────┬──────────────┘
                 │  Caddy reverse proxy (HTTPS, LAN-only)
                 ▼
            your browser
```

> ⚠️ **Stand-in values**: like the previous posts, the hostname (`core`), internal domain (`homemesh.internal`) and Tailscale IP (`100.64.12.x`) are fictional examples — swap in your own real values.

## Deploying the server: three steps

The server runs with Docker Compose; the official port is 25774 and data lives in a local volume:

```yaml
# compose.yml
services:
  komari:
    image: ghcr.io/komari-monitor/komari:1.4.3
    container_name: komari
    restart: unless-stopped
    ports:
      - "127.0.0.1:25774:25774"   # local only, Caddy fronts it
    volumes:
      - ./data:/app/data
```

```bash
docker compose up -d
```

On first boot it stops at a setup wizard — open it in a browser (or hit the API) to fill in the admin account, site name and the metrics DB DSN (embedded SQLite is fine):

```bash
# Browser: open http://core:25774
# Or via API:
curl -X POST http://127.0.0.1:25774/api/install/complete \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"<strong-password>","sitename":"homelab","metric_dsn":"file:/app/data/metrics.db?mode=rwc&_txlock=immediate"}'
```

> `metric_dsn` is the connection string for the *metrics* store; this SQLite path is enough for personal use. The password must contain uppercase, lowercase and digits.

Then hide the port behind the Caddy entry point from before to get a portless domain:

```caddy
https://komari.homemesh.internal {
    bind 100.64.12.1
    tls /certs/server.crt /certs/server.key
    reverse_proxy 127.0.0.1:25774
}
```

Add one MagicDNS `extra_records` entry (`komari.homemesh.internal` → `100.64.12.1`) and the panel is reachable at `https://komari.homemesh.internal`. The server is ready — but the dashboard is still empty, because **the data has to be pushed in by agents**.

## Installing agents: one binary for every OS

Grab the agent binary from [komari-agent releases](https://github.com/komari-monitor/komari-agent/releases) as `komari-agent-<os>-<arch>`. First create a device on the panel's "nodes" page to get its token (the token = that device's identity), then run the same command on each machine:

```bash
# Linux (as a systemd service)
komari-agent \
  -e https://komari.homemesh.internal \
  -t <this device's token> \
  --disable-web-ssh --disable-auto-update
```

How I installed it on each platform:

- **Linux**: put the binary in `/usr/local/bin/komari-agent` and wrap it in a systemd unit, then `enable --now`.
- **macOS**: the same binary, wrapped in a LaunchAgent plist — runs as your user, no sudo needed.
- **Windows**: a scheduled task (`schtasks`) that runs at startup as SYSTEM.

> Why a scheduled task and not `sc` for Windows? The agent is a console program that doesn't answer the Windows SCM handshake, so a plain `sc create` fails to start; a scheduled task is the simplest reliable route.

Once all three boxes run it, refresh the panel — the devices pop up and start reporting every second.

## Two pitfalls I hit

**① The "region / flag" badge looks wrong — usually a proxy or the GFW.** The panel labels each device's region by GeoIP, based on the public egress IP the agent reports. And the agent finds its public IP by calling several public sites (`visa.cn`, `toutiao.com`, `ip.sb`, …) and reading its own IP from the reply. That's where it breaks — **if the box uses a proxy / VPN, or those sites are unreachable, the lookup fails or returns the proxy's IP**, so the region shows up empty or as the proxy's country. It's purely cosmetic and doesn't affect monitoring data; to make it right, add `--custom-ipv4` with a real egress IP.

**② Agent and server on the same host — don't let the agent dial its own public domain.** The agent talks to the server over WebSocket. Reverse proxies like Caddy pass the `Upgrade` header through natively, so cross-host via the domain is fine; but when an agent **on the same box** connects to `https://its-own-domain`, the traffic loops back through the tailscale0 interface — in my case that made the WebSocket stream die right after upgrade and the client see `200` instead of `101`. Easy fix: same-host agents use `-e http://127.0.0.1:25774`, cross-host agents use the domain.

## A couple of security suggestions

Komari ships a web terminal / remote-exec capability. If you only want it as a dashboard, put these two flags on every agent:

- `--disable-web-ssh`: **turns off the remote terminal and remote execution**, so the panel becomes read-only monitoring — nobody with panel access can control your machine;
- `--disable-auto-update`: you decide when to upgrade, instead of the agent updating itself at 3am.

Combined with "server bound to LAN only + Caddy HTTPS in front", both the monitoring data and the machine control surface stay inside your Tailscale network.

## Wrap-up

With Komari installed, the machines at home finally share one "instrument panel": the control box, the MacBook, the gaming PC, the Orange Pi, the cloud VPS — every one's CPU / memory / disk / network at a glance, plus history charts to look back. Who's secretly eating resources, whose disk is about to fill — it's obvious now.

String together with the first three posts, the whole home stack reads: **Tailscale to connect → Caddy to use → Sunshine to control → Komari to observe.**

Related links:

- [Komari main repo](https://github.com/komari-monitor/komari) (server + Web UI)
- [komari-agent](https://github.com/komari-monitor/komari-agent) (agent for every OS)
- [Official docs](https://www.komari.wiki/)