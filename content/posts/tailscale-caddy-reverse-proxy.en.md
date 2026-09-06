+++
title = 'Tailscale Beyond Networking: Caddy + MagicDNS for Portless, Domain-Based Service Access'
date = '2026-09-06T12:30:00+08:00'
draft = false
tags = ['Tailscale', 'Headscale', 'Caddy', 'reverse-proxy', 'MagicDNS', 'self-hosted', 'tutorial']
description = 'MagicDNS resolves hostnames but still forces you to type ports after the domain. Use Caddy + a wildcard cert from your existing mesh CA + headscale extra_records to give every service a memorable, portless HTTPS domain — green lock included, zero client setup.'

[cover]
  image = 'covers/tailscale-caddy-reverse-proxy.jpg'
+++

The first two posts finished the "can I reach it" part. Part one pulled home and cloud machines into one Tailscale mesh and set up MagicDNS, so `core.homemesh.internal` now resolves to the main server. Part two made that headless server usable as if you were sitting in front of it.

But once things are reachable, a small daily annoyance remains: **MagicDNS stops at the host level — accessing services still means typing ports.**

My `core` box runs a dozen services in Docker, each listening on its own port. So reaching them looks like this:

```text
core.homemesh.internal:5000    ← file server (dufs)
core.homemesh.internal:4533    ← music (Navidrome)
core.homemesh.internal:5678    ← automation (n8n)
core.homemesh.internal:8090    ← downloads (qBittorrent)
core.homemesh.internal:8083    ← book library (Calibre-Web)
```

Scattered ports, no pattern, impossible to remember — and typing `https://core.homemesh.internal:8083` on a phone is misery.

This post fixes exactly that: **Caddy reverse proxy + one wildcard certificate + a few MagicDNS custom records gives every service a memorable `service.host.domain` name** — type the domain, skip the port, and stay HTTPS green-locked.

```text
dufs.core.homemesh.internal        straight to https://, no :5000
navidrome.core.homemesh.internal   straight to https://, no :4533
```

> ⚠️ **Placeholder note**: as in the previous posts, all IPs use the `100.64.12.x` example range, and hostnames (`core` / `relay-1`) and the internal domain (`homemesh.internal`) are fictional. Swap in your own values.

## The big picture: how a domain becomes "portless"

Here's the thing people often get backwards: **the reverse proxy doesn't hide the port — it makes every service sit behind 443 (the default HTTPS port)**. Any `https://xxx` request hits port 443 at the network layer; what decides "which service" is the SNI / Host header from the TLS handshake, and that's exactly what Caddy routes on. So the whole thing is three pieces working together:

```
        macbook / phone / pad (clients)
             │  https://dufs.core.homemesh.internal
             │  (no port, green lock in the browser)
             ▼
   ┌─────────────────────────────────┐
   │  core · 100.64.12.1             │
   │                                 │
   │  Caddy  :443 (https) · :80 (redirect)│ ← ① route by SNI/Host
   │   ┌─────┐  ┌─────┐  ┌─────┐    │
   │   │dufs │  │navi │  │ n8n │    │ ← ② forward to local ports
   │   └─┬───┘  └─┬───┘  └─┬───┘    │
   │    :5000    :4533    :5678      │
   └─────────────────────────────────┘
             ▲
   headscale `dns.extra_records`     ← ③ MagicDNS points *.core.homemesh.internal at core
```

The three pieces:

| Component | What it does |
| --- | --- |
| Cert | `*.core.homemesh.internal` wildcard, **signed by the mesh CA you already trust from part one** |
| DNS | headscale `extra_records` static entries pointing each service name at `core`'s Tailscale IP |
| Proxy | Caddy listening on `core`'s 443, forwarding by hostname to the right local port |

The cert piece is the clever one — it's what makes the green lock **zero client cost**, so each part below gets its own section.

## Step 1: Sign a wildcard cert (reusing the mesh CA — the green-lock key)

First, be clear on why this gives an instant green lock with nothing to install anywhere.

**In part one, every device already loaded `ca.crt` into its system trust store** (to trust the self-signed control plane). That means your mesh CA (`HomeMesh Root CA`) is now a **trust anchor on every device**. So if a new cert is signed by *that same CA*, its chain resolves straight to that anchor — macOS, Windows, iPhone, Android all trust it automatically, no action needed.

Go the other way — have Caddy's built-in internal CA sign instead — and the chain is no longer anchored on any device, forcing you to **reinstall a new CA on every device**, re-doing the "full trust" dance on iOS. All pain, no gain.

So the right move is simple: **pull `ca.key` and `ca.crt` from relay-1 to core and sign a wildcard cert with them.**

