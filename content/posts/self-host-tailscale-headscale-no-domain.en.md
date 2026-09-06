+++
title = 'Self-Hosting a Tailscale Network from Scratch: Headscale Without a Domain'
date = '2026-09-05T12:00:00+08:00'
draft = false
tags = ['Tailscale', 'Headscale', 'VPN', 'Self-Hosted', 'Networking', 'Tutorial']
description = 'Build your own Tailscale-style mesh VPN with no domain and no ICP filing: a self-hosted Headscale control plane with an embedded DERP relay, IP + self-signed CA auth, and a Headplane web panel — with every command and the gotchas.'

[cover]
  image = 'covers/self-host-tailscale-headscale-no-domain.jpg'
+++

Tailscale is one of the best mesh-networking tools around: it pulls devices scattered across different networks — your home server, laptop, phone, company and overseas VPS — into a single virtual LAN, with peer-to-peer direct connections, automatic NAT traversal, built-in ACLs and MagicDNS. The catch: out of the box it connects to Tailscale's official coordination server, so the control plane isn't yours.

If you want to own the control plane, the standard answer is [Headscale](https://github.com/juanfont/headscale), an open-source implementation of the Tailscale control plane. But here's the problem: **almost every Headscale tutorial assumes you have a domain + ICP filing + public HTTPS**. In many places (mainland servers, for example) domain filing is slow, bureaucratic, and sometimes simply unobtainable.

This post documents a route that needs **no domain at all** — the same approach I use for my own network:

- **Control plane**: connect directly via public IP + a self-signed CA, no domain or filing needed
- **DERP relay**: use Headscale's built-in "embedded DERP", reusing the same certificate
- **MagicDNS**: use an internal namespace (`xxx.internal`) that never touches public DNS
- **Management panel**: add a Headplane instance for a web UI over nodes, users and ACLs

The whole thing unfolds in four steps: **① stand up the DERP server → ② install Tailscale on each node → ③ configure Headscale → ④ install the Headplane panel**. Every step ships copy-pasteable commands, and the pitfalls I actually hit are collected at the end.

> ⚠️ **Note on values**: every public IP in this post (`203.0.113.10`) is an RFC 5737 documentation address, and all hostnames, LAN IPs and usernames are fictional. Swap in your own real values as you follow along.

## Final topology

Here's the big picture; every step after this fills in a piece of the diagram:

```
                Public VPS (control plane + embedded DERP)
        ┌────────────────────────────────────────────────┐
        │  relay-1 · Ubuntu 24.04                        │
        │    Headscale 0.29.3      ← tcp 443             │
        │    Embedded DERP         ← tcp 443 + udp 3478 │
        │    Self-signed CA, SAN = 203.0.113.10          │
        └───────────────────┬────────────────────────────┘
                            │ public IP 203.0.113.10
   ──────────── internet ───┼──────────────────────────────
        │            │             │            │
   ┌────▼───┐   ┌────▼───┐   ┌─────▼───┐   ┌────▼────┐
   │ core    │   │ macbook│   │  winpc  │   │ cloud-eu│
   │100.64.  │   │100.64. │   │100.64.  │   │100.64.  │
   │ 12.1    │   │ 12.3   │   │ 12.5    │   │ 12.2    │
   └─────────┘   └────────┘   └─────────┘   └─────────┘
   (plus workcloud / nas / phone / pad, added later)
```

The virtual network uses Headscale's default CGNAT range `100.64.0.0/10`; node addresses are allocated by the control plane from that pool (examples here use `100.64.12.x` — your actual addresses show up in `tailscale status`).

| Node | Role | OS | Tailscale address (example) |
| --- | --- | --- | --- |
| relay-1 | Control plane + DERP (public VPS) | Ubuntu 24.04 | not a node |
| core | Home main server | Arch Linux | 100.64.12.1 |
| macbook | Daily laptop | macOS | 100.64.12.3 |
| winpc | Gaming / main desktop | Windows 11 | 100.64.12.5 |
| cloud-eu | Overseas VPS | Linux | 100.64.12.2 |
| workcloud | Company cloud server | Linux | 100.64.12.4 |
| nas | Another LAN box | Arch Linux | 100.64.12.6 |
| phone | Phone | iOS | 100.64.12.7 |
| pad | Tablet | Android (HyperOS) | 100.64.12.8 |

## Prerequisites: a server with a public IP

The whole setup needs one anchor: a cloud server with a **static public IP** (the smallest instance of any provider is plenty). It hosts the control plane and the DERP relay. I'll call it `relay-1` with the example public IP `203.0.113.10`.

Two things to do up front:

1. **Open the firewall / security-group ports** (public ingress):

| Port | Protocol | Purpose |
| --- | --- | --- |
| 443 | TCP | Headscale control plane + DERP (same port) |
| 3478 | UDP | STUN (NAT traversal / DERP discovery) |

2. **Check the in-server firewall**: if iptables/nftables is on, allow those ports; simplest is to leave INPUT at default ACCEPT and let the cloud security group be the gatekeeper.

> Why host DERP and the control plane on the same box? Because Headscale's embedded DERP **reuses the control plane's TLS certificate**. Once a client trusts your CA, both the control plane and DERP verify in one go — no extra certs or ports. That's one of the pillars that makes the no-domain approach work.

---

## Step 1: Stand up the DERP server (which also brings up the control plane)

A clarification first: the "DERP server" here is not a separate `derper` process — it's **Headscale's embedded DERP**. It lives in the same process and on the same port as the control plane, and enabling it is just a few lines in `config.yaml`. So this step is really = **install Headscale + generate a self-signed cert + enable the embedded DERP**.

### 1.1 Install Headscale

On relay-1 (Ubuntu 24.04):

```bash
# as root
VERSION=0.29.3
wget -O headscale.deb \
  "https://github.com/juanfont/headscale/releases/download/v${VERSION}/headscale_${VERSION}_linux_amd64.deb"

# If GitHub releases are slow or unreachable (common on mainland servers), use a mirror:
# wget -O headscale.deb \
#   "https://ghfast.top/https://github.com/juanfont/headscale/releases/download/v${VERSION}/headscale_${VERSION}_linux_amd64.deb"

dpkg -i headscale.deb
systemctl enable headscale
```

The DEB creates the `headscale` system user and the `/etc/headscale` + `/var/lib/headscale` directories for you.

### 1.2 Generate the self-signed certificate (the heart of "no domain")

With no domain and no Let's Encrypt, you terminate TLS on the control plane with a self-signed cert — put the **public IP into the certificate's SAN** and clients can connect via `https://<IP>` directly.

```bash
mkdir -p /etc/headscale/ca && cd /etc/headscale/ca

# ① CA private key + self-signed root (the root can live long)
openssl ecparam -genkey -name prime256v1 -out ca.key
openssl req -new -x509 -key ca.key -sha256 -days 3650 -out ca.crt \
  -subj "/CN=HomeMesh Root CA"

# ② Server key + CSR
openssl ecparam -genkey -name prime256v1 -out server.key
openssl req -new -key server.key -out server.csr -subj "/CN=relay-1"

# ③ Extension file: SAN holds the public IP (add more IPs/hostnames if you have endpoints)
cat > extfile.cnf <<'EOF'
subjectAltName = IP:203.0.113.10
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
EOF

# ④ Sign the server cert with the CA
#    ⚠️ Validity must be ≤ 397 days: Apple enforces a 398-day cap,
#    and a 10-year cert gets rejected by macOS/iOS.
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -days 397 -sha256 -extfile extfile.cnf

# ⑤ Permissions: CA key stays root-only; server key must be readable by the headscale user
chown root:root ca.key && chmod 600 ca.key
chgrp headscale server.key server.crt && chmod 640 server.key server.crt
```

You now have three artifacts to distribute or keep:

- `ca.crt` — copied to **every client** and installed into its system trust store (used repeatedly in Step 2)
- `server.crt` / `server.key` — stay on relay-1 only, used by Headscale
- `ca.key` — stays on relay-1 only, **never leaves the box**

> **What if you get a domain later?** Then it's simpler: skip self-signing and set `tls_letsencrypt_hostname` + `acme_email` in `config.yaml` to have Headscale fetch a Let's Encrypt cert automatically — and clients no longer need a manually-installed CA. Self-signing is the fallback for "no domain".

### 1.3 Configure config.yaml and enable the embedded DERP

Edit `/etc/headscale/config.yaml`. The fields that matter for this setup (see the official config-example for the full list):

```yaml
# ── Control-plane URL: a bare public IP, the no-domain trick ──
server_url: https://203.0.113.10
listen_addr: 0.0.0.0:443

# ── Self-signed cert paths (from Step 1.2) ──
tls_cert_path: /etc/headscale/ca/server.crt
tls_key_path: /etc/headscale/ca/server.key

# ── Virtual address pool: Headscale's default CGNAT range ──
prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48
  allocation: sequential

# ── Embedded DERP (the star of Step 1) ──
derp:
  server:
    enabled: true
    region_id: 999
    region_code: "homemesh"
    region_name: "HomeMesh Embedded DERP"
    verify_clients: true
    stun_listen_addr: "0.0.0.0:3478"          # STUN — pair with the 3478/udp security group rule
    private_key_path: /var/lib/headscale/derp_server_private.key
    automatically_add_embedded_derp_region: true
    ipv4: 203.0.113.10                          # public address advertised to clients
  urls:
    - https://controlplane.tailscale.com/derpmap/default   # keep official DERP as fallback
  paths: []
  auto_update_enabled: true
  update_frequency: 3h

# ── Database: sqlite is the easy starting point ──
database:
  type: sqlite
  sqlite:
    path: /var/lib/headscale/db.sqlite

# ── DNS / MagicDNS: internal namespace, never touches public DNS ──
dns:
  magic_dns: true
  base_domain: homemesh.internal
  override_local_dns: false          # ⚠️ keep false — see gotcha #8 at the end
  nameservers:
    global:
      - 1.1.1.1
      - 1.0.0.1

# ── Store the ACL in the DB so Headplane can edit it ──
policy:
  mode: database

log:
  level: info
```

A few things worth spelling out:

- `server_url` is the external address clients log in to with `tailscale up`. Writing `https://<public-ip>` means no domain is involved.
- `derp.server.enabled: true` turns on the embedded DERP; `region_code` is your custom identifier and `ipv4` is the public IP.
- `base_domain` is only the tailnet's **internal namespace**, resolved by the client's built-in resolver (`100.100.100.100`) and never touching public DNS.
- `override_local_dns: false` lets MagicDNS own only `*.homemesh.internal`, while public names keep resolving via each node's local DNS.

### 1.4 Start and verify the control plane + DERP

```bash
# Use restart after editing config (not enable --now — the DEB already started the service)
systemctl restart headscale
systemctl status headscale

# Verify the 443 control plane + self-signed cert
curl -sk https://203.0.113.10/ | head -n 5

# Verify STUN is listening
ss -ulnp | grep 3478

# Follow the logs
journalctl -u headscale -f
```

`curl` uses `-k` because the cert is self-signed; any normal response means TLS is up. **Control plane + DERP + STUN are now live**, ready for nodes.

---

## Step 2: Install Tailscale on each node and join

Repeat four actions on every device: **install Tailscale → install the CA → restart tailscaled → `tailscale up`**. Order matters — especially "restart tailscaled after installing the CA": tailscaled is a Go program whose system CA pool is loaded **once at process start**, so without a restart it won't recognize your self-signed CA and will raise `x509: certificate signed by unknown authority`.

### 2.1 Linux (core / cloud-eu / workcloud / nas)

Debian/Ubuntu example:

```bash
# ① Install the Tailscale client
curl -fsSL https://tailscale.com/install.sh | sh
# On Arch Linux instead:
# sudo pacman -S tailscale

# ② Install the self-signed CA (copy relay-1's ca.crt to this box first)
#    Debian/Ubuntu:
sudo cp ~/ca.crt /usr/local/share/ca-certificates/homemesh-ca.crt
sudo update-ca-certificates
#    Arch Linux:
# sudo cp ~/ca.crt /etc/ca-certificates/trust-source/anchors/homemesh-ca.crt
# sudo trust extract-compat

# ③ Restart tailscaled — REQUIRED, reloads the CA pool
sudo systemctl restart tailscaled

# ④ Join (--authkey is generated in Step 3.1)
sudo tailscale up \
  --login-server=https://203.0.113.10 \
  --authkey=<PREAUTH_KEY> \
  --hostname=core
```

### 2.2 macOS

```bash
# ① Install the CLI via brew (NOT the App Store version — it's locked to the official control plane)
brew install tailscale

# ② Add the CA to the system keychain (putting it in the System keychain makes it a trust anchor)
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain ~/ca.crt

# ②' Verify the system now trusts this CA (no certificate error = success)
security verify-cert -c ~/ca.crt
curl -I https://203.0.113.10

# ③ Restart tailscaled (brew service) — required, reloads the CA pool
sudo brew services restart tailscale
# or: sudo launchctl kickstart -k system/homebrew.mxcl.tailscale

# ④ Join
sudo tailscale up \
  --login-server=https://203.0.113.10 \
  --authkey=<PREAUTH_KEY> \
  --hostname=macbook
```

> macOS has a special gotcha: **the server cert's validity must be ≤ 397 days**. A 10-year cert works fine on Linux clients but macOS fails repeatedly with `Trust evaluate failure: [leaf OtherTrustValidityPeriod]` — Apple enforces a 398-day cap. That's why Step 1.2 uses `-days 397`. The CA itself can be long-lived (10 years); only the server (leaf) cert must stay short.

### 2.3 Windows

1. Download and install the official `.msi`.
2. Import the CA into the "Trusted Root Certification Authorities" store:

```powershell
# Administrator PowerShell
Import-Certificate -FilePath .\ca.crt -CertStoreLocation Cert:\LocalMachine\Root
Restart-Service Tailscale
```

3. Connect: tray icon → Log in → top-right **•••** → **Use custom coordination server** → `https://203.0.113.10`. Or via CLI:

```powershell
tailscale up --login-server=https://203.0.113.10 --authkey=<PREAUTH_KEY> --hostname=winpc
```

### 2.4 iOS

The official App Store Tailscale app **natively supports a custom coordination server** — no third-party app needed:

1. Get `ca.crt` onto the phone (AirDrop / a download link / a temporary HTTP server), open it in Safari and install the profile.
2. Turn on **full trust** (the step everyone misses): Settings → General → About → **Certificate Trust Settings** → find your CA → flip the switch on.
3. Verify: reopen `https://203.0.113.10` in Safari (close the old tab first to bypass cache) — if the "This connection is not private" warning is gone, system-level trust is OK.
4. Reopen the Tailscale app → Log in → **•••** → **Use custom coordination server** → `https://203.0.113.10` → pick your user on the auth page.

> ⚠️ **Installing the profile ≠ trusting the cert.** Without full trust, Safari keeps showing "This connection is not private", the control plane logs spam `tls: bad certificate`, and login keeps failing. See gotcha #7.

### 2.5 Android

The official Play / F-Droid Tailscale app **natively supports a custom coordination server too** — no third-party app needed:

1. Get `ca.crt` onto the tablet (a download link / temporary HTTP server both work).
2. **Install it as a "CA certificate"** (the critical, easy-to-get-wrong step): Settings → Security / Passwords & security → Encryption & credentials → Install a certificate → **CA certificate** → pick the downloaded `ca.crt`. (Menus vary by brand — look for the words "CA certificate".)
3. Verify you got the type right: Settings → … → Encryption & credentials → Trusted credentials → the **User** tab should list your CA (e.g. `HomeMesh Root CA`). If it's not there, you installed the wrong type.
4. Open the Tailscale app → top-right settings/avatar → Accounts → top-right **⋮** → **Use an alternate server** → `https://203.0.113.10`.
5. Tap **⋮** again → **Use an auth key** → paste the pre-auth key.
6. Back on the main screen, tap Log in / Connect.

> ⚠️ **Android's gotcha is different from iOS**: iOS is "install the profile → flip the full-trust switch"; Android has no such switch — **trust depends on which certificate type you install**. It must be a "CA certificate" (landing in the system trust anchors), not a "VPN and apps" certificate (just an app/private-key credential the app ignores — the control plane log will still spam `tls: bad certificate`). Tapping the downloaded file directly often installs the wrong type. See gotcha #9.

> ⚠️ There's also an **upstream known bug** in the Android app with self-signed certs: even with the CA installed correctly, DERP relay connections may still not trust it, showing `no-derp-connection` / `tls-connection-failed` health warnings. In practice, the tablet reaches peers **directly when on the same LAN**, but when away (cellular / another Wi-Fi) and relying on DERP relay, it may fail. For mobile devices that must work anywhere, prefer a publicly-trusted cert (Let's Encrypt / a domain) over self-signed.

