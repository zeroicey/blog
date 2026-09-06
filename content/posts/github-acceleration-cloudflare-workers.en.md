+++
title = 'Accelerate GitHub Without a VPN: Self-Hosting a gh-proxy Mirror on Cloudflare Workers'
date = '2026-09-06T21:00:00+08:00'
draft = false
tags = ['GitHub', 'Cloudflare', 'Workers', 'CDN', 'Proxy', 'Self-Hosted', 'Tutorial', 'Userscript']
description = 'GitHub is slow and flaky behind the Great Firewall, and airport proxy nodes are unstable. Deploy the single-file open-source gh-proxy on Cloudflare Workers, bind it to your own domain, and pull releases, raw files, archives and git clones without a VPN — then make your proxy core dial it directly and add a userscript for the browser.'

[cover]
  image = 'https://s3.blog.zeroicey.me/covers/github-acceleration-cloudflare-workers.jpg'
+++

Using GitHub from mainland China comes with two problems almost nobody escapes: **pages that won't load, and downloads that crawl**. `git clone` sits on `Receiving objects` for half an hour; a release binary dies halfway through; `raw.githubusercontent.com` fails more often than it works. So you either fire up a VPN or lean on an airport proxy node.

But airport nodes are their own headache. I've been through round after round of this: nodes that flap between alive and dead by the minute, and health checks that report `alive` while real traffic times out the moment you send it. Using one to fetch from GitHub feels like "it worked this time, maybe not next, pure luck".

There's a cleaner way: **everything you download from GitHub is a static resource, so a reverse proxy sitting on Cloudflare's edge can accelerate it.** The open-source project `gh-proxy` packages exactly this into a *single file, serverless* Worker. Drop it into Cloudflare Workers, bind your own domain, and from then on you prefix any GitHub URL to pull it without a VPN:

```text
# before: direct, slow / flaky / blocked
git clone https://github.com/owner/repo.git
curl -L https://github.com/owner/repo/releases/download/v1.0/app.zip

# after: through your own accelerator
git clone https://gh.example.com/https://github.com/owner/repo.git
curl -L https://gh.example.com/https://github.com/owner/repo/releases/download/v1.0/app.zip
```

Coverage is broad: **release assets, source archives, blob/raw files, gists, and `git clone`**. This post documents the full setup, including two gotchas you'll never find without burning time on them (both live in the "make the proxy core dial it directly" section).

> ⚠️ **Placeholders**: `gh.example.com` below is a stand-in — replace it with your own domain hosted on Cloudflare (for example, a `gh` subdomain of your main domain). The proxy-core section uses mihomo (Clash Meta) with its default config path `/etc/mihomo/config.yaml`; the same ideas apply to any Clash-derived core.

## How it works: why one prefix is enough

`gh-proxy` is essentially a domain rewriter. When a request hits `gh.example.com`, the Worker strips the path after `gh.example.com/`, re-assembles it onto `https://github.com`, and fetches the content back. So every static GitHub resource — release assets, raw files, repo archives — becomes reachable through a single prefix:

```
you ──► gh.example.com (Cloudflare edge, nearby entry, near-zero loss)
           │  Worker re-assembles https://github.com/...
           ▼
        github.com (US origin, unfiltered Cloudflare egress)
```

What actually accelerates things is **Cloudflare's edge network**: your request enters a nearby Cloudflare node, and the hop from Cloudflare to GitHub's origin rides its backbone, bypassing the congested, unstable international route between your network and GitHub. All you've done is replace "connecting to GitHub" with "connecting to Cloudflare" — which is an order of magnitude more reliable where you are.

## Why a custom domain is mandatory

Here's a trap you'll hit every time: **the default `*.workers.dev` domain Cloudflare gives you is unreachable from mainland China.** So "deploy to Workers and you're done" simply doesn't hold behind the wall — you must own a domain **hosted on Cloudflare** and bind it as the Worker's custom domain. That's also the precise precondition for the whole scheme: as long as the domain and the Worker live in the same Cloudflare account, binding is just one routing entry.

