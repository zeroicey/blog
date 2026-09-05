+++
title = 'High-Performance Remote Desktop over Tailscale: Sunshine + Moonlight + a DP Dummy Plug'
date = '2026-09-05T15:00:00+08:00'
draft = false
tags = ['Tailscale', 'Headscale', 'Sunshine', 'Moonlight', 'RustDesk', 'Remote Desktop', 'Self-Hosted', 'Tutorial']
description = 'Give a monitor-less server a remote desktop: a DP dummy plug fakes a display, Sunshine encodes with an AMD iGPU in hardware, and Moonlight streams over a Tailscale P2P connection. Compare with RustDesk soft-encoding, and self-host a RustDesk relay.'
+++

The previous post, *Self-Hosting Tailscale from Scratch*, pulled all my machines into one virtual LAN. Every device can now reach every other by address — but "reachable" and "feels like sitting at the keyboard" are two different things.

One of those machines is my "ops center": an Arch Linux server with no monitor attached, running Docker, Samba and SSH in a corner of the room, permanently dark. It *does* have a graphical desktop (for the occasional GUI tool), it just has no screen — a classic **headless host**.

For a long time its remote control was **RustDesk**. It worked, but two things always annoyed me: **high CPU** while being controlled, and mediocre latency.

The root cause, once I traced it: RustDesk on Linux + AMD iGPU basically falls back to **software encoding** — its hardware acceleration is solid for NVIDIA (NVENC) and Intel (QSV), but weak for AMD. And this machine happens to have an AMD iGPU (Radeon 780M), so every session made the CPU churn through software video encoding.

Since Tailscale had already solved the transport (P2P, NAT traversal, ACLs), what I really needed was one thing: **a controlled endpoint that encodes with the GPU**. The answer is **Sunshine + Moonlight** — the pair designed for game streaming, which turns out to be excellent as a remote desktop.

This post records the whole setup in three layers:

1. **Hardware**: a DP dummy plug — how to make a GPU believe a display is attached when there is none;
2. **Software**: installing and tuning Sunshine (host) and Moonlight (client);
3. **Comparison**: why RustDesk still lives on, and how to self-host its relay pair (hbbs/hbbr).

> ⚠️ **Placeholder note**: as in the previous post, every IP below uses the example `100.64.12.x` range, and hostnames/usernames are fictional. Substitute your own.

## The big picture: one path, three roles

```
                    ┌─────────────────────────────────────────┐
                    │  core（headless host, no monitor）       │
                    │    Arch Linux · AMD Radeon 780M iGPU     │
                    │    ┌──────────────┐                      │
                    │    │  DP dummy    │  ← fakes a 1080p panel │
                    │    └──────┬───────┘                      │
                    │           │ DisplayPort                   │
                    │    ┌──────▼───────┐                      │
                    │    │ lightdm+bspwm│  ← minimal GUI session │
                    │    └──────┬───────┘                      │
                    │    ┌──────▼───────┐                      │
                    │    │   Sunshine   │  ← capture + HW encode │
                    │    └──────┬───────┘                      │
                    └───────────┼──────────────────────────────┘
                                │ Tailscale P2P (100.64.12.1)
     ─────────  virtual LAN ────┼──────────────────────────────
              │                  │                  │
        ┌─────▼─────┐      ┌─────▼─────┐      ┌─────▼─────┐
        │  macbook  │      │   winpc   │      │   phone   │
        │  100.64.  │      │  100.64.  │      │  100.64.  │
        │   12.3    │      │   12.5    │      │   12.7    │
        │  Moonlight│      │  Moonlight│      │  Moonlight│
        └───────────┘      └───────────┘      └───────────┘
```

The path: **Sunshine captures the host desktop and hardware-encodes it into a video stream → pushed over Tailscale P2P to the client → Moonlight decodes it and sends input back**. No public ports, no relay, everything inside the virtual LAN built in the previous post.

| Node | Role | OS | Tailscale addr (example) |
| --- | --- | --- | --- |
| core | controlled host (Sunshine server) | Arch Linux + AMD iGPU | 100.64.12.1 |
| macbook | client (Moonlight) | macOS | 100.64.12.3 |
| winpc | client (Moonlight) | Windows 11 | 100.64.12.5 |
| phone | client (Moonlight) | iOS | 100.64.12.7 |

## Hardware: a DP dummy plug that fakes a display

### Why you need one

A GPU's output port (HDMI/DP) learns about a display through an **EDID** blob — resolution, refresh rate, vendor, capabilities. With nothing attached, the system reads no EDID; the graphical session may not even start, or comes up stuck at a broken resolution (like 640×480), which looks awful over remote control.