### 2.6 Verify the join

```bash
tailscale status                 # nodes and online state
tailscale ip -4                  # this node's allocated Tailscale address

tailscale ping 100.64.12.2       # "p2p" = direct, "via <addr>" = relayed over DERP
getent hosts core.homemesh.internal   # verify MagicDNS
```

Once every node is in, `headscale nodes list` on relay-1 should show each one with its `100.64.12.x` address.

---

## Step 3: Configure Headscale (no official panel)

The network is up. Now shape it into what you want: users, pre-auth keys, MagicDNS, ACLs. Everything happens on relay-1 via the `headscale` CLI — which is exactly what "not using the official coordination server" buys you: full control.

### 3.1 Users and pre-auth keys

```bash
# Create a user (nodes belong to this user)
sudo headscale users create sam
sudo headscale users list
# id | name
# 1  | sam

# Generate a pre-auth key — feed it as --authkey when nodes join
# ⚠️ In 0.29.x, `preauthkeys create --user` wants the NUMERIC id (from the id column),
#    passing a name errors with a strconv.ParseUint failure
sudo headscale preauthkeys create --user 1 --expiration 24h
# the key.xxx line in the output = <PREAUTH_KEY>, use it in Step 2

sudo headscale preauthkeys list --user 1
```

### 3.2 Node management