Nowadays "your own domain" has a very low bar: point any domain's NS at Cloudflare (free tier), and you're set. Once bound, requests to `gh.example.com` are routed to your Worker at the CDN layer.

## Step 1: three prerequisites

- A **Cloudflare account** (free) with at least one domain hosted on CF (this post uses `example.com`, adding a `gh` subdomain).
- **wrangler**, Cloudflare's official CLI: `npm install -g wrangler` (or via `pnpm`/`npx`).
- **An API token** to deploy the Worker and bind the domain programmatically. In the Cloudflare dashboard, go to `My Profile → API Tokens → Create Token` and build a custom one:

| Scope  | Resource | Permission |
| --- | --- | --- |
| Account | — | Workers Scripts · Edit |
| Account | — | Workers Routes · Edit |
| Zone | example.com | DNS · Edit |

> Or just pick the built-in **"Edit Cloudflare Workers"** template and add a single `DNS · Edit` entry for `example.com`. The token is shown only once — save it.

## Step 2: grab the single file

The Workers version of `gh-proxy` is **one `index.js`** (a bit over a hundred lines). No need to clone the whole repo — though cloning is fine if you want to read the source:

```bash
# either grab just the file (faster)
curl -O https://raw.githubusercontent.com/hunshcn/gh-proxy/master/index.js

# or clone the repo
git clone https://github.com/hunshcn/gh-proxy
```

For the root-path + custom-domain setup, this file needs **no edits at all**: `PREFIX='/'` is already correct. (You'd only change it if you wanted to serve the proxy under a sub-path like `/gh/`.)

## Step 3: config + deploy

Create `wrangler.toml` next to `index.js`:

```toml
name = "gh-proxy"
main = "index.js"
compatibility_date = "2025-06-01"

# custom domain: requires example.com and the Worker in the same account
# wrangler creates the DNS record and the binding for you
routes = [
  { pattern = "gh.example.com", custom_domain = true }
]
```

Then deploy:

```bash
export CLOUDFLARE_API_TOKEN="<your token>"
wrangler deploy
```

Seeing `gh.example.com (custom domain)` in the output means both the code and the domain are live. That single command does three things: uploads the script, creates the Worker trigger, and binds the custom domain — DNS record included. **No clicking around the Cloudflare dashboard required.**

## Step 4: verify

```bash
# raw file
curl -s -o /dev/null -w "%{http_code}\n" \
  https://gh.example.com/https://raw.githubusercontent.com/hunshcn/gh-proxy/master/index.js
# → 200

# release asset
curl -LO https://gh.example.com/https://github.com/owner/repo/releases/download/v1.0/app.tar.gz

# clone
git clone https://gh.example.com/https://github.com/hunshcn/gh-proxy
```

All three working means the proxy itself is ready. Devices with unfiltered connectivity can already use it VPN-free.

## Step 5: make the proxy core dial this domain directly (the real gotchas live here)

If your environment routes traffic through a Clash / mihomo core, both your browser and terminal go through it first. In that case `gh.example.com` will be treated as an "overseas site" and pushed onto some airport node by default — **going in a full circle right back to that unstable airport, making your own accelerator pointless.**

The right move is to declare `gh.example.com` as **DIRECT** in the core, so it reaches the Cloudflare edge without touching any airport node. It looks like a one-line change, but I hit three walls here, one by one:

**Gotcha 1: `hosts` belongs at the top level, not inside `dns:`.** mihomo's static resolution config is a top-level `hosts:` block (sibling to `rules:` and `dns:`). If, like me, you shove it under `dns:`, it gets **silently ignored** — no error, no warning.

**Gotcha 2: DIRECT can still "fail to resolve".** After adding `DOMAIN,gh.example.com,DIRECT`, the logs still showed:

```text
dial DIRECT (match Domain/gh.example.com) ...
error: dns resolve failed: context deadline exceeded
```

Two things stacked up: the machine has no usable IPv6 route, yet the core's `ipv6: true` sends the AAAA records to dial as well; and since this is a *cold domain*, the core follows its `fallback-filter` (non-mainland IPs must resolve via DoH like `1.1.1.1`/`8.8.8.8`) — and those DoH servers are exactly the ones blocked when dialed directly from the mainland. So resolution times out. (Old domains like `cloudflare.com` happen to work only because they're warm in the cache.)