For a machine "with a desktop but never a screen", the most robust fix is not fiddling with X11 virtual outputs — it's a $5 **DisplayPort/HDMI dummy plug**: a thumb-sized dongle with a tiny EEPROM holding an EDID; plug it in and the system believes a real panel is attached.

This setup uses a 1080p DP dummy; after plugging it in, `xrandr` shows:

```
DisplayPort-0 connected primary 1920x1080+0+0
   1920x1080     60.00*+  59.94  ...   ← EDID preferred mode is 1080p@60
```

### Two practical tips

1. **Lock it to 1080p.** Some dummies expose up to 4K in their EDID, but encoding cost scales with resolution; 1080p is the sweet spot for office-style remote control. Don't chase 4K "for sharpness".
2. **Keep it, don't replace it with a virtual display.** Sunshine has its own virtual-display path, but it's fiddlier on AMD. A physical dummy is reliable and cheap — it's the foundation of the whole headless setup.

Pair it with a minimal GUI stack and the machine becomes "no screen, but a real desktop":

```
lightdm (autologin) → Xorg :0 → bspwm tiling WM + polybar status bar
```

`lightdm` is set to auto-login so Sunshine always has a full desktop session right after boot, no human needed. In `lightdm.conf`:

```ini
[Seat:*]
autologin-user=<your-user>
autologin-session=bspwm
autologin-user-timeout=0
```

## Host: installing and configuring Sunshine

### Step 1: verify your GPU can hardware-encode

This is the premise of everything — check your GPU supports hardware encoding (at least one of H.264/HEVC/AV1). Install `libva-utils` (provides `vainfo`):

```bash
sudo pacman -S libva-utils    # Arch; Debian/Ubuntu: sudo apt install vainfo
vainfo
```

Look for `VAEntrypointEncSlice` (the encode entrypoint). A Radeon 780M reports all three:

```
VAProfileH264High               : VAEntrypointEncSlice   ← H.264 encode ✓
VAProfileHEVCMain               : VAEntrypointEncSlice   ← HEVC encode ✓
VAProfileAV1Profile0            : VAEntrypointEncSlice   ← AV1 encode ✓
```

If you only see `VLD` (decode) and no `EncSlice`, you can't hardware-encode. Sunshine actually uses the newer Vulkan Video encoders (`h264_vulkan` / `hevc_vulkan` / `av1_vulkan`) on AMD anyway — same hardware, same result.

### Step 2: install Sunshine

**Here's the catch**: Sunshine is *not* in the Arch official repos, and the AUR `sunshine`/`sunshine-bin` packages make you wait forever (slow source builds, slow AUR metadata sync). Fastest route: grab a **prebuilt package from the official GitHub release**:

```bash
# 1) Find the latest version tag
#     https://github.com/LizardByte/Sunshine/releases/latest
#     (from CN, direct connection is often faster than the proxy — see pitfalls):
#     curl --noproxy '*' -s https://api.github.com/repos/LizardByte/Sunshine/releases/latest

# 2) Arch: download the .pkg.tar.zst and install locally
sudo pacman -U sunshine-<version>-x86_64.pkg.tar.zst

#    Debian/Ubuntu use the matching .deb:
#    sudo dpkg -i sunshine-ubuntu-24.04-amd64.deb
#    Fedora uses .rpm, or install the org.lizardbyte.app.Sunshine Flatpak
```

The release also ships a `sunshine.AppImage` for a zero-install run, but for a service, a real package is cleaner.

Downloads are the part that bites people in CN networks — see pitfalls 1 and 2.

### Step 3: set credentials and start

```bash
# Web admin panel credentials (for tuning later)
sunshine --creds <username> <password>

# Start as a user service (follows your graphical session)
systemctl --user enable --now sunshine
systemctl --user status sunshine
```

Sunshine is a **user-level service** — it must run inside your graphical session to capture the `DISPLAY=:0` X11 desktop. After starting, check the log for two things: display detected, hardware encoder active:

```bash
journalctl --user -u sunshine -n 50
# expect:
#   Detected display: DisplayPort-0 ... connected: true
#   Found HEVC encoder: hevc_vulkan [vulkan]
#   Found AV1 encoder: av1_vulkan [vulkan]
```

### Step 4: ports Sunshine uses

| Port | Proto | Purpose |
| --- | --- | --- |
| 47984 | TCP | HTTP (discovery / pairing) |
| 47989 | TCP | HTTPS / GameStream main |
| 47990 | TCP | Web admin UI |
| 48010 | TCP | RTSP / main connection |
| 47998–48000 | UDP | video stream (RTP) |
| 48002 | UDP | control |