```bash
sudo headscale nodes list          # online state / allocated IPs
sudo headscale nodes delete <id>   # remove a node
sudo headscale nodes expire <id>   # expire a node

# always restart after editing config.yaml
sudo systemctl restart headscale
sudo journalctl -u headscale -f
```

### 3.3 Configuring MagicDNS

MagicDNS is Headscale's built-in private DNS — it lets every device reach the others by **internal hostname** instead of memorizing IPs. Its resolution never touches public DNS and needs no filing, so it works fine with the no-domain approach.

All config lives in the `dns` block of `config.yaml` (shown in Step 1.3). Field by field:

```yaml
dns:
  magic_dns: true                 # master switch: turn on the built-in DNS
  base_domain: homemesh.internal  # internal namespace; node FQDN = <hostname>.<base_domain>
  override_local_dns: false       # ⚠️ only own .internal; public names still use local DNS (gotcha #8)
  nameservers:
    global:                       # fallback upstream (internal records are answered from the node table)
      - 1.1.1.1
      - 1.0.0.1
    split: {}                     # optional: route specific domains to specific upstreams
  search_domains: []              # optional: e.g. [homemesh.internal]
  extra_records: []               # optional: hand-written static A/AAAA records
```

How it works:

- After connecting, each client is given a built-in resolver address `100.100.100.100`; queries for `*.homemesh.internal` are routed to it and answered by Headscale from the **current online-node table**.
- A node's FQDN = `--hostname value + base_domain`. `--hostname=core` → `core.homemesh.internal`, so keep hostnames free of spaces and special characters.
- To add a purely static record that isn't a node, use `extra_records`:

