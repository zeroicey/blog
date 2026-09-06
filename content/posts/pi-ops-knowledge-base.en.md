+++
title = 'Building a Self-Documenting AI Ops Knowledge Base with Pi'
date = '2026-09-07T00:20:00+08:00'
draft = false
tags = ['Pi', 'AI Agent', 'DevOps', 'Prompt Engineering', 'Self-hosted', 'Tutorial']
description = 'Stop your AI ops assistant from forgetting context every session, repeating the same pitfalls, and leaving no trace of changes. A three-layer knowledge base on Pi: AGENTS.md static injection + an ops-context dynamic-injection extension + skill SOPs, plus a one-shot scaffolding prompt.'
[cover]
  image = 'https://s3.blog.zeroicey.me/covers/pi-ops-knowledge-base.jpg'
+++

Every post so far has been about *giving machines things*. This one flips the lens — it's about **giving the AI that works for you a memory**.

I've been doing ops through an AI assistant for a while now, and the biggest takeaway isn't about its capability — it's about how **bad its memory is**. Every new session it forgets the last one completely: which services this box runs, how the ports are split, how that weird error got fixed last time, which service needs extra care after a change. I'd have to re-explain everything. Worse, **it steps on the same rake twice** — spend half an hour diagnosing some cryptic failure, and the next session it hits the identical one from scratch.

So I distilled my ops workflow into an *external memory*, baked into a git repo. Now every new session the AI starts already knowing where to look, what rules to follow, and what's still unfinished. This post tears that whole setup apart and shows you exactly how to build it.

## Why Pi, not the generic approach

This entire scheme is **hard-wired to Pi** — if you're not on Pi, most of what follows simply doesn't apply. Pi has a few hooks no other agent gives you, and they're the foundation of this architecture:

1. **Auto-loaded `AGENTS.md`**: drop an `AGENTS.md` in the project root and Pi feeds it in as persistent context on every startup — a piece of permanent system prompt for your AI. This is not the "manually `@file` it" or "stuff it into global config" approach; it's **project-scoped, automatic, zero-touch**.
2. **Project-level `.pi/extensions/*.ts`**: you can put TypeScript extensions inside the repo, and Pi discovers and loads them automatically. They can hook lifecycle events like `before_agent_start` and `session_start`, and register custom slash commands.
3. **Auto-discovered `.pi/skills/*/SKILL.md`**: a markdown file with `name` + `description` front matter is a "skill" — when a task matches the description, Pi pulls its content in automatically. Effectively you can hand your AI a "when you hit this scenario, follow this SOP."

Stack those three hooks together and they map onto my three memory-failure symptoms. I built three layers on top:

| Layer | Carrier | When it fires | What it fixes |
| --- | --- | --- | --- |
| Static injection | `AGENTS.md` | On every Pi startup | "doesn't know where to look or what rules to follow" |
| Dynamic injection | `.pi/extensions/ops-context.ts` | First agent turn of each session | "doesn't know what just happened / what's unfinished" |
| Skill SOPs | `.pi/skills/ops-*` | Triggered by task description | "diagnoses chaotically, leaves no trail after changes" |

## The folder architecture (the core)

The whole thing is just a normal git repo:

```text
ops/
├── AGENTS.md                          # persistent context: system overview + lookup routing + discipline
├── README.md                          # repo overview + directory tree + automation table
├── .gitignore                         # ignore task artifacts / caches
├── specs/                             # living specs (revise in place, keep current)
│   ├── conventions.md                 #   writing conventions (naming / format / append-only)
│   ├── deployment.md                  #   service deployment house standard (docker compose)
│   ├── maintenance.md                 #   monitoring / backup / updates / drift audit
│   └── decisions.md                   #   decision ledger (append-only, traceable choices)
├── runbooks/                          # per-service quick-start runbooks (one file per service, verified)
│   └── <service>.md                    #   port / compose / data / common ops / troubleshooting
├── pitfalls/                          # pitfall archive (four-part: symptom → cause → fix → prevention)
│   ├── _index.md                       #   index: date | title | tags | status | doc
│   └── YYYY-MM-DD-slug.md
├── logs/
│   └── changelog.md                   # change log (append-only, for auditing)
├── inventory/                         # system asset snapshots (measured, never remembered)
│   ├── machine.md                      #   hardware / OS / key paths
│   ├── services.md                     #   services: ports, compose, data locations
│   ├── hosts.md                        #   host ledger: devices + SSH aliases + key mapping
│   └── network.md                      #   three-layer network + node list + proxy chain
├── reports/                           # historical audit reports (read-only archive)
├── scripts/                           # reusable ops scripts
└── templates/
    └── pitfall.md                     # pitfall doc template
```

