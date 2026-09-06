+++
title = 'Your Coding Agents in Your Pocket: Self-Hosting Paseo'
date = '2026-09-06T22:00:00+08:00'
draft = false
tags = ['Paseo', 'AI Agent', 'Tailscale', 'self-hosted', 'Claude Code', 'Codex', 'tutorial']
description = 'Sick of being glued to a terminal while Claude Code or Codex works? Paseo unifies them into one interface, runs the agents on your own machine, and lets you steer them from your phone. Headless setup and three real gotchas included.'
[cover]
  image = 'https://s3.blog.zeroicey.me/covers/paseo-ai-coding-agents.jpg'
+++

The previous posts in this series handled the "infrastructure" side of things: pulling home and cloud machines into one Tailscale network, giving a headless box a remotely operable desktop, assigning each service a port-free HTTPS hostname with Caddy + MagicDNS, plus server monitoring and GitHub acceleration.

But after "reachable, manageable, visible", one thing kept nagging me — **the AI tools I actually write code with were still trapped inside a terminal**.

Claude Code, Codex, OpenCode, Pi... these command-line coding agents have become my main workhorses. They read your repository, edit files, run tests, and carry a task through to completion. But they share one pain point: **once they start, you're chained to that terminal.** Want to add a requirement halfway through? Check progress? Throw away a run and try a different model? Back to the machine, back to the terminal window.

Could I, out and about or on the couch, pull out my phone, see how the agent churning in the background is doing, and toss in an extra "and add the tests too"?