```yaml
dns:
  extra_records:
    - name: "dash.homemesh.internal"
      type: "A"
      value: "100.64.12.1"
```

After editing, `systemctl restart headscale`; if a client doesn't pick up new records immediately, restart its tailscaled to force a pull. Verify:

```bash
# on any node
getent hosts core.homemesh.internal    # → 100.64.12.1
curl http://core.homemesh.internal:8080/
nslookup dash.homemesh.internal        # custom record
```

### 3.4 ACLs (deny by default, allow what you need)

Headscale's ACLs use the `grants` format, and **anything unmatched is denied** (deny-by-default) — a good minimum-privilege baseline. Each grant has three fields:

- `src`: source address(es)
- `dst`: destination address(es)
- `ip`: allowed destination port(s), or `*` for all

If a "source × destination × port" combination isn't covered by any grant, the connection is refused. Storing the policy in the DB (`policy.mode: database`) also lets Headplane edit it from the UI.

Write `/etc/headscale/acl-policy.hujson`:

```hujson
{
  "grants": [
    // 1) servers can reach each other: full access
    {
      "src": ["100.64.12.1", "100.64.12.2", "100.64.12.4", "100.64.12.6"],
      "dst": ["100.64.12.1", "100.64.12.2", "100.64.12.4", "100.64.12.6"],
      "ip": ["*"]
    },
    // 2) personal devices (laptop/desktop/phone/tablet) can reach each other: full access
    {
      "src": ["100.64.12.3", "100.64.12.5", "100.64.12.7", "100.64.12.8"],
      "dst": ["100.64.12.3", "100.64.12.5", "100.64.12.7", "100.64.12.8"],
      "ip": ["*"]
    },
    // 3) main server core → personal devices: full access (for remote troubleshooting)
    {
      "src": ["100.64.12.1"],
      "dst": ["100.64.12.3", "100.64.12.5", "100.64.12.7", "100.64.12.8"],
      "ip": ["*"]
    },
    // 4) personal devices → core: whitelisted ports only (example, adjust as needed)
    {
      "src": ["100.64.12.3", "100.64.12.5", "100.64.12.7", "100.64.12.8"],
      "dst": ["100.64.12.1"],
      "ip": ["22", "443", "3000", "8080", "25565"]
    },
    // 5) personal devices → nas: SSH only
    {
      "src": ["100.64.12.3", "100.64.12.5", "100.64.12.7", "100.64.12.8"],
      "dst": ["100.64.12.6"],
      "ip": ["22"]
    }
  ]
}
```

