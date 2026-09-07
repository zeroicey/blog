+++
title = 'Self-hosting an OpenWrt side-router on the Orange Pi Zero 3: three pitfalls and a red-light scare'
date = '2026-09-07T22:40:00+08:00'
draft = false
tags = ['OpenWrt', 'Orange Pi', 'Router', 'Self-Hosted', 'Embedded', 'Tutorial']
description = 'No package feeds on the official snapshot, a kernel panic on immortalWrt, and a red LED that did not mean failure — the full journey of getting OpenWrt running as a side-router on the Orange Pi Zero 3.'
[cover]
  image = 'https://s3.blog.zeroicey.me/covers/orangepi-zero3-openwrt-router.jpg'
+++

Most of my recent posts have been about running services on a proper x86 server. This one goes in a different direction: **installing OpenWrt on a palm-sized Orange Pi and turning it into a "side-router" for my home network**.

There is no shortage of ready-made OpenWrt options for cheap ARM boards. But along the way I ran into three genuine pitfalls — and very nearly got fooled by a red LED into thinking the board was dead. The whole arc felt representative enough to be worth writing down for the next person.

> ⚠️ **Note on examples**: hostnames, LAN IPs, and the auth endpoint in this post are all fictitious; swap in your own values when following along.

## Why the Orange Pi Zero 3

The reasoning is simple: it is cheap, sips power, has a gigabit NIC, and draws so little that leaving it on 24/7 doesn't hurt. Quick specs:

| Item | Value |
| --- | --- |
| SoC | Allwinner H618 (4×Cortex-A53) |
| RAM | 2GB |
| NIC | 1× Gigabit |
| Storage | microSD + 16MB SPI flash |
| Wireless | Onboard (no good mainline driver; not used here) |

A side-router also fits its profile perfectly: **it doesn't replace the main router — it just sits on the LAN and handles the "network edge" chores** like transparent proxying and DNS filtering for a subset of devices. The main router keeps doing dialing and Wi-Fi; each does its own job.

## Pitfall 1: the official snapshot boots, but you can't install anything

The first step was picking an image. The "proper" way is the official firmware selector: pick `xunlong_orangepi-zero3`, download a snapshot image, and write it to the microSD.

Flashing went fine, the board booted, SSH worked — and the moment I tried to install anything, `opkg update` fell over: **not a single feed would update**.

The reason, which took me a while to piece together: OpenWrt's **snapshot images have "rolling" package feeds**. You flash a snapshot from a given day, and the feeds for that day are only kept on the server for a short window before being rotated out. By the time you actually boot and run `opkg update`, that day has already expired, so no packages resolve.

It's a deeply counterintuitive pitfall: **the image itself is fine, yet installing packages is effectively dead**. For day-to-day use, the "no feeds" property of snapshots is enough to scare people off.

## Pitfall 2: immortalWrt panics the moment it boots

Since the official snapshot was a dead end, I tried a popular derivative: **immortalWrt**, which also ships official zero3 images. I flashed it, powered on, and got **only the red light — no green**. By all known signs, the kernel never came up.

After digging, the root cause surfaced: immortalWrt 25.x at the time was on **kernel 6.6**, and on Allwinner's H616/H618 (the sunxi platform), 6.6 triggers a **kernel panic in the `sun6i_spi` interrupt handler while loading the `ntfs3` module**. It's a known upstream issue (search `immortalwrt#1680` for the original oops), and the observable symptom is exactly this: power on → only red → never boots.

The workaround is easy enough: use a 6.1 LTS kernel image, or switch to a branch that has already moved kernels. But it taught me the real lesson: **on sunxi boards, the kernel version matters more than the distro name**.

## The turning point: a "cloud-built" repo

Stuck between these two pitfalls, I found a much more pleasant path: a **"cloud-build" repo on GitHub**.