```bash
# ① get the CA from relay-1 (ca.key is sensitive; don't drop it in a shared dir)
mkdir -p /etc/caddy/ca
scp relay-1:/etc/headscale/ca/ca.key /etc/caddy/ca/
scp relay-1:/etc/headscale/ca/ca.crt /etc/caddy/ca/
chmod 600 /etc/caddy/ca/ca.key

# ② generate a server key + CSR
openssl genrsa -out /etc/caddy/server.key 2048
openssl req -new -key /etc/caddy/server.key -out /tmp/srv.csr \
  -subj "/C=CN/O=HomeMesh/CN=dufs.core.homemesh.internal"

# ③ extension file: wildcard SAN, so one cert covers every service name
cat > /tmp/srv.cnf <<'EOF'
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = DNS:*.core.homemesh.internal
EOF

# ④ sign with the mesh CA, 397 days (Apple limit, see the gotchas)
openssl x509 -req -in /tmp/srv.csr -CA /etc/caddy/ca/ca.crt -CAkey /etc/caddy/ca/ca.key \
  -CAcreateserial -out /etc/caddy/server.crt -days 397 -sha256 -extfile /tmp/srv.cnf

# ⑤ verify the chain reaches HomeMesh Root CA
openssl verify -CAfile /etc/caddy/ca/ca.crt /etc/caddy/server.crt
# server.crt: OK
```

A few points:

- **The SAN must be the wildcard `*.core.homemesh.internal`**, so future services need no new cert — one covers `dufs.`, `navidrome.`, `n8n.` … and everything else.
- The `CN` is basically dead in modern browsers (they trust SAN, not CN); put a placeholder like the first service name.
- Keep it at **397 days** — don't sign 10 years. Apple enforces a ≤398-day cap on server certs and rejects longer ones with `OtherTrustValidityPeriod` (we hit this in part one).

Be honest about the security trade-off here: this means **`ca.key` now lives in one more place (on core, not just relay-1)**. For a home setup, that trade buys a green lock across every device with zero reconfiguration. If you want strict isolation, skip the mesh CA and let Caddy's internal CA sign — at the cost of reinstalling a CA on every device. I revisit this in "Security notes" below.

## Step 2: Add MagicDNS custom records (make the names resolve to core)

MagicDNS only resolves hostnames by default — `core.homemesh.internal` → `100.64.12.1`. It doesn't know `dufs.core.homemesh.internal`. Teach it with headscale's **`extra_records`** (static custom records).

In `/etc/headscale/config.yaml` on relay-1:

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
    # … append the rest of your services the same way
```

Restart to apply, then verify from any node:

```bash
systemctl restart headscale

# on any node
getent hosts dufs.core.homemesh.internal
# 100.64.12.1   dufs.core.homemesh.internal
```

> `extra_records` currently supports A / AAAA records — enough for this. Note the `base_domain`: it's just the tailnet's internal namespace, answered by the built-in resolver (`100.100.100.100`), so it **never touches public DNS or requires an ICP/备案 registration** — the same trick as part one's no-domain setup.

## Step 3: Deploy Caddy to route by hostname

I run Caddy in Docker on `core` with **host networking** — so it can bind the Tailscale address directly and reach other services on `127.0.0.1` without crossing network namespaces.

`compose.yaml`:

```yaml
services:
  caddy:
    image: caddy:2-alpine
    container_name: caddy
    restart: unless-stopped
    network_mode: host            # bind host network, proxy 127.0.0.1 directly
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - /etc/caddy:/certs:ro      # server.crt / server.key (read-only)
    healthcheck:
      test: ["CMD", "wget", "-q", "-O", "/dev/null", "http://127.0.0.1:2019/config/"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
```

`Caddyfile` (the key parts are `bind 100.64.12.1` + explicit cert):

```caddy
{
    auto_https off                 # disable auto-issuance; we bring our own cert
}

https://dufs.core.homemesh.internal {
    bind 100.64.12.1               # bind the Tailscale IP only
    tls /certs/server.crt /certs/server.key
    reverse_proxy 127.0.0.1:5000
}

https://navidrome.core.homemesh.internal {
    bind 100.64.12.1
    tls /certs/server.crt /certs/server.key
    reverse_proxy 127.0.0.1:4533
}

# … append one block per service …

# catch-all: http(80) → https, so typing the bare domain also works
http://*.core.homemesh.internal {
    bind 100.64.12.1
    redir https://{host}{uri} permanent
}
```

Start and verify:

```bash
docker compose up -d
curl -s --resolve dufs.core.homemesh.internal:443:100.64.12.1 \
     --cacert /etc/caddy/ca/ca.crt \
     -o /dev/null -w '%{http_code}\n' https://dufs.core.homemesh.internal/