Dry-run first, then apply and view:

```bash
# validate syntax/structure before applying
sudo headscale policy check -f /etc/headscale/acl-policy.hujson
sudo headscale policy set -f /etc/headscale/acl-policy.hujson
sudo headscale policy get      # view the currently active policy
```

> The logic: **server group ↔ server group, and personal-device group ↔ personal-device group, all open; the main server has full control over the other servers; personal devices reach the main server's services only through a port whitelist.** Anything not listed (e.g. database port 5432) is denied by default.

---

## Step 4: Install Headplane (your own web panel)

Headscale ships only a CLI, no GUI. For day-to-day point-and-click tasks — viewing nodes, removing devices, issuing pre-auth keys, editing ACLs — use **Headplane**, which manages Headscale over its HTTP API and doesn't need to run on the same machine as Headscale.

I run it in Docker on the main server `core` (inside the tailnet, so the panel stays private).

### 4.1 Create a Headscale API key

Headplane authenticates with a Headscale API key, generated on relay-1:

```bash
sudo headscale apikeys create --expiration 720h
# output: <prefix>.<secret> — the secret prints only once; copy the whole line for the login
sudo headscale apikeys list       # afterwards you only see the masked prefix; just create a new one if lost
```

### 4.2 Deploy Headplane (Docker Compose)