Yes. The answer is an open-source project called [Paseo](https://github.com/getpaseo/paseo).

## What Paseo is

In one line: **Paseo is a self-hosted "control tower" for coding agents. It unifies Claude Code, Codex, GitHub Copilot, OpenCode and Pi behind one interface, keeps the agents running on your own machine, and lets you steer them from your phone, desktop, browser, or the CLI — from anywhere.**

Its selling points happen to line up with this series' whole "self-hosted" creed:

- **Self-hosted**: agents run on *your* machine, with your tools, configs and skills. Data never leaves it.
- **Multi-provider**: Claude Code / Codex / Copilot / OpenCode / Pi in one place, pick the right model per job.
- **Cross-device**: start on the desktop, check in from the phone, script it from the terminal.
- **Privacy-first**: no telemetry, no tracking, no forced account.

## Architecture: one daemon, many clients

The architecture is refreshingly simple. At the core is a background process called the **daemon**, which actually spawns and manages those agent processes; the phone app, desktop app, web UI and CLI are all just "remotes" that connect to it over the network:

```text
  your terminal agents (Claude Code / Codex / Pi ...)
        │  spawned, fed and read back by the daemon
        ▼
  ┌──────────────────────────────┐
  │  core · paseo daemon          │
  │  (listening on 6767, password)│
  └──────────┬───────────────────┘
             │  HTTPS / Tailscale / localhost
     ┌───────┼─────────┬──────────┐
     ▼       ▼         ▼          ▼
   phone   desktop     web UI     CLI
```

The daemon also exposes a WebSocket API, so you can script against it — issue bots, dashboards, orchestration services.

> ⚠️ **Placeholder note**: as in previous posts, hostnames (`core`), internal domains (`homemesh.internal`) and Tailscale IPs (`100.64.12.x`) are made-up examples — substitute your own real values. Paseo itself is a public open-source project; `6767` is its default port and `paseo` its command name — those are public facts.

## Setup: three steps on a headless server

I run it on an always-on "home server" (`core` from this series). For servers and remote machines with no GUI, the project recommends the CLI path instead of the desktop app:

```bash
npm install -g @getpaseo/cli
paseo daemon start
```

But that's just a foreground run — it disappears on reboot. For a long-lived service, hand it to systemd:

```ini
[Unit]
Description=Paseo daemon
After=network-online.target

[Service]
Type=simple
EnvironmentFile=/etc/paseo/paseo.env   # holds PASEO_PASSWORD=...
ExecStart=/usr/local/bin/paseo daemon start --foreground --listen 100.64.12.1:6767 --web-ui
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

Three points: `--foreground` lets systemd own the foreground process; `--listen` binds to the Tailscale interface address, reachable only inside your network, never on the public internet; `--web-ui` turns on the bundled web console. Keep the password in a separate env file, not on the command line.

The one prerequisite: **at least one agent CLI installed and logged in on that machine** — Claude Code or Codex, say. Paseo ships no agents of its own; it's the dispatcher.

## Connecting your phone

Nothing server-side to install on the phone — it's just the regular app. I go over Tailscale directly (this series' familiar route):

1. install Tailscale on the phone and make sure it can `ping` the host;
2. open the Paseo app → **Direct connection**;
3. enter `100.64.12.1:6767` and the daemon password.

Once connected you can see every running agent on the phone, tap into one for a live scrolling terminal, launch new tasks, and append "also add the tests" to an in-flight run. The app also supports voice input — dictate a task while you're out for a walk.

## Three gotchas I actually hit

Setup looks easy, but a few small details tripped me up. All of them took reading source and logs to pin down — sharing so you don't have to:

### 1. The daemon is clearly up, but the CLI won't connect

Symptom: after `paseo daemon start --listen 100.64.12.1:6767`, the log prints `Server listening on http://100.64.12.1:6767`, yet `paseo ls` immediately errors:

```text
Cannot reach the daemon at localhost:6767: Transport closed (code 1006)
```

**Why**: `--listen` only affects *this* run — it never writes back to the persisted `config.json`. So the daemon listens on `100.64.12.1:6767`, while the CLI reads the default `127.0.0.1:6767` from `config.json` and dials `localhost:6767` — nobody home, handshake closed.

**Fix**: put the real listen address into `~/.paseo/config.json` under `daemon.listen`, keeping it consistent with the startup flag.

### 2. Agents won't start under systemd because the node version "drifts"

Symptom: the daemon is up, but launching an agent fails, and `paseo provider diagnostic` reports the agent binary as missing.

**Why**: a systemd *user* unit defaults to `PATH=/usr/local/bin:/usr/bin`. If (like me) you install node via a version manager such as nvm / fnm / asdf, the globally installed `paseo` / `pi` binaries aren't on that PATH; worse, if there's a `/usr/bin/node` around, its version may not match the one your agents were installed against. Two versions fighting.

**Fix**: set `Environment=PATH=...` explicitly in the unit, with your version manager's node dir and the global bin dir first:

```ini
Environment=PATH=/home/you/.local/share/fnm/node-versions/v22/bin:/home/you/.npm-global/bin:/usr/local/bin:/usr/bin
```

Self-check afterwards: `paseo provider diagnostic <provider>` should say `Status: Ready`.

### 3. deny-by-default ACL means clients silently can't connect

Symptom: the phone won't connect no matter what, yet curl from the host itself works fine.

**Why**: in this series' network I use a deny-by-default ACL — personal devices can reach the host on a short whitelist of ports. A new service's port isn't on the list, so it's silently dropped (not a timeout, just "won't connect" — very confusing).

**Fix**: in your ACL policy, add 6767 to the "personal devices → host" port whitelist and apply it; clients reconnect and it just works. Which is a reminder worth nailing down: **every time you add a self-hosted service, besides deploying it and poking the firewall, also open its port in the ACL** — all three, every time.

## Wrapping up

Paseo fills the one gap left in my "self-hosted starter kit": **turning coding agents into a service you can steer on a whim, from anywhere.** Its value isn't a new agent (the agents are still Claude Code, Codex, Pi and friends) — it's gathering the intelligence scattered across a pile of terminal windows into one unified, remote, scriptable interface.

If you, like me, prefer running things on your own machine and hate being chained to a terminal window, it's worth a try. `npm install -g @getpaseo/cli` gets it running in five minutes; pair it with Tailscale and your home server becomes the AI coding workbench in your pocket.