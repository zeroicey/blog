+++
title = 'Homepage: A YAML-Driven Navigation Dashboard for Your Homelab'
date = '2026-09-06T22:50:00+08:00'
draft = false
tags = ['Homepage', 'Dashboard', 'Self-hosted', 'Docker', 'Tailscale', 'Tutorial']
description = 'When your services outgrow the browser bookmark bar, reach for Homepage (gethomepage). A declarative YAML navigation dashboard with native Docker integration that shows uptime, CPU and memory for every service in one glance — plus a local-icon and aurora-background glow-up.'
[cover]
  image = 'https://s3.blog.zeroicey.me/covers/homepage-dashboard.jpg'
+++

In the earlier posts of this series I finished the "hardware" side of the homelab: the machines are pulled into one Tailscale virtual LAN, the headless box is remotely operable, every service has a port-free HTTPS domain, and a monitoring panel keeps an eye on resource usage.

But a new problem surfaced — **the more services I run, the harder it is to remember where they live.** File storage, monitoring, notes, music, e-books, an LLM gateway, automation… they're scattered across a pile of `*.internal` domains, and every visit means digging through bookmarks and half-remembering ports. What I wanted was a real "start page": open the browser, see every service at a glance, click and go.

This post recommends exactly the tool that does that — [Homepage](https://github.com/gethomepage/homepage).

## Why Homepage

Plenty of self-hosted "start page" projects exist, and I tried most of them:

| Tool | What it is | Why I passed |
| --- | --- | --- |
| Homer | Minimal static navigation, YAML config | Just an icon wall — no service or resource status |
| Dashy | Highly customizable dashboard | Very capable, but the config surface is a Swiss army knife |
| Heimdall | Veteran web UI, GUI config | No YAML to write, but the UI and ecosystem feel dated |
| Homarr | Plex / \*arr media-stack integration | Not my stack |
| **Homepage** | **Declarative YAML + Docker integration + service widgets** | ✅ The pick |

Homepage won on three counts:

1. **Declarative YAML.** The entire dashboard is a handful of `yaml` files — Git-versioned, diffable, scriptable. It matches how I already do ops.
2. **Native Docker integration.** Mount `docker.sock` and every card shows its container's **uptime plus CPU / memory**, expandable to network and traffic. It's not just a launcher — it's a living status wall.
3. **Looks good out of the box.** Themes, backgrounds, glassmorphism, and dozens of service widgets (weather, calendar, system resources, third-party statuses…) add up to a genuinely modern dashboard.

## Deploy: one compose file

Homepage is a Next.js app. The official image is `ghcr.io/gethomepage/homepage`, port 3000 inside the container, config mounted in:

```yaml
# compose.yml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:3000"          # local only; let Caddy reverse-proxy it
    volumes:
      - ./config:/app/config           # yaml config directory
      - /var/run/docker.sock:/var/run/docker.sock:ro   # docker integration (read-only)
```

```bash
docker compose up -d
```

> The `docker.sock` mount is optional: without it the panel is just static navigation; with it (always `ro`) cards get container status and resource stats.

Then give it a port-free domain with the Caddy + MagicDNS setup from earlier in the series:

```caddy
https://home.homemesh.internal {
    bind 100.64.12.1
    tls /certs/server.crt /certs/server.key
    reverse_proxy 127.0.0.1:3000
}
```

Open `https://home.homemesh.internal` and you'll see an empty panel — the rest is all YAML.

## Config: everything is YAML

Three files do the heavy lifting:

| File | Purpose |
| --- | --- |
| `services.yaml` | groups + navigation entries (href, icon, description, Docker integration) |
| `settings.yaml` | title, theme, background, layout, language |
| `widgets.yaml` | header widgets (search, clock, weather, system resources…) |

A `services.yaml` looks like this:

```yaml
- System & Ops:
    - Dashboard:
        icon: /icons/homepage.png
        href: https://home.homemesh.internal
        description: The start page
        server: local
        container: homepage
    - Monitoring:
        icon: /icons/komari.png
        href: https://komari.homemesh.internal
        description: CPU / memory / disk
        server: local
        container: komari

- Data & Storage:
    - Files:
        icon: /icons/dufs.png
        href: https://dufs.homemesh.internal
        server: local
        container: dufs
```

See the `server` + `container` pair? That's the Docker integration — `server` names the Docker instance from `docker.yaml` (a local socket works fine), and `container` names the container. Once set, a red/green status dot appears on the card, expandable to CPU / memory / network. Group your dozen-odd web services under AI / Data / Media / Life / Ops and the dashboard comes together.

`settings.yaml` controls the look:

```yaml
title: Homelab
description: Personal dashboard
theme: dark
background: /images/bg.jpg     # custom background
cardBlur: sm                   # frosted-glass cards
language: zh-Hans
```

## Making it pretty: background + self-hosted official icons

The default Homepage is a bit plain. Two small moves lift it a lot.

**① Add a background and glassmorphism.** I generated a 2560×1440 dark aurora gradient with ImageMagick (soft indigo / teal / violet glows, heavily blurred), mounted it, set `background: /images/bg.jpg`, added `cardBlur: sm` for frosted cards, and switched the header to `clean`. The result is this post's cover.

**② Use official logos — and host them locally.** This is the most overlooked, most visible detail: by default Homepage fetches icons from **remote CDNs** in the browser (`si-*` / `mdi-*` / bare names). If your device can't reach that CDN (common in some networks), every icon degrades into an ugly letter avatar. The fix is to download and serve them yourself:

```bash
# Official app logos from the dashboard-icons repo, into ./icons/
curl -o ./icons/<name>.png \
  "https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/png/<name>.png"

# Custom / niche services: convert a Material Design icon to a white PNG
convert -background none -density 1200 /tmp/<name>.svg -resize 128x128 \
  -channel RGB -negate +channel png32:./icons/<name>.png
```

Add two read-only mounts to compose:

```yaml
    volumes:
      - ./icons:/app/public/icons:ro
      - ./images:/app/public/images:ro
```

Then point every `icon` in `services.yaml` at `/icons/<name>.png` — icons now render reliably, no CDN mood swings involved.

## Two gotchas from deployment

**① "Host validation failed" (400).** Newer Homepage validates the HTTP `Host` header and answers 400 for anything not whitelisted — bare IPs and fresh domains trip this all the time. Add an env var listing every allowed `host:port`:

```yaml
    environment:
      - HOMEPAGE_ALLOWED_HOSTS=127.0.0.1:3000,100.64.12.1:3000,home.homemesh.internal
```

> Every time you add a domain / port, remember to append it to the whitelist and restart — otherwise another 400.

**② Don't healthcheck with busybox `wget`.** A check written as `wget -q -O /dev/null http://127.0.0.1:3000/` gives false positives — the image's busybox `wget` returns exit code 0 even on 4xx / 5xx, so a page serving 400 still reads healthy. Use Node to strictly check `r.ok` instead:

```yaml
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:3000/').then(r=>{if(!r.ok)process.exit(1)}).catch(()=>process.exit(1))"]
```

## A word on security

Homepage has no auth — it's just an entry page plus a status wall. Two things are non-negotiable:

1. **Bind local + reverse-proxy.** Like every other service in the series, bind the container port to `127.0.0.1` only and expose it through `https://home.homemesh.internal` (HTTPS, LAN-only), never to the public internet.
2. **Mount `docker.sock` read-only.** Reading container status is all you need; don't hand the panel write access to the Docker API.

## Wrapping up

Set Homepage as the browser's start page and the homelab finally gets its "front end": open the tab and there's a wall of cards — services grouped neatly, status dots green and red at a glance, one click to get anywhere. Together with the earlier posts, the whole chain reads:

**Tailscale to connect → Caddy to name → Sunshine to control → Komari to watch → Homepage to find.**

If your self-hosted list is still short, start with Homepage now; the day it grows into dozens of entries, you'll be glad you picked a navigation panel you can maintain in YAML.

Related links:

- [gethomepage/homepage](https://github.com/gethomepage/homepage) (main repo)
- [Official docs](https://gethomepage.dev/) (authoritative config reference)
- [homarr-labs/dashboard-icons](https://github.com/homarr-labs/dashboard-icons) (self-hosted app icon library)