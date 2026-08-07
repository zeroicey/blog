+++
title = 'Migrating from Tmux to Herdr: A Modern Terminal Multiplexer Built for AI Agents'
date = '2026-08-08T03:30:00+08:00'
draft = false
tags = ['Herdr', 'Tmux', 'Terminal Multiplexer', 'AI Agent', 'Migration', 'Productivity']
+++

I'm a heavy Tmux user. I love its keybindings, but the UI has always been unbearable to me — and customizing it felt even worse.

Then someone handed me a link: [herdr.dev](https://herdr.dev/), advertised as "a modern Tmux."

I tried it and got hooked immediately. I migrated my entire set of Tmux keybindings over, hit a few pitfalls along the way, and unlocked a few neat tricks. This post documents the complete migration playbook in case someone else needs it.

# What is Herdr

Herdr's official positioning is **"the runtime coding agents run on"** — a terminal multiplexer built for AI agents. Note that it's not just a prettier Tmux; its soul is agent-awareness:

- **A background server owns your terminals**: all your agent terminals (Claude Code, Codex, Cursor, opencode, and others) live inside its resident server. Disconnect, close the laptop lid, reboot — your layout and sessions are restored automatically, and you can reattach from anything with a keyboard, including over SSH. This isn't Tmux's attach/detach model; it's Herdr keeping your terminals alive for you.
- **Agent state at a glance**: the sidebar auto-detects ~19 known agent CLIs and marks every pane as 🔴 blocked / 🟡 working / 🟢 idle — you always know who is waiting on you.
- **A real PTY per pane**: not a wrapped fake terminal, so full-screen TUIs (lazygit, btop) render correctly.
- **Agent-to-agent automation**: the CLI and socket API are the same surface, so agents can spawn panes for each other, read output, and wait on each other.
- **Lightweight**: a ~10MB Rust binary (Ratatui), no Electron, no account, no telemetry, AGPL open source.

Installation is one line:

```bash
curl -fsSL https://herdr.dev/install.sh | sh
# or: brew install herdr
```

Running `herdr --default-config` prints the entire default configuration, including the full keybinding system — very friendly.

**Concept mapping** (the most important mental switch):

| Tmux | Herdr |
|---|---|
| session | **workspace** |
| window | **tab** |
| pane | **pane** |

# Why migrating from Tmux is easy

Tmux's keybinding model is "one prefix + one key," and Herdr is identical (`prefix+n` = press the prefix, then `n`). Unlike Zellij's modal approach, you don't have to learn a whole new interaction model. So:

- **Same single-prefix model**, muscle memory transfers, ~90% of bindings are 1:1 remaps;
- **Vim-style copy mode by default** (`v` to select, `y` to copy, `/` to search, `w/b/e` word jumps) — the whole `mode-keys vi` section of your Tmux config just disappears;
- **Mouse is a first-class citizen**: clicking panes, dragging split borders, and right-click menus all work natively.

# The migration playbook

## 1. The prefix, in one line

My Tmux prefix was `M-w` (Alt+w). Herdr reproduces it in one line:

```toml
[keys]
prefix = "alt+w"
```

## 2. Pitfall: split naming is reversed ⚠️

This is the biggest trap. In Herdr:

- `split_vertical` (default `prefix+v`) produces **side-by-side** panes = the effect of Tmux's `split-window -h`;
- `split_horizontal` (default `prefix+minus`) produces **stacked** panes = the effect of Tmux's `split-window -v`.

The naming is completely counterintuitive, but mapping by *effect* rather than by *name* makes it a non-issue. To keep my Tmux muscle memory (`\` = left/right, `-` = top/bottom):

```toml
split_vertical   = "prefix+\\"
split_horizontal = "prefix+minus"
```

## 3. Nice trick: a raw `\` key is accepted

The line `"prefix+\\"` (which is just a single `\` after TOML escaping) was the one I was least confident about — Herdr's docs use "named punctuation" (`minus`, `comma`, `ampersand`, `plus`, `backtick`) and never mention backslash. But `herdr server reload-config` returned zero diagnostics — **the raw `\` just works**. No need to work around this one.

## 4. Pitfall: `alt+1..8` range syntax is rejected → the array trick ⚠️

My Tmux config used `bind -n M-1..8 select-window` to jump directly between windows. Trying to reproduce it as `switch_tab = "alt+1..8"` was rejected:

```json
{"diagnostics":["invalid keybinding: keys.switch_tab = \"alt+1..8\"; disabling binding"],"status":"partial"}
```

Herdr only accepts the full official-style range (`prefix+1..9`); partial ranges are not supported. But Herdr lets you give a binding an **array** of keys — so spell it out:

```toml
switch_tab = ["alt+1", "alt+2", "alt+3", "alt+4", "alt+5", "alt+6", "alt+7", "alt+8"]
```

This time `status: applied`, zero diagnostics — `alt+1..8` now switches tabs directly. Note it deliberately omits `alt+9`, which is reserved for workspace switching.

## 5. Skipping `( )`, using `alt+9/0` for workspaces

My Tmux used `prefix+(/)` to switch sessions and `bind -n M-9/M-0` as direct session shortcuts. In Herdr I wasn't sure whether `( )` are valid key names, so I skipped them and used the reliable `alt` chords instead:

```toml
previous_workspace = "alt+9"
next_workspace = "alt+0"
```

## 6. floax floating window → native popup

I used tmux-floax (`-n M-p`) for a floating terminal in Tmux. Herdr has native popup support — one config block does the job:

```toml
[[keys.command]]
key = "alt+p"
type = "popup"
command = "zsh"
width = "80%"
height = "80%"
```

## 7. Splitting in the current directory

In Tmux, `split-window -h -c "#{pane_current_path}"` makes new panes inherit the current directory. Herdr has it as a single option and it's even the default behavior:

```toml
[terminal]
new_cwd = "follow"
```

## 8. Hot reload + diagnostics: zero-cost trial and error

No restart needed after editing the config:

```bash
herdr server reload-config
```

It returns JSON — `status: applied` means everything passed; `partial` means some binding was disabled, with `diagnostics` explaining exactly why. Invalid keys **fall back to a safe default** with a warning instead of breaking anything. Combined with `prefix+?` to list all active bindings, verifying your setup is painless.

## 9. The complete config

Final `~/.config/herdr/config.toml`:

```toml
[keys]
prefix = "alt+w"

# Reload config — the tmux equivalent of prefix+r source-file
reload_config = "prefix+r"

# --- tab switching (herdr tab ≈ tmux window) ---
previous_tab = "prefix+h"
next_tab = "prefix+l"
new_tab = "prefix+o"
switch_tab = ["alt+1", "alt+2", "alt+3", "alt+4", "alt+5", "alt+6", "alt+7", "alt+8"]

# --- workspace switching (herdr workspace ≈ tmux session) ---
previous_workspace = "alt+9"
next_workspace = "alt+0"

# --- pane movement (tmux: bind -n M-h/M-j/M-k/M-l) ---
focus_pane_left  = "alt+h"
focus_pane_down  = "alt+j"
focus_pane_up    = "alt+k"
focus_pane_right = "alt+l"

# --- split (names are reversed: split_vertical = side-by-side, split_horizontal = stacked) ---
split_vertical   = "prefix+\\"
split_horizontal = "prefix+minus"

# --- misc ---
close_pane = "alt+q"          # tmux: bind -n M-q kill-pane
# zoom is already prefix+z by default, matching tmux — no need to write it

# --- floax floating window → herdr native popup ---
[[keys.command]]
key = "alt+p"
type = "popup"
command = "zsh"
width = "80%"
height = "80%"

[terminal]
default_shell = "/bin/zsh"
new_cwd = "follow"            # tmux: split with -c "#{pane_current_path}"
```

# Migration takeaways

- **Cost**: ~90% of bindings migrate 1:1, the remaining 10% are the pitfalls above. The whole migration took me under half an hour.
- **What's better than Tmux**: no more fighting to customize the UI (built-in catppuccin and other themes, native mouse, agent-state sidebar); persistence means terminals are "kept alive in the background" rather than "left hanging in the foreground," so reconnect after a drop is seamless; and if you run several AI agents at once (multiple Claude Code / Codex sessions, knowing at a glance which one is waiting on you), it's borderline essential.
- **What's still missing**: the plugin ecosystem is young. Tmux's tpm, tmux-yank, and friends have no direct equivalent yet (though catppuccin is a built-in theme, and I replaced floax with a native popup). If you depend heavily on Tmux plugins, you may want to wait.

If you're a Tmux user running AI agents, it's worth a try. The moment your keybindings are in place, you'll say the same thing I did: "Damn, it works!"