Each directory owns exactly one job, no overlap. The reasoning:

- **`inventory/` is the "facts layer"** — the current state of the machine, and *measured*, not remembered (`docker ps` / `ss -tlnp` / `systemctl cat`). Docs drift precisely when you "remember" instead of "measure".
- **`specs/` is the "rules layer"** — how to write docs, deploy services, back up and update. Living documents.
- **`runbooks/` is the "operations layer"** — one quick-start manual per service.
- **`pitfalls/` is the "lessons layer"** — every pitfall becomes a four-part note; check `_index.md` first.
- **`logs/changelog.md` is the "time layer"** — append-only, never rewrite history, answering "what changed before".

## Layer one: AGENTS.md (static injection)

The highest value-per-character in the whole system — a tiny amount of text that nails the three main threads: where to look, what rules to obey, what needs confirmation first. It lives in the repo root and loads on startup:

```markdown
# ops —— AI working rules

You are working in the ops repository of the core server. This file is persistent context: any system operation on this machine must follow these rules.

## System overview (quick facts)

- Host: **core** · Arch Linux · 8C16T · 32GB · NVMe with /data on the same disk
- Network: LAN `192.168.0.5` (eno1) · **Tailscale `100.64.12.1` (self-hosted headscale)**
- Proxy: mihomo system service (mixed-port `127.0.0.1:7890`, rule mode)
- Services: Docker Compose first (compose in `/srv/compose/<service>/`, data in `/data/services/<service>/`)
- sudo: passwordless, AI may `sudo -n`; dangerous ops still need impact acknowledged first
- Full fact snapshots: inventory/machine.md · services.md · network.md

## Lookup routing (read before acting)

| Scenario | Read first |
| --- | --- |
| Starting any maintenance/deploy task | `inventory/services.md` for port/dependency status |
| Operating a specific service | its `runbooks/<service>.md` |
| Troubleshooting | `pitfalls/_index.md` → hit the doc → verify by measuring |
| "What changed before" | `logs/changelog.md` |
| Deploy / decommission a service | `specs/deployment.md` |
| Monitoring / backup / updates | `specs/maintenance.md` |

## Maintenance discipline

1. Before acting: check inventory for port/dependency conflicts; measure, don't rely on memory.
2. Always archive afterwards: append changelog, write pitfalls, sync inventory on topology change, revise specs when stale.
3. Honest records: changelog is append-only; pitfall docs cover symptom→cause→fix→prevention.
4. Confirm dangerous ops first: deleting data, taking down shared services, touching proxy/network.
```

> ⚠️ **Substitution note**: consistent with the previous posts, host name (`core`), LAN IP (`192.168.0.5`) and Tailscale IP (`100.64.12.1`) are fictional examples — swap in your real values. Service names are also generalized.

Notice the "lookup routing" table — it's **pointers, not content**. No matter how many runbooks you add later, that table stays a handful of lines, because the AI only needs to know *where to look*, not have everything poured into context. That's the key to controlling context growth here: **the static injection is constant and restrained**.

## Layer two: ops-context.ts (a per-session dynamic brief)

`AGENTS.md` is "always", but it can't cover "what the last session just changed". So I wrote an extension hooked onto Pi's `before_agent_start`: on the first turn of every session it reads the last 12 lines of `logs/changelog.md` plus the "unresolved" pitfalls in `pitfalls/_index.md`, and injects a short brief — invisible to the user — so the AI picks up right where the previous session left off.

