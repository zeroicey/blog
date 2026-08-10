+++
title = 'Bring Your OpenCode Go Subscription into Claude Code: A $10/Month DeepSeek Alternative'
date = '2026-08-10T14:37:00+08:00'
draft = false
tags = ['OpenCode Go', 'Claude Code', 'DeepSeek', 'routatic-proxy', 'AI Agent', 'Productivity', 'Tutorial']
description = 'DeepSeek official API is pay-as-you-go and getting more expensive. OpenCode Go gives you $60 of model usage for $10/month. This post connects that subscription to Claude Code via the open-source routatic-proxy, with a full setup guide and two real pitfalls.'
+++

I've written code with Claude Code for a long time, and I've switched model backends more than once. Last year I moved to the DeepSeek official API — pay-as-you-go, and 100 RMB evaporated within days. Then one afternoon, `claude` slapped me with a **402 Payment Required**. Out of balance.

Then I looked at DeepSeek's official pricing and my blood pressure spiked: deepseek-v4-flash costs ¥1 per million input tokens and ¥2 per million output tokens; pro costs ¥3/¥6. And the official announcement says "**API pricing will be raised significantly in the near future**."

Around the same time I discovered the OpenCode Go subscription: **$5 for the first month, then $10/month, with $60 of included model usage** — a 6x multiplier. DeepSeek V4 Flash costs $0.14/$0.28 per million tokens there, which works out to roughly **150,000 requests per month** at typical usage. Bottomless value.

The catch: Claude Code only speaks the Anthropic API format, while OpenCode Go exposes an OpenAI-compatible endpoint. How do you bridge them?