On `core`, create `/opt/headplane` with `compose.yaml`:

```yaml
services:
  headplane:
    image: ghcr.io/tale/headplane:0.7.1
    container_name: headplane
    restart: unless-stopped
    environment:
      # Headscale uses a self-signed CA — feed it to the container's Node outbound requests
      NODE_EXTRA_CA_CERTS: /etc/headplane/ca.crt
      TZ: Asia/Shanghai
    ports:
      # bind only to localhost + the tailnet, never the public internet
      - "127.0.0.1:8088:3000"
      - "100.64.12.1:8088:3000"
    volumes:
      - ./config.yaml:/etc/headplane/config.yaml:ro
      - /etc/ca-certificates/trust-source/anchors/homemesh-ca.crt:/etc/headplane/ca.crt:ro
      - /data/headplane:/var/lib/headplane
```

The companion `config.yaml` (same directory):

```yaml
server:
  host: "0.0.0.0"
  port: 3000
  # reach the panel over the tailnet so Mac/phone can open it too
  base_url: "http://100.64.12.1:8088"
  # 32 chars, generated with: openssl rand -hex 16
  cookie_secret: "<paste>"
  # plain HTTP (Tailscale already encrypts the transport); must be false or browsers refuse the cookie
  cookie_secure: false
  data_path: "/var/lib/headplane"

headscale:
  # Headscale's HTTP API (same 443 port as the control plane, self-signed TLS)
  # no tls_cert_path here — trust the self-signed CA via NODE_EXTRA_CA_CERTS instead (see compose.yaml)
  url: "https://203.0.113.10"
```