```typescript
// .pi/extensions/ops-context.ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { readFile } from "node:fs/promises";
import { join } from "node:path";

const CHANGELOG_TAIL_LINES = 12;
const MAX_BRIEF_CHARS = 2400;

async function readText(path: string): Promise<string | null> {
	try { return await readFile(path, "utf8"); } catch { return null; }
}

function changelogTail(raw: string): string | null {
	const lines = raw.split("\n").map((l) => l.trim()).filter((l) => l.startsWith("- "));
	if (lines.length === 0) return null;
	return lines.slice(-CHANGELOG_TAIL_LINES).join("\n");
}

function openPitfalls(raw: string) {
	const results: { title: string; file: string }[] = [];
	for (const line of raw.split("\n")) {
		if (!line.trim().startsWith("|")) continue;
		const cells = line.split("|").map((c) => c.trim());
		if (cells.length < 6 || cells[1] === "Date") continue;
		if (!(cells[4] ?? "").includes("unresolved")) continue;
		results.push({ title: cells[2] ?? "(untitled)", file: cells[5] ?? "" });
	}
	return results;
}

export default function opsContextExtension(pi: ExtensionAPI) {
	let injected = false;

	const buildBrief = async (cwd: string) => {
		const parts: string[] = [];
		const changelog = await readText(join(cwd, "logs", "changelog.md"));
		if (changelog) {
			const tail = changelogTail(changelog);
			if (tail) parts.push(`[Recent changes]\n${tail}`);
		}
		const index = await readText(join(cwd, "pitfalls", "_index.md"));
		if (index) {
			const open = openPitfalls(index);
			if (open.length > 0) {
				parts.push(`[Unresolved pitfalls ×${open.length}] read these before related work:\n`
					+ open.map((p) => `- ${p.title} ${p.file}`).join("\n"));
			}
		}
		if (parts.length === 0) return "";
		let brief = "[ops-context] Ops knowledge base auto-brief. Discipline in AGENTS.md; archive via ops-record when done.\n\n";
		brief += parts.join("\n\n");
		if (brief.length > MAX_BRIEF_CHARS) brief = brief.slice(0, MAX_BRIEF_CHARS) + "\n…(truncated, read the source for full content)";
		return brief;
	};

	pi.on("before_agent_start", async (_event, ctx) => {
		if (injected) return undefined;
		const brief = await buildBrief(ctx.cwd);
		if (!brief) return undefined;
		injected = true;
		return { message: { customType: "ops-context", content: brief, display: false } };
	});

	pi.on("session_start", () => { injected = false; });

	pi.registerCommand("ops-brief", {
		description: "Show the ops brief and refresh next injection",
		handler: async (_args, ctx) => {
			injected = false;
			ctx.ui.notify((await buildBrief(ctx.cwd)) || "ops brief is empty", "info");
		},
	});
}
```

Three design decisions worth calling out:

1. **`MAX_BRIEF_CHARS = 2400` hard cap** — no matter how big the changelog grows, the injected brief never exceeds this. Reading the *full* history is always on-demand from the source files, never all-at-once into context.
2. **"Unresolved" pitfalls are surfaced separately** — resolved ones don't nag you; only the unfinished ones whisper in your ear every session. Clear them all and this section silently disappears (idle, not erroring).
3. **`/ops-brief` manual refresh** — want the brief mid-session, or force a re-injection next turn? One slash command.

## Layer three: two skills (troubleshooting SOP + archive discipline)

Skills fire "when the scenario matches". Here there are two: one for "how to diagnose", one for "how to close out".

**① ops-troubleshoot (diagnosis SOP)** — the trigger words live in the description, so Pi auto-loads it on "service unreachable / container crash / port down":

```markdown
---
name: ops-troubleshoot
description: System troubleshooting SOP. Use when a service is unreachable, a container exits abnormally, a port doesn't connect, there are error stacks, disk/memory alarms, or network drops.
---

# Troubleshooting SOP

Follow this order strictly. Don't skip steps.

## 0. Check pitfalls first (highest hit rate)
Read pitfalls/_index.md, match tags by symptom, read the hit doc.

## 1. Locate the target
Read inventory/services.md for the service's port, compose dir, data dir, dependencies.

## 2. Measure reality (trust command output, not memory)
docker ps -a --filter name=<name> / docker logs --tail 100 <name>
systemctl status <unit> / journalctl -u <unit> -n 100
ss -tlnp | grep :<port> / df -h / && free -h

## 3. Network issues: layer-by-layer
container → host binding (127.0.0.1) → mesh (Tailscale IP) → LAN (LAN IP)

## 4. Look back at changes
Scan logs/changelog.md recent entries: "what changed before" is often the root cause.

## 5. Archive when fixed
New pitfall → four-part doc; every action → changelog; topology change → inventory.

## Red lines
Shared services need impact acknowledged before restarting; proxy/network changes can cut connectivity — tell the user first.
```

"Check pitfalls first" turned out to be the highest-hit step I didn't expect — over time you realize **most "new problems" are just variants of "old pitfalls"**, and scanning the index beats grepping logs from scratch.

