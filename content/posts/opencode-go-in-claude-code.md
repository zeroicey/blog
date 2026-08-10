+++
title = '把 OpenCode Go 订阅搬进 Claude Code：一套每月 $10 的 DeepSeek 平替方案'
date = '2026-08-10T14:37:00+08:00'
draft = false
tags = ['OpenCode Go', 'Claude Code', 'DeepSeek', 'routatic-proxy', 'AI Agent', '效率工具', '教程']
description = 'DeepSeek 官方 API 按量付费太贵还即将涨价？OpenCode Go 每月 $10 就能拿到 $60 的模型用量。本文用开源项目 routatic-proxy 把这份订阅接进 Claude Code，附完整配置和两个真实的坑。'
+++

我一直用 Claude Code 写代码，模型后端换过好几家。去年换到 DeepSeek 官方 API，结果按量付费那个速度，充 100 块用不了几天就见底——然后某天下午，`claude` 直接给我甩了个 **402 Payment Required**，余额不够了。

再一看 DeepSeek 官方价格，血压上来了：deepseek-v4-flash 输入 ¥1/百万 tokens、输出 ¥2/百万 tokens，pro 更是 ¥3/¥6，而且官方公告写着"**计划近期整体上调 API 定价，涨幅较大**"。

正好那阵子发现 OpenCode Go 的订阅：**首月 $5，之后每月 $10，包含 $60 的模型用量**——相当于 6 倍杠杆。DeepSeek V4 Flash 在里面的单价是 $0.14/$0.28，按典型用量每月能发 **15 万次请求**，堪称量大管饱。

问题是：Claude Code 只认 Anthropic 的 API 格式，OpenCode Go 给的是 OpenAI 格式的端点，两者怎么接？

答案是 [routatic-proxy](https://github.com/routatic/proxy)——一个把 Claude Code 的 Anthropic 请求**透明转换**成上游格式的本地代理。我折腾了一晚上把它跑通了，这篇文章把完整方案写下来，含两个真实的坑。

# 方案思路

```
┌─────────────┐   Anthropic API 格式   ┌──────────────────┐   OpenAI 格式    ┌──────────────────┐
│ Claude Code │ ─────────────────────> │  routatic-proxy  │ ──────────────> │  OpenCode Go     │
└─────────────┘   http://127.0.0.1:3456 │  本地代理 :3456   │  格式转换/路由    │  $10/月，$60 用量 │
                                        └──────────────────┘                  └──────────────────┘
```

Claude Code 以为自己在对 Anthropic 说话，实际上请求被代理改写成 OpenAI 格式、按你配置的路由规则转发给 OpenCode Go，再转回来。全程不需要改 Claude Code 一行配置，只是换了个"API 地址"。

这个项目（923 stars，AGPL-3.0）的核心功能：

- **透明代理**：Anthropic Messages API ↔ OpenAI / Gemini / Responses 格式互转，SSE 流式也支持
- **模型路由**：按场景（默认 / 思考 / 长上下文 / 后台）自动选模型，还支持 sonnet/opus/haiku 家族映射
- **Fallback 链**：模型挂了自动换下一个，还能配电路熔断（circuit breaker）
- **热重载**：改配置文件不用重启
- **GUI 面板**：实时指标、请求历史（macOS 原生窗口，Linux 浏览器）

![routatic-proxy 项目仓库](https://s3.blog.zeroicey.me/20260810144500.png)

# 第一步：安装

macOS 直接 Homebrew（项目是 Go 写的，一条命令）：

```bash
brew tap routatic/tap
brew install routatic-proxy
```

新版 Homebrew 会要求先信任第三方 tap：

```bash
brew trust routatic/tap
```

Linux 用户可以 `brew` 或下载 release 二进制，Windows 走 Scoop。装完验证：

```bash
routatic-proxy --version
# routatic-proxy version 0.6.2
```

# 第二步：初始化配置

```bash
routatic-proxy init
# Created config at ~/.config/routatic-proxy/config.json
```

生成的配置默认路由到 `opencode-go`，模型是 `deepseek-v4-pro`（default 场景）+ `deepseek-v4-flash`（background/fast 场景），带一整套 fallback 链。API key 用环境变量引用：

```json
{
  "api_key": "${ROUTATIC_PROXY_API_KEY}",
  ...
}
```

去 [opencode.ai/auth](https://opencode.ai/auth) 订阅 Go 并复制 API key（`sk-opencode-...`），导出即可：

```bash
export ROUTATIC_PROXY_API_KEY=sk-opencode-你的key
routatic-proxy validate
# Configuration is valid!
```

# 第三步：启动代理

```bash
routatic-proxy serve -b    # 后台运行
routatic-proxy status      # 查看状态
```

代理默认监听 `127.0.0.1:3456`。顺手设置开机自启，省得每次手动起：

```bash
routatic-proxy autostart enable
```

# 第四步：配置 Claude Code

官方推荐的姿势只需要两个环境变量：

```bash
export ANTHROPIC_BASE_URL=http://127.0.0.1:3456
export ANTHROPIC_AUTH_TOKEN=unused
```

`ANTHROPIC_AUTH_TOKEN` 随便填什么，代理不校验。**不要**设 `ANTHROPIC_API_KEY`，否则 Claude Code 会用它去请求 Anthropic。

推荐把变量写进 shell 配置文件持久化。注意你自己的登录 shell 是 bash 就写 `~/.bash_profile`，是 zsh 就写 `~/.zshrc`，别写错文件（别问我怎么知道的）。

# 第五步：真实调用验证

先模拟 Claude Code 发一个 Anthropic 格式的请求：

```bash
curl -s -X POST http://127.0.0.1:3456/v1/messages \
  -H "x-api-key: unused" -H "anthropic-version: 2023-06-01" -H "Content-Type: application/json" \
  -d '{"model":"claude-sonnet-4-20250514","max_tokens":50,"messages":[{"role":"user","content":"Say OK"}]}'
```

再看代理日志确认路由到了预期的模型：

```
INFO received request model=claude-sonnet-4-20250514 streaming=false messages=1 tools=0 max_tokens=50
INFO routing request scenario=override model=deepseek-v4-flash provider=opencode-go tokens=10
INFO model succeeded model=deepseek-v4-flash attempt=1
INFO request completed model=deepseek-v4-flash attempts=1 latency=7.02s
```

然后直接用 Claude Code 跑一条，我用 `claude -p --verbose` 验证，返回的模型就是 DeepSeek V4 Flash，退出码 0：

![Claude Code 通过代理跑通](https://s3.blog.zeroicey.me/20260810143430.png)

# 第六步：按你的需求调路由

`~/.config/routatic-proxy/config.json` 里 `models` 定义了不同场景的模型：

| 场景 | 说明 | 默认模型 |
|---|---|---|
| `default` | 普通对话 | deepseek-v4-pro |
| `fast` | 短小请求 | deepseek-v4-flash |
| `background` | 后台任务 | deepseek-v4-flash |
| `think` | 思考模式 | glm-5.2 |
| `long_context` | >80K 上下文 | minimax-m3 |
| `complex` | 多工具复杂任务 | deepseek-v4-pro |

我把 `default` 和 sonnet/opus/haiku 家族映射都改成了 `deepseek-v4-flash`——Claude Code 发来的请求模型名是 `claude-sonnet-4-...` 这种，会被 `model_family_overrides` 拦截，不改的话实际用的是 kimi/glm 而不是 flash：

```json
"model_family_overrides": {
  "sonnet": { "provider": "opencode-go", "model_id": "deepseek-v4-flash", ... },
  "opus":   { "provider": "opencode-go", "model_id": "deepseek-v4-flash", ... },
  "haiku":  { "provider": "opencode-go", "model_id": "deepseek-v4-flash", ... }
}
```

改完重启代理生效（`hot_reload` 默认关）：

```bash
pkill -f 'routatic-proxy serve' && routatic-proxy serve -b
```

# 坑 1：402 Payment Required —— Claude Code 根本没走代理 ⚠️

这是我踩的最大的坑。代理一切正常、curl 测试全通，但一开 `claude` 就 402。查了半天发现：**`~/.claude/settings.json` 里的 `env` 段优先级高于 shell 环境变量**。

我之前把 `ANTHROPIC_BASE_URL` 指到了 `https://api.deepseek.com/anthropic`（直连 DeepSeek 官方），这个配置留在 settings.json 里，Claude Code 启动时读它、无视我 shell 里 export 的新地址，于是直连官方 API 打余额不足的旧 key → 402。

修复：把 settings.json 的 env 改成指向代理：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:3456",
    "ANTHROPIC_AUTH_TOKEN": "unused"
  }
}
```

# 坑 2：shell 变量写错文件 ⚠️

macOS 上如果 `$SHELL` 是 bash 但 `.zshrc` 里也有一份变量（很多教程都让写 `.zshrc`），bash 登录 shell 不会加载它。按自己的登录 shell 选文件：bash 写 `~/.bash_profile`，zsh 写 `~/.zshrc`，别两份文件混着猜。

# 为什么划算：账算给你看

| 方案 | 成本 | 每月 DeepSeek V4 Flash 用量 |
|---|---|---|
| DeepSeek 官方 API（按量） | flash 输入 ¥1/M、输出 ¥2/M | 充多少用多少，官方预告涨价 |
| OpenCode Go（包月） | $10/月（首月 $5） | **$60 额度** ≈ 15.8 万次请求/月 |

OpenCode Go 的 $10 是"6 倍杠杆"：官方希望你每月用到 $60 的量，这个额度大部分模型都够敞开了造。而且：

- **不止 DeepSeek**：同一个订阅还能用 Kimi K2.6/K3、GLM-5.2、Qwen3.7、MiniMax、GPT-5.6 Luna、Grok 4.5 等，Claude Code 里切模型不用再买别的 key
- **超额兜底**：用超了 Go 额度，代理的 fallback 链会落到 OpenCode Zen 的**免费模型**（nemotron-3-ultra-free 等），不至于直接断
- **模型质量**：DeepSeek V4 Flash 在 Go 里的价格是 $0.14/$0.28，比官方 API 便宜一截，编码场景完全够用

![OpenCode Go 订阅价格与用量](https://s3.blog.zeroicey.me/20260810144510.png)

DeepSeek 官方定价（含"即将涨价"公告）：

![DeepSeek 官方 API 价格](https://s3.blog.zeroicey.me/20260810144520.png)

# 还能再白嫖：模型发现

Claude Code 2.1.129+ 支持从代理自动发现模型，在 `/model` 选择器里显示。开启：

```bash
export CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1
```

小坑：Claude Code 只显示 `claude`/`anthropic` 开头的模型 ID，所以 `deepseek-v4-flash` 不会出现在选择器里，但直接在 `/model` 输入模型名或场景别名（`fast`、`complex`）依然有效。

# 总结

整套方案就三步：**装代理 → 填 key → 指两个环境变量**。成本从"按量付费、说断就断"变成"每月 $10 敞开了用"，还顺手解锁了十几个模型。写这篇的时候我已经用这套跑了好几天，稳定性和直连官方没区别。

唯一要说清楚的：这只是把 OpenCode Go 的**官方订阅**搬了个客户端（Claude Code）来接，走的是官方 API key 和官方端点，没有违反任何条款，订阅本身就是拿来给任何支持 OpenAI/Anthropic 格式的工具用的。

如果你也在被 API 账单按着头打，试试这套方案。跑通的那一刻你会和我一样喊出那句："我靠，成功了！"