The answer is [routatic-proxy](https://github.com/routatic/proxy) — a local proxy that **transparently converts** Claude Code's Anthropic requests into whatever format the upstream provider expects. I got it working after an evening of tinkering, and this post documents the full setup, including two real pitfalls.

# How It Works

```
┌─────────────┐   Anthropic API format   ┌──────────────────┐   OpenAI format   ┌──────────────────┐
│ Claude Code │ ──────────────────────> │  routatic-proxy  │ ───────────────> │  OpenCode Go     │
└─────────────┘   http://127.0.0.1:3456  │   local :3456    │  transform/route │  $10/mo, $60 usage│
                                         └──────────────────┘                  └──────────────────┘
```

Claude Code thinks it's talking to Anthropic, but the proxy rewrites each request into OpenAI format, forwards it to OpenCode Go according to your routing rules, and translates the response back. No Claude Code configuration changes needed — you just point it at a different "API endpoint".

The project (923 stars, AGPL-3.0) highlights:

- **Transparent proxy**: Anthropic Messages API ↔ OpenAI / Gemini / Responses format conversion, SSE streaming included
- **Model routing**: automatic model selection per scenario (default / thinking / long-context / background), plus sonnet/opus/haiku family mapping
- **Fallback chains**: automatically try the next model when one fails, with a circuit breaker
- **Hot reload**: config changes without restarting
- **GUI dashboard**: live metrics, request history (native window on macOS, browser on Linux)

![routatic-proxy repository](https://s3.blog.zeroicey.me/20260810143410.png)

# Step 1: Install

Homebrew on macOS (it's a Go CLI):

```bash
brew tap routatic/tap
brew install routatic-proxy
```

Newer Homebrew requires trusting the third-party tap first:

```bash
brew trust routatic/tap
```

Linux users can use `brew` or download a release binary; Windows goes through Scoop. Verify:

```bash
routatic-proxy --version
# routatic-proxy version 0.6.2
```

# Step 2: Initialize Configuration

```bash
routatic-proxy init
# Created config at ~/.config/routatic-proxy/config.json
```

The generated config routes to `opencode-go` by default: `deepseek-v4-pro` for the `default` scenario, `deepseek-v4-flash` for `background`/`fast`, with a full set of fallback chains. The API key is referenced via environment variable:

```json
{
  "api_key": "${ROUTATIC_PROXY_API_KEY}",
  ...
}
```

Subscribe to Go and copy your API key (starts with `sk-opencode-`) from [opencode.ai/auth](https://opencode.ai/auth), then export it:

```bash
export ROUTATIC_PROXY_API_KEY=sk-opencode-your-key
routatic-proxy validate
# Configuration is valid!
```

# Step 3: Start the Proxy

```bash
routatic-proxy serve -b    # run in background
routatic-proxy status      # check status
```

The proxy listens on `127.0.0.1:3456` by default. Enable autostart so you never have to start it manually:

```bash
routatic-proxy autostart enable
```

# Step 4: Configure Claude Code

The official recommendation is just two environment variables:

```bash
export ANTHROPIC_BASE_URL=http://127.0.0.1:3456
export ANTHROPIC_AUTH_TOKEN=unused
```

`ANTHROPIC_AUTH_TOKEN` can be anything — the proxy doesn't validate it. **Do not** set `ANTHROPIC_API_KEY`, or Claude Code will try to reach Anthropic with it.

Persist the variables in your shell profile: write to `~/.bash_profile` if your login shell is bash, or `~/.zshrc` if it's zsh. Don't mix them up (don't ask how I know).

# Step 5: Verify with a Real Call

First, simulate Claude Code with an Anthropic-format request:

```bash
curl -s -X POST http://127.0.0.1:3456/v1/messages \
  -H "x-api-key: unused" -H "anthropic-version: 2023-06-01" -H "Content-Type: application/json" \
  -d '{"model":"claude-sonnet-4-20250514","max_tokens":50,"messages":[{"role":"user","content":"Say OK"}]}'
```

Then check the proxy log to confirm it routed to the expected model:

```
INFO received request model=claude-sonnet-4-20250514 streaming=false messages=1 tools=0 max_tokens=50
INFO routing request scenario=override model=deepseek-v4-flash provider=opencode-go tokens=10
INFO model succeeded model=deepseek-v4-flash attempt=1
INFO request completed model=deepseek-v4-flash attempts=1 latency=7.02s
```

Now run Claude Code for real. I verified with `claude -p --verbose` — the response came back from DeepSeek V4 Flash, exit code 0:

![Claude Code working through the proxy](https://s3.blog.zeroicey.me/20260810143430.png)

# Step 6: Tune the Routing

The `models` section of `~/.config/routatic-proxy/config.json` defines which model handles which scenario:

| Scenario | Purpose | Default model |
|---|---|---|
| `default` | regular conversation | deepseek-v4-pro |
| `fast` | short requests | deepseek-v4-flash |
| `background` | background tasks | deepseek-v4-flash |
| `think` | thinking mode | glm-5.2 |
| `long_context` | >80K context | minimax-m3 |
| `complex` | tool-heavy tasks | deepseek-v4-pro |

I changed `default` and the sonnet/opus/haiku family mappings to `deepseek-v4-flash` — Claude Code sends model names like `claude-sonnet-4-...`, which get intercepted by `model_family_overrides`. Without this change, those requests would use kimi/glm instead of flash:

```json
"model_family_overrides": {
  "sonnet": { "provider": "opencode-go", "model_id": "deepseek-v4-flash", ... },
  "opus":   { "provider": "opencode-go", "model_id": "deepseek-v4-flash", ... },
  "haiku":  { "provider": "opencode-go", "model_id": "deepseek-v4-flash", ... }
}
```

Restart the proxy for changes to take effect (`hot_reload` is off by default):

```bash
pkill -f 'routatic-proxy serve' && routatic-proxy serve -b
```

# Pitfall 1: 402 Payment Required — Claude Code Wasn't Using the Proxy ⚠️

This was the worst one. The proxy was fine, curl tests all passed, but `claude` immediately returned 402. The culprit: **the `env` block in `~/.claude/settings.json` takes precedence over shell environment variables**.

I had previously pointed `ANTHROPIC_BASE_URL` at `https://api.deepseek.com/anthropic` (direct to DeepSeek). That setting lived in settings.json, so Claude Code read it at startup, ignored the new address I exported in the shell, and hit the official API with an old key that had no balance → 402.

Fix: point settings.json's env at the proxy:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:3456",
    "ANTHROPIC_AUTH_TOKEN": "unused"
  }
}
```

# Pitfall 2: Writing Shell Variables to the Wrong File ⚠️

On macOS, if `$SHELL` is bash but you only wrote the variables to `.zshrc` (as many tutorials suggest), a bash login shell won't load them. Pick the file that matches your login shell: `~/.bash_profile` for bash, `~/.zshrc` for zsh.

# Why It's Worth It: The Math

| Option | Cost | Monthly DeepSeek V4 Flash usage |
|---|---|---|
| DeepSeek official API (pay-as-you-go) | flash ¥1/M in, ¥2/M out | whatever you top up, price hike announced |
| OpenCode Go (subscription) | $10/month ($5 first month) | **$60 included** ≈ 158K requests/month |

The $10 buys $60 of usage — a 6x multiplier. And it gets better:

- **More than DeepSeek**: the same subscription covers Kimi K2.6/K3, GLM-5.2, Qwen3.7, MiniMax, GPT-5.6 Luna, Grok 4.5, and more. Switching models in Claude Code costs nothing extra.
- **Over-limit fallback**: when the Go quota is exhausted, the proxy's fallback chain lands on OpenCode Zen's **free models** (nemotron-3-ultra-free etc.) instead of failing outright.
- **Model quality**: DeepSeek V4 Flash costs $0.14/$0.28 per million tokens here, cheaper than the official API, and it handles coding tasks well.

![OpenCode Go pricing and usage](https://s3.blog.zeroicey.me/20260810143400.png)

DeepSeek official pricing (including the "price hike coming soon" notice):

![DeepSeek official API pricing](https://s3.blog.zeroicey.me/20260810143420.png)

# Bonus: Model Discovery

Claude Code 2.1.129+ can auto-discover models from the proxy and show them in the `/model` picker:

```bash
export CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1
```

Minor caveat: Claude Code only displays model IDs starting with `claude`/`anthropic`, so `deepseek-v4-flash` won't appear in the picker — but typing the model name or a scenario alias (`fast`, `complex`) directly in `/model` still works.

# Summary

Three steps: **install the proxy → paste the key → set two environment variables**. The cost model flips from "pay-as-you-go, breaks whenever the balance runs out" to "$10/month and forget about it", with a dozen extra models thrown in. I've been running this setup for days while writing this post — stability is identical to going direct.

To be clear: this just gives Claude Code a new client front-end for your **official OpenCode Go subscription** — it uses the official API key and endpoints, no ToS violations. The subscription is meant to be used by any tool that speaks OpenAI/Anthropic formats.

If the API billing machine is beating you up too, give this a try. The moment it runs, you'll say the same thing I did: "Damn, it works!"