**Gotcha 3: DNS-related config changes need a `restart`, not a `reload`.** HUP hot-reload does not rebuild the DNS resolver, so your `hosts` edit will look like it "didn't take effect".

**The correct final config** (pin the domain to a reachable IPv4 edge address, sidestepping all of the above):

```yaml
# top level, NOT inside dns:
hosts:
  "gh.example.com": "172.67.1.2"   # the A record your domain actually resolves to (IPv4)

rules:
  - 'DOMAIN,gh.example.com,DIRECT'
```

```bash
# a full restart is required (HUP doesn't rebuild DNS)
sudo systemctl restart mihomo && sleep 20

# verify: 200 + `using DIRECT` in the logs
curl -x http://127.0.0.1:7890 -o /dev/null -w "%{http_code}\n" \
  https://gh.example.com/
sudo journalctl -u mihomo --since "1 min ago" | grep gh.example.com
```

Seeing `match Domain(gh.example.com) using DIRECT` in the logs means you're done: traffic through the proxy core now dials the Cloudflare edge directly, bypassing the airport.

## Step 6: automatic acceleration in the browser (userscript)

Command line sorted — what about the browser? You can't be manually prefixing every GitHub link. The answer is a script manager plus a userscript that rewrites links:

1. **Install a script manager**: [Tampermonkey](https://www.tampermonkey.net/) for Chrome/Edge, or Tampermonkey/Violentmonkey for Firefox.
2. **Install an accelerator script** (pick one):
   - [**GitHub加速下载**](https://greasyfork.org/zh-CN/scripts/504224): built specifically for gh-proxy; set the accelerator address to `https://gh.example.com/` after installing.
   - [**Github 增强 - 高速下载**](https://greasyfork.org/zh-CN/scripts/412245): a more feature-rich, long-standing script. Click the extension icon → custom accelerator, and fill in `https://gh.example.com/` for `Raw`, `Git Clone`, and `Release (Code ZIP)`.

Once installed, raw files, release downloads and clone links on GitHub pages are rewritten to your accelerator automatically — matching the command-line experience.

## Stability, quotas, and boundaries

- **Quota**: Cloudflare Workers free tier gives you 100,000 requests/day and 1,000/min. For one person's use (even frequent clones and release downloads) that's far more than enough; the $5/month paid tier is 10M/month.
- **Stability**: the setup itself is rock-solid — a Worker is stateless, so there's no "crashes after a few days" failure mode. The only variable is Cloudflare's free edge occasionally being slow/jittery from the mainland, which is still far more reliable than direct GitHub and than unstable airport nodes.
- **Boundary (important)**: the author's public demo instance is long overloaded. **Self-host it, and keep it to yourself.** Don't paste your accelerator domain anywhere public — free quota doesn't survive abuse; expose it and someone will throttle it for you.
- **Security**: `gh-proxy` only proxies GitHub-related hosts (release/archive/raw/blob/gist/info/git-*), not a general-purpose proxy. It can't be used to reach arbitrary sites, so the abuse surface is naturally tiny.

## Wrapping up

The value here isn't "yet another proxy" — it's turning a high-frequency pain point into **zero-maintenance infrastructure**: deploy once, works forever, VPN-free on both the CLI and the browser. Combined with the Tailscale mesh from the earlier posts, my home machines, cloud servers, and day-to-day development all fold into a single self-hosted system. That was the goal all along: **wherever I can own a piece of the network path, I do.**

If you're tired of GitHub's crawl, spend ten minutes walking through this post — you'll probably wonder why you didn't do it sooner.