⚠️ **With a deny-by-default Tailscale ACL** (the "unmatched = deny" policy from the previous post's Step 3.4), you *must* add these ports to the "clients → core" allowlist, or the client is silently dropped before it can even handshake:

```hujson
{
  "src": ["100.64.12.3", "100.64.12.5", "100.64.12.7"],
  "dst": ["100.64.12.1"],
  "ip": ["47984", "47989", "47990", "48010", "47998-48000", "48002"]
}
```

Re-run `headscale policy set` afterwards.

### Step 5: how to reach the Web admin UI

Sunshine's Web UI (`47990`) denies non-LAN origins by default. Tailscale addresses are CGNAT (`100.64.0.0/10`), so it treats them as "WAN" and 403s. Two options:

1. On the host itself, open `https://127.0.0.1:47990`;
2. For remote tuning, use an SSH tunnel (recommended):

```bash
ssh -L 47990:localhost:47990 core
# then open https://localhost:47990 in a browser
```

For "occasionally pop in" use, you can skip the Web UI entirely — default hardware encode + 30fps is already good; parameters are tuned mainly on the Moonlight side (next section).

## Client: tuning Moonlight

Official clients exist everywhere:

| Platform | Get it from |
| --- | --- |
| Windows | Microsoft Store / GitHub `moonlight-stream/moonlight-qt` |
| macOS | App Store / moonlight-qt |
| iOS | App Store "Moonlight Game Streaming" |
| Android | Play / GitHub moonlight-android |

### Connect and pair

1. Add Host → IP = the host's Tailscale address `100.64.12.1`, port default (47989).
2. First connect **pairs**: Sunshine generates a 4-digit PIN, enter it in the client. A headless host can't show the PIN overlay, so grab it from the host log:

```bash
journalctl --user -u sunshine -n 50 | grep -i pin
# Enter the following PIN on the Moonlight client: XXXX
```

### Tuning for remote work (not gaming)

This is the easiest place to go wrong. Moonlight's defaults target high-fps gaming, which is pure waste for remote office use. After testing, the combo that feels best:

| Setting | Recommended | Why |
| --- | --- | --- |
| Resolution | **1920×1080** | match the dummy plug; don't pick 4K |
| Frame rate | **30 FPS** | smooth enough for office work; 60fps is for games and doubles cost |
| Bitrate | **10 Mbps** (HEVC) / 20 Mbps (H.264) | the sweet spot for 1080p static content; higher is waste |
| Codec | **HEVC (H.265)** | the host already HW-encodes HEVC; clearer than H.264 at equal bitrate; "Auto" is fine |
| Hardware decode | on (default) | VideoToolbox on Mac/iOS, D3D11VA on Windows — the client stays cheap too |

One line: **1080p + 30fps + HEVC + 10 Mbps**. If the cursor feels laggy, bump to 60fps / 20 Mbps — but the base set is enough and cheapest for daily work.

> Real-world case: two clients on the same host, one at 77 Mbps and the other at 7 Mbps — visually nearly identical, but the former is just needlessly taxing the encoder and the network. Bitrate is not "higher = better".

## Comparison & coexistence: why RustDesk still lives here

Switching to Sunshine didn't make me delete RustDesk wholesale — and the reason matters, because the two are **not substitutes; they divide labor**.

| Dimension | RustDesk | Sunshine + Moonlight |
| --- | --- | --- |
| Encoding | mostly software (weak on AMD) | hardware (Vulkan/VA-API) |
| Host CPU | high | very low |
| Latency / quality | medium | low / good |
| Network | self-hosted hbbs/hbbr relay | P2P (over Tailscale) |
| Headless support | mediocre | good (with dummy plug) |
| Best at | linking arbitrary devices, non-encodeable devices | a hardware-encoding host as the main controlled endpoint |

**RustDesk's CPU problem only fires when the *controlled* machine is a Linux box with an AMD iGPU.** Conversely, when the controlled side is Windows (NVENC) or Mac (VideoToolbox), RustDesk hardware-encodes fine. This clears up a common confusion:

> "So if I use RustDesk from my MacBook to my Windows desktop, does it also suffer from the AMD problem?" — No. **Encoding always happens inside the *controlled* machine's own client, using that machine's own GPU**; the relay (hbbs/hbbr) does no encoding and touches no GPU — it only helps peers "find each other + forward traffic when NAT traversal fails".

So in this deployment, RustDesk keeps a relay server dedicated to "other devices linking to each other". It's self-hosted and easy to run; here's how.

### Self-hosting a RustDesk relay (hbbs / hbbr)

The relay has two parts: **hbbs** (the ID server: peer discovery, key exchange, NAT punching) and **hbbr** (the relay: forwards traffic as a fallback when punching fails). Official Docker images — one compose:

```yaml
services:
  hbbs:
    container_name: hbbs
    image: rustdesk/rustdesk-server:latest
    command: hbbs -r <relay-addr>:21117    # -r is hbbr's public address, used on punch failure
    volumes:
      - ./data:/root                   # holds id_ed25519 and the sqlite DB
    ports:
      - "21115:21115"                  # NAT probing
      - "21116:21116"                  # TCP: ID registration
      - "21116:21116/udp"              # UDP: punching/registration
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
      - "21117:21117"                  # relay traffic
    restart: unless-stopped
```

After starting, **grab hbbs's public key** (clients must enter it or they can't connect):

```bash
docker compose up -d
docker logs hbbs | grep Key
# Key: l5+...=     ← copy this
# or read it straight from the volume
cat data/id_ed25519.pub
```

On every client that should link to others, configure a **custom server**:

- **ID server**: `<server-addr>:21116`
- **Key**: paste the hbbs public key above

Now those devices find each other through your self-hosted hbbs/hbbr. Bind it to `0.0.0.0` or just the Tailscale address, reachable over the LAN from the previous post — no public exposure needed.

## Pitfalls (all hit for real)

1. **GitHub downloads fail *through* the proxy but succeed *directly*.** Many self-hosted proxies can't reach `github.com` / `objects.githubusercontent.com`, timing out after 15s, while a direct connection works. Rule: try `--noproxy '*'` for GitHub assets first.
2. **Big GitHub files drop mid-transfer (curl error 56).** Half-downloaded resets are common; single-threaded curl fails repeatedly. Use `aria2c -x 16 -s 16` for segmented download:

   ```bash
   http_proxy= https_proxy= all_proxy= aria2c -x 16 -s 16 -k 1M \
     -o sunshine.pkg.tar.zst "https://github.com/.../sunshine-<version>-x86_64.pkg.tar.zst"
   ```

3. **No `sunshine` in Arch repos, and AUR is slow.** `pacman -S sunshine` says not found; AUR metadata sync can take 60s+ from CN. Use the GitHub release `.pkg.tar.zst` with `pacman -U`.
4. **The systemd unit is not `sunshine.service`.** The packaged unit is `app-dev.lizardbyte.app.Sunshine.service`; `systemctl --user enable sunshine` fails with "unit does not exist". Run `systemctl --user daemon-reload` first, then enable the full name (or use the `sunshine.service` alias after reload).
5. **Tailscale gets 403 on the Web UI.** `origin_web_ui_allowed` defaults to `lan`, and Tailscale's CGNAT range counts as WAN. Don't loosen it — use an SSH tunnel.
6. **deny-by-default ACL blocks Sunshine.** Under the previous post's lax-free policy, the six Sunshine port groups must be explicitly allowlisted or the client spins forever without even handshaking.
7. **No audio: `Couldn't connect to pulseaudio: Access denied`.** Sunshine captures audio over PipeWire's pulse-compat layer; if the user service can't reach the pulse socket, there's no sound. Office-style remote control usually doesn't need audio — ignore it, or debug by letting the Sunshine service access your pipewire-pulse socket.
8. **Turn off the compositor.** If the host runs picom (shadows, transparency, animations), it's pure overhead over remote control — there's no physical screen to admire. Comment out `picom -b &` in `~/.config/bspwm/bspwmrc` for a bit more CPU headroom.
9. **Don't nuke PipeWire while uninstalling RustDesk.** On Arch, `pacman -Rns rustdesk` cascades into the PipeWire audio stack that RustDesk depends on — and Sunshine needs it for audio. Check dependencies before removing, or reinstall `pipewire pipewire-audio pipewire-pulse alsa-card-profiles` afterwards.

## Security notes

- Keep Sunshine entirely on Tailscale — **no public ports, no port forwarding**; the host firewall only opens those ports on `tailscale0`.
- In the ACL, allow Sunshine ports only to *your* client IPs; stay minimal.
- Set a strong Web UI password via `sunshine --creds`; access the Web UI only locally or over SSH, don't loosen `origin_web_ui_allowed`.
- If the RustDesk relay only serves your LAN, bind it to the Tailscale address instead of `0.0.0.0`.
- Lock 1080p and keep bitrate under 20 Mbps — cheaper host CPU, less bandwidth, less client decode work.

## Summary

The causal chain is short: **RustDesk can't hardware-encode on AMD → high CPU; Tailscale already solved "how to connect"; Sunshine + Moonlight solves "what encodes"; a DP dummy solves "no screen"**. Put the four together and a dark, monitor-less server in the corner gets a fluid, low-footprint desktop.

Hardware cost is a few-dollar dongle; software cost is zero (all open source, reuse the existing LAN). In return: host CPU drops from dozens of percent (software encoding) to single digits, latency and quality both improve, and one client works across Mac, Windows and phone.

RustDesk doesn't have to leave — keep it as a relay for devices that can't hardware-encode, each doing what it does best. Hope this helps anyone taming a headless host over a wireguard-style LAN.