Start and verify:

```bash
cd /opt/headplane
openssl rand -hex 16   # generate cookie_secret and paste it into config.yaml
docker compose up -d
docker compose logs -f
```

### 4.3 Open the panel

Browse to `http://100.64.12.1:8088/admin` from any device in the tailnet (phone included), paste the **Headscale API key** into the login field, and you get a GUI for users, nodes, pre-auth keys, API keys and ACLs.

---

## Gotchas (all hit in real life)

Roughly in order of how often they bite:

1. **Installed the CA but didn't restart tailscaled → `x509 unknown authority`.** A Go process loads the CA pool once at startup, while `openssl s_client` re-reads it every time and passes — super misleading. Mantra: **install CA → restart tailscaled → up**.
2. **Headscale 0.29 `preauthkeys create --user` wants a numeric ID.** `users create` takes a name, `preauthkeys create` takes an ID; passing a name throws `ParseUint`. Look it up with `users list`.
3. **After the DEB install the service is already running — `enable --now` won't restart it.** The postinst started the service with the default config, so you must `systemctl restart headscale` after editing.
4. **Permission problems**: `/var/lib/headscale` (`derp_server_private.key` / `noise_private.key`) must be owned by the headscale user, and `server.key` needs `chgrp headscale && chmod 640`. Don't run `headscale serve` in the foreground as root — it creates root-owned state files.
5. **Mainland servers fail to fetch the DEB straight from GitHub.** Release assets on `objects.githubusercontent.com` are often blocked; prefix with `ghfast.top/` for a mirror.
6. **macOS rejects long-lived certs** (`OtherTrustValidityPeriod`): Apple caps at 398 days, so sign the server cert for 397 days (the CA can be long-lived).
7. **On iOS, installing the profile ≠ trusting it.** You must also flip the switch in Certificate Trust Settings, otherwise you get `tls: bad certificate` and repeated login failures. When debugging phone joins, first grep the control-plane log for that line.
8. **Don't set `override_local_dns: true`.** It forces all public DNS through the configured nameservers (e.g. 1.1.1.1), which can be very slow or time out on some networks — manifesting as browser lag. Keep it `false` so MagicDNS only owns `.internal`.
9. **On Android, install the CA as a "CA certificate" type — and mind the DERP/self-signed bug.** Android has no iOS-style full-trust switch: trust depends on the cert type ("CA certificate" → trust anchor; "VPN and apps" → ignored, control plane still logs `tls: bad certificate`). Even installed correctly, DERP relay may still distrust the self-signed CA (`no-derp-connection`); same-LAN direct connections are unaffected, but cross-network use may need a publicly-trusted cert.

## Security notes

- `ca.key` stays on relay-1 only (root, mode 600); only `ca.crt` gets distributed.
- Keep the server cert at ≤397 days (Apple limit); re-sign and `systemctl restart headscale` before it expires.
- Run ACLs in **deny-by-default** and allowlist only the ports you need.
- On relay-1, open only 443/tcp + 3478/udp in the security group; everything else stays closed.
- Rotate API keys regularly (`apikeys create` to mint new, `apikeys expire` to revoke old).

## Summary

The whole approach boils down to one idea: **take Tailscale's control plane back into your own hands with Headscale, then erase the domain/filing prerequisite with "public IP + self-signed CA".** The embedded DERP reuses that same certificate, MagicDNS lives in an internal namespace, and a Headplane panel makes daily management visual.

The total cost is one minimal public VPS. In return you get peer-to-peer direct connections across networks, MagicDNS internal names, deny-by-default ACLs, and freedom from the official coordination server. Hope this record helps with your own mesh-networking build.