Repos like this one ([jym66/openWRT-OrangePiZero3](https://github.com/jym66/openWRT-OrangePiZero3)) don't contain the system source themselves. Instead, **GitHub Actions periodically compiles firmware from upstream source and publishes it to Releases**, weekly. The author maintains two lines:

- **Lean lede**: based on coolsnowwolf/lede (the popular "Lean" fork), kernel tracking upstream;
- **immortalWrt**: based on the immortalWrt official branch.

Two things made it exactly what I needed: first, **its lede line was already on kernel 6.12**, sidestepping the 6.6 panic; second, **it targets the zero3 board perfectly** — `sunxi/cortexa53 → xunlong_orangepi-zero3` — no manual target tweaking.

So I downloaded the lede line's `squashfs-sdcard.img.gz`, verified the sha256, `dd`'d it onto the card, and read it back to compare (a lesson from an earlier time when I burned a corrupt boot partition). Powered on, and… hold on, see the next section.

## Pitfall 3: a red-light scare

After plugging it in, I stared at the board and my heart sank: **only the red light again, no green.**

Was it another panic? I nearly yanked the power and re-flashed. Fortunately I paused and probed the default address `192.168.1.1` with `arping` — **and it answered**. SSH in, and the system was fine: kernel 6.12.104, memory detected in full.

The truth: **the Lean lede branch simply doesn't configure a green status LED for this board.** Red is just the power LED and is always on; green stays off because nothing ever turns it into a "status" LED. So "only red" is perfectly normal on this kind of firmware — you can't use the "green = booted" rule that applies to official images.

> This one deserves emphasis because it's so counterintuitive: **to tell whether an Allwinner board actually booted, don't trust the lights — test it with `arping` / `ssh` first.**

## What I configured once it was actually running

After confirming the system was healthy, the rest was shaping it into a proper side-router. Here are the key steps, with commands trimmed to the essentials:

### 1. Static IP — and disable its own DHCP first

The fresh firmware defaults to `192.168.1.1` and ships with a DHCP server enabled. Plug it straight into your LAN and it will **race your main router to hand out IPs** — a "rogue DHCP". So the very first thing is a static IP and turning its DHCP off:

```sh
uci set network.lan.ipaddr='192.168.0.10'      # your subnet
uci set network.lan.gateway='192.168.0.1'
uci set network.lan.dns='192.168.0.1'
uci set dhcp.lan.ignore='1'                     # stop it from racing DHCP
uci commit network && uci commit dhcp
```

### 2. Transparent proxy (Fake-IP)

This firmware comes with proxy plugins like OpenClash / PassWall preinstalled. For a side-router, **Fake-IP (enhanced) mode** is the most pleasant: point clients' gateway and DNS at the board, foreign domains resolve to `198.18.x.x` fake IPs, and the proxy core routes by domain rules — domestic direct, foreign proxied — all transparent with zero client config.

### 3. Taking over DHCP for the whole house (optional but satisfying)

If you want **every device to use the side-router automatically**, let the board take over DHCP: disable DHCP on the main router, re-enable it on the board, and hand out the board itself as both gateway and DNS. Reconnect any device and its traffic flows through the side-router automatically.

> ⚠️ Think the tradeoff through first: once the board owns DHCP, **if it loses power, the whole house loses networking** — so make the power connection solid.

There's a hidden trap here too: OpenWrt's dnsmasq runs a "is there already a DHCP server on this LAN?" check at startup and refuses to serve if it detects the main router still serving. It takes over only after the main router's DHCP is off and the board restarts; to skip the check entirely, add `option force '1'` to `dhcp.lan`.

### 4. AdGuard Home for ad-blocking

`opkg install adguardhome` is all it takes. It's **DNS-level ad-blocking**: it keeps a blocklist of ad domains and resolves them to `0.0.0.0`, so third-party banners and popups in pages simply never load.

The only nuance is wiring it into the proxy's DNS: have AdGuard listen on port 53 and point its upstream at OpenClash's fake-ip DNS — ads get blocked while foreign domains still go through fake-ip, without stepping on each other. Finally, point all devices' DNS at the board.

> A bit of expectation-setting: DNS ad-blocking catches third-party banners/trackers, but **can't block in-stream video ads (e.g. YouTube pre-rolls)** — those need a browser extension.

### 5. Moving campus-network auth onto the board

Our network has one extra requirement: the campus network needs periodic web authentication, and it drops you when the session lapses. It used to be kept alive by a script on another machine; eventually I moved it onto the board — **precisely because it's the one box that's always on**.

The keep-alive itself is simple: a small Go program that periodically checks connectivity and POSTs credentials when it gets redirected to the auth page. Since the board is arm64 + musl while the original binary was x86, I just cross-compiled a static build with `CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build`, dropped it onto the board, and set a per-minute cron. This bit really shows off the appeal of ARM boards: **compile once, run it always-on, without touching your main machine's resources.**

### 6. Monitoring

Finally I installed a monitoring agent (I use [Komari](https://github.com/komari-monitor/komari), a lightweight Go-based monitor) so its CPU / memory / disk / network curves show up on my dashboard. Same story here: cross-compiled an arm64 binary and wrote a procd service for autostart.

## Wrapping up: a few hard-won lessons

After all that, the most valuable takeaways are these, listed plainly:

1. **On Allwinner boards, the kernel version matters more than the distro name.** 6.6 has the `ntfs3`+`sun6i_spi` panic; 6.1 and 6.12 are fine.
2. **The "no feeds" of an official snapshot is by design, not your fault.** For daily use, prefer a release with long-lived feeds or a third-party cloud-build repo.
3. **Don't judge a board's state by its LEDs alone.** lede-family firmware may never light the green LED; a lone red light can still mean it's already up — test with `arping`/`ssh`.
4. **Soft reboot can hang.** On Allwinner boards `reboot` occasionally fails to come back; you may need a physical power cycle, so prefer local reloads for config changes.
5. **Before a side-router takes over DHCP, think through the single point of failure** — secure the power supply.

## Conclusion

From "the official image has no package feeds" to the red-light scare, this little board ended up as a stable network-edge box: transparent proxy, ad-blocking, campus-auth keep-alive, whole-house DHCP, and monitoring — all running 24/7 on a device that draws single-digit watts. The main server keeps doing compute and storage; each does what it's good at.

If you've got an idle board lying around, give this route a try — **a cheap board can genuinely stand in for a proper router**, as long as you get the image and the kernel right.

Related links:

- [jym66/openWRT-OrangePiZero3](https://github.com/jym66/openWRT-OrangePiZero3) — the cloud-build repo used here
- [OpenWrt firmware selector](https://firmware-selector.openwrt.org/) — official wizard
- [coolsnowwolf/lede](https://github.com/coolsnowwolf/lede) — Lean lede source
- [Komari](https://github.com/komari-monitor/komari) — lightweight server monitoring