# 401 (dufs anonymous, expected; an HTTP response means the proxy path works)
```

- `auto_https off` stops Caddy from trying to auto-issue, since the explicit `tls` directive loads our cert.
- **`bind 100.64.12.1` is the easiest line to miss and the most important**: without it, host networking makes Caddy listen on `0.0.0.0`, exposing these service names on the LAN or public interface. With it, 443/80 exist only inside the mesh.
- The trailing `http://*.core.homemesh.internal` wildcard handles "user typed the bare domain and the browser went to http/80", 301-ing everything to https.

## Step 4: Roll it out to every service

From here it's mechanical. Adding a service = three edits: one `extra_records` A record + one `Caddyfile` block + (if you hadn't already allowed 443) one ACL line. This is the table I ended up with:

| Service | Domain | Backend port |
| --- | --- | --- |
| dufs file server | `dufs.core.homemesh.internal` | 5000 |
| Navidrome music | `navidrome.core.homemesh.internal` | 4533 |
| qBittorrent downloads | `qbittorrent.core.homemesh.internal` | 8090 |
| n8n automation | `n8n.core.homemesh.internal` | 5678 |
| Calibre-Web library | `calibre-web.core.homemesh.internal` | 8083 |
| MinIO Console | `minio.core.homemesh.internal` | 9001 |
| Pocket-ID auth | `pocket-id.core.homemesh.internal` | 1411 |

Verification is cheap: curl each domain and check it returns that service's own response (a 200 home page, a 302 to login, or a 401 unauth all mean "the link works" — the proxy delivered the request to the right backend).

A small architectural bonus: because all traffic now flows through 443, **a deny-by-default ACL only needs to allow a single `443` for "client → core"** — the scattered service ports (5000, 4533, 5678…) no longer need individual exposure. Tighter least-privilege for free. (Whether you shrink the old whitelist is up to you; both can coexist.)

## Gotchas (all real, all hit in practice)

1. **Don't mint a fresh CA.** The knee-jerk move is Caddy's internal CA — but then the chain isn't anchored on any device, forcing a CA reinstall everywhere and wasting the CA you already trusted in part one. **Reuse the mesh CA.**
2. **Host networking and `bind` go together.** Forgetting `bind 100.64.12.1` makes Caddy listen on `0.0.0.0`, leaking internal service names to the LAN/public. Write bind in every block.
3. **`caddy reload` may ignore your new config.** After editing, `caddy reload --config` sometimes doesn't take effect (logs still show the old backend). Just `docker compose restart caddy` — it's seconds and it always works. Don't fight the reload.
4. **Some services only bind the Tailscale IP, not `127.0.0.1`.** Pointing the proxy backend at the wrong address (e.g. `127.0.0.1:1411` when the service only listens on `100.64.12.1:1411`) gives a 502. **Check `ss -tlnp` / `docker ps` first, and match the proxy target to what the service actually binds.**
5. **Keep certs ≤ 397 days.** Apple's 398-day cap means a 10-year cert works on Linux but is rejected on macOS/iOS (`OtherTrustValidityPeriod`). Re-sign before expiry and restart Caddy.
6. **Don't test certs with `curl -H 'Host: ...'`.** `-H Host` doesn't set SNI, so Caddy can't pick a matching cert and replies `TLS alert internal error` — making you think the cert is broken. Use `--resolve domain:443:IP`, or just visit the real domain in a browser.

## Security notes

- **Think about where `ca.key` lives.** Reusing the mesh CA gives the green lock but means `ca.key` exists on one more machine. Fine for home use. For stricter isolation, mint an intermediate CA for Caddy and cross-sign it under the mesh root — devices still trust it, while the intermediate's private key can be rotated/revoked without ever exposing the root key.
- **Bind the Tailscale IP only.** No `0.0.0.0`, no port forwarding — services stay invisible to the public internet and the LAN.
- Keep the ACL deny-by-default: client→core allows just 443; backend ports appear in no whitelist at all.
- Set a renewal reminder (397 days). Expiry shows up as an abrupt browser warning; re-sign and restart Caddy to fix.

## Wrap-up

In one line: **Tailscale makes the network reachable, Caddy makes the services memorable.** MagicDNS names each host, `extra_records` names each service on top of that, a wildcard cert anchored on the mesh CA handles the green lock, and Caddy fans every hostname out to the right port on 443.

The result: reaching an internal service goes from "memorize an IP and a port" to "remember the service name". Type `dufs.core.homemesh.internal` on a phone or laptop and the page opens — clean, fast, green. Three open-source components, zero hardware cost, just another floor on top of the foundation the earlier posts laid.

If you've got a pile of self-hosted services and you're sick of port numbers, this "Caddy + MagicDNS + mesh CA" combo solves it in one go. Hope this helps.