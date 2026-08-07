+++
title = '从 Tmux 迁移到 Herdr：一个为 AI Agent 而生的现代终端复用器'
date = '2026-08-08T03:30:00+08:00'
draft = false
tags = ['Herdr', 'Tmux', '终端复用器', 'AI Agent', '迁移', '效率工具']
+++

我是一个重度 Tmux 用户。它的快捷键我用得很顺手，但那个 UI 我是真的受不了，自己动手定制也觉得很难受。

直到有人甩给我一个链接：[herdr.dev](https://herdr.dev/)，号称"现代版 Tmux"。

试了一下，直接上头。然后我就把整套 Tmux 键位搬了过去，中途踩了几个坑、也解锁了几个骚操作。这篇博客把完整的迁移方案记下来，万一有人也需要呢？

# Herdr 是什么

Herdr 的官方定位是 **"the runtime coding agents run on"** —— 一个为 AI Agent 而生的终端复用器。注意，它不只是更好看的 Tmux，灵魂是 agent 感知：

- **后台常驻 server 接管终端**：你的所有终端（Claude Code、Codex、Cursor、opencode 等）都在它的后台 server 里。断线、合盖、重启，布局和会话都会自动恢复，随时可以从任何终端甚至 SSH 重连。这不是 Tmux 那种 attach/detach，是它替你把终端"养"着。
- **Agent 状态一目了然**：侧边栏自动识别 ~19 种已知 agent CLI，把每个 pane 标记成 🔴 blocked / 🟡 working / 🟢 idle，谁在等你一清二楚。
- **每个 pane 是真 PTY**：不是包装出来的假终端，全屏 TUI（lazygit、btop）都正常渲染。
- **Agent 之间的自动化**：CLI 和 socket API 是同一套表面，agent 可以互相开 pane、读输出、等彼此。
- **轻量**：~10MB 的 Rust 二进制（Ratatui），没有 Electron、没有账号、没有遥测，AGPL 开源。

安装就一行：

```bash
curl -fsSL https://herdr.dev/install.sh | sh
# 或者 brew install herdr
```

跑 `herdr --default-config` 能打印出完整的默认配置，整个键位系统都写在里面，非常友好。

**概念映射**（最重要的认知切换）：

| Tmux | Herdr |
|---|---|
| session | **workspace** |
| window | **tab** |
| pane | **pane** |

# 为什么 Tmux 玩家迁移起来不难

Tmux 的键位模型是"一个前缀 + 一个键"，Herdr 完全一样（`prefix+n` = 按前缀再按 n）。不像 Zellij 那种模态键位，你需要重新学一套交互。所以：

- **单前缀模型同构**，肌肉记忆平移，90% 的绑定是 1:1 重映射；
- **copy mode 默认就是 vim 风格**（`v` 选、`y` 复制、`/` 搜索、`w/b/e` 跳词），Tmux 里那套 `mode-keys vi` 配置直接不需要了；
- **鼠标是一等公民**：点击、拖拽分割线、右键菜单都原生支持。

# 迁移方案

## 1. 前缀，一行改

我 Tmux 里把前缀设成了 `M-w`（Alt+w），Herdr 同样一行：

```toml
[keys]
prefix = "alt+w"
```

## 2. 坑：分屏的命名是反的 ⚠️

这是最大的坑。Herdr 里：

- `split_vertical`（默认 `prefix+v`）产生的是**左右并排** = Tmux `split-window -h` 的效果；
- `split_horizontal`（默认 `prefix+minus`）产生的是**上下堆叠** = Tmux `split-window -v` 的效果。

命名完全反直觉，但**按效果而不是按名字映射**就没事。我要保留 Tmux 的肌肉记忆（`\` 左右、`-` 上下）：

```toml
split_vertical   = "prefix+\\"
split_horizontal = "prefix+minus"
```

## 3. 骚操作：裸 `\` 居然直接被接受

上面 `"prefix+\\"`（TOML 里转义后就是一个 `\`）原本是我最没底的一行——Herdr 文档里的标点键是"命名标点"（`minus`、`comma`、`ampersand`、`plus`、`backtick`），没提反斜杠。结果 `herdr server reload-config` 返回零诊断，**裸 `\` 直接过了**。这个坑不用踩。

## 4. 坑：`alt+1..8` 范围语法被拒 → 数组大法 ⚠️

我原来在 Tmux 里用 `bind -n M-1..8 select-window` 直接切窗口，想在 Herdr 复刻成 `switch_tab = "alt+1..8"`。结果被拒：

```json
{"diagnostics":["invalid keybinding: keys.switch_tab = \"alt+1..8\"; disabling binding"],"status":"partial"}
```

Herdr 只认官方那种 `prefix+1..9` 全范围，不支持部分范围。但 Herdr 的绑定支持写成**数组**，于是逐条展开：

```toml
switch_tab = ["alt+1", "alt+2", "alt+3", "alt+4", "alt+5", "alt+6", "alt+7", "alt+8"]
```

这次 `status: applied`，零诊断，`alt+1..8` 直接切 tab 成功。而且故意不含 `alt+9`，把 `alt+9` 留给了 workspace 切换。

## 5. 绕开 `( )` 标点键，workspace 用 `alt+9/0`

Tmux 里我用 `prefix+(/)` 切 session，`bind -n M-9/M-0` 切 session。Herdr 里 `( )` 这类标点键名我不确定，直接绕开，用确定的 `alt` 组合：

```toml
previous_workspace = "alt+9"
next_workspace = "alt+0"
```

## 6. floax 浮动窗口 → 原生 popup

我在 Tmux 里用 tmux-floax（`-n M-p`）开浮动终端。Herdr 原生就有 popup 机制，一个配置块搞定：

```toml
[[keys.command]]
key = "alt+p"
type = "popup"
command = "zsh"
width = "80%"
height = "80%"
```

## 7. 分屏沿用当前目录

Tmux 里 `split-window -h -c "#{pane_current_path}"` 让新 pane 继承当前目录。Herdr 一个选项就是默认行为：

```toml
[terminal]
new_cwd = "follow"
```

## 8. 热重载 + 诊断，试错零成本

改完配置不用重启：

```bash
herdr server reload-config
```

返回 JSON，`status: applied` 表示全过，`partial` 表示有绑定被禁用并给出 `diagnostics` 具体原因。写错的键会**安全回退到默认**并给警告，不会炸。再配合 `prefix+?` 查看当前生效的所有键位，验证绑定非常方便。

## 9. 完整配置

最终 `~/.config/herdr/config.toml`：

```toml
[keys]
prefix = "alt+w"

# 重载配置 —— 对应 tmux 的 prefix+r source-file
reload_config = "prefix+r"

# --- tab 切换（herdr 的 tab ≈ tmux 的 window）---
previous_tab = "prefix+h"
next_tab = "prefix+l"
new_tab = "prefix+o"
switch_tab = ["alt+1", "alt+2", "alt+3", "alt+4", "alt+5", "alt+6", "alt+7", "alt+8"]

# --- workspace 切换（herdr 的 workspace ≈ tmux 的 session）---
previous_workspace = "alt+9"
next_workspace = "alt+0"

# --- pane 移动（对应 tmux: bind -n M-h/M-j/M-k/M-l）---
focus_pane_left  = "alt+h"
focus_pane_down  = "alt+j"
focus_pane_up    = "alt+k"
focus_pane_right = "alt+l"

# --- 分屏（命名是反的：split_vertical=左右并排，split_horizontal=上下堆叠）---
split_vertical   = "prefix+\\"
split_horizontal = "prefix+minus"

# --- 其它 ---
close_pane = "alt+q"          # 对应 tmux: bind -n M-q kill-pane
# zoom 默认就是 prefix+z，与 tmux 一致，不用写

# --- floax 浮动窗口 → herdr 原生 popup ---
[[keys.command]]
key = "alt+p"
type = "popup"
command = "zsh"
width = "80%"
height = "80%"

[terminal]
default_shell = "/bin/zsh"
new_cwd = "follow"            # 对应分屏时的 -c "#{pane_current_path}"
```

# 迁移体验总结

- **迁移成本**：90% 的绑定 1:1 平移到 Herdr，剩下 10% 是上面这几个坑。整个迁移我半小时内搞定。
- **比 Tmux 好的地方**：UI 终于不用自己折腾了（内置 catppuccin 等主题，鼠标原生，侧边栏 agent 状态）；持久化是"后台养着"，不是 Tmux 那种"前台挂着"，断线重连无感；对跑多个 AI agent 的场景（Claude Code / Codex 同时开、谁在等你一目了然）是刚需级体验。
- **还缺的地方**：插件生态还很年轻。Tmux 的 tpm、tmux-yank 等没有直接对应物（好在 catppuccin 是内置主题，floax 我用原生 popup 还原了）。如果重度依赖 Tmux 插件，可以再观望。

如果你也是 Tmux 玩家、也在跑 AI agent，值得试试。配好键位的那一刻，你会和我一样喊出那句："我靠，成功了！"