**② ops-record (archive discipline)** — handles closing out. Without it, none of the knowledge base above accumulates:

```markdown
---
name: ops-record
description: Ops close-out archive discipline. Use after any system change, fix, deploy/decommission, or config edit.
---

# Ops archive flow (mandatory when done, no exceptions)

1. Write the changelog: append to logs/changelog.md
   `- YYYY-MM-DD [category] one-line result (@operator)`
   Categories: deploy / fix / maint / pitfall / cron. Append only, never rewrite.

2. New pitfall → new doc: copy templates/pitfall.md to
   pitfalls/YYYY-MM-DD-slug.md, fill four parts (symptom/cause/fix/prevention),
   add a row to _index.md, mark "resolved" when fixed.

3. Topology change → sync inventory/services.md (network.md if needed), refresh the snapshot date.

4. Stale spec → revise specs/.

5. git add -A && git commit -m "<category>: <one-liner>"

Self-check: new changelog entry? new pitfall indexed? inventory matches reality? git committed?
```

## The spec layer: three living documents

Beyond the skeleton + prompts, three specs pin down *how to do things*. Just the essentials here:

- **`conventions.md`**: writing conventions — pitfall filenames as `YYYY-MM-DD-slug.md`, append-only changelog, measured inventory, append-only decisions. **Uniform formatting is what lets the machine parse everything** (the extension literally parses the `_index.md` table to find "unresolved" pitfalls; break the format and the injection breaks).
- **`deployment.md`**: the service deployment house standard — compose in `/srv/compose/<service>/`, data in `/data/services/<service>/`, secrets via `.env` (never inline), ports **double-bound** (`127.0.0.1` + mesh IP), healthcheck required (`healthy` is the acceptance criterion, not an external curl probe).
- **`maintenance.md`**: how to adjust the monitoring baseline, backup principles (back up before touching, land everything in `/data/backups/`), the update flow, and a **"document drift audit"** I added later — periodically take inventory's authoritative facts and grep the whole repo's living docs to catch "a decommissioned technology name still lingering in docs, teaching the AI to do it the old way".

## One-shot scaffolding: don't hand-copy, let Pi build it

You don't have to create any of these files by hand. I've distilled the whole thing into a bootstrap prompt — on a fresh machine, install Pi, create an empty `ops/` directory, enable passwordless sudo, and paste the prompt in. It generates `AGENTS.md`, the extension, both skills, all the specs and templates, then measures the machine to fill in `inventory/` and `runbooks/`, and finally `git init` commits it.

The prompt lives in the [zeroicey/vibe](https://github.com/zeroicey/vibe) repo at `prompts/init-pi-ops.md`:

> 📄 [init-pi-ops.md — Pi ops knowledge base scaffolding prompt](https://github.com/zeroicey/vibe/blob/main/prompts/init-pi-ops.md)

Its flow is "**scaffold from templates first → measure and fill facts → force a final self-check**", so what comes out is never a hollow shell. Your only prep is the opening line: Pi installed, `ops/` dir created, sudo passwordless.

## On keeping secrets

This system is fundamentally "write down every fact about your server", so it needs the same isolation as code:

- Keep the repo in private git (or local only) — **don't push to a public repo**; it's full of real ports, IPs, service names.
- When sharing publicly like this very post, **sanitize**: fictional host name (`core`), fictional IP range (`100.64.12.x`), generic service names ("an API gateway", not the real one).
- Never keep secrets in plaintext in the knowledge base — record "which env file holds the key", never the value.

## Closing

Once this is running, ops settles into a comfortable rhythm: a new session starts with the AI already knowing where it left off; a problem starts with the pitfall index, so old pits never bite twice; every change is honestly archived into changelog and pitfalls. **You make the decisions; it remembers, looks up, and follows the rules** — which is exactly the division of labor "AI ops" should have.

Tie it back to the earlier service series: this workflow governs **"what changed and how to reproduce it"**, complementing the "infrastructure" pieces — Tailscale mesh, Caddy domains, Komari monitoring. One is the machine's skeleton, the other is the AI's memory.

Related links:

- [zeroicey/vibe](https://github.com/zeroicey/vibe) — my vibe coding resource collection, including this post's scaffolding prompt
- [Pi coding agent](https://github.com/earendil-works/pi-coding-agent) — the host for project-level extensions / skills / AGENTS.md