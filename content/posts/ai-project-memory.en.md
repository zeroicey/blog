+++
title = 'Giving AI Coding Assistants "Project Memory": A Self-Iterating Design'
date = '2026-08-23T21:15:00+08:00'
draft = false
tags = ['AI Agent', 'pi', 'Project Memory', 'Prompt Engineering', 'ADR', 'Workflow']
description = 'A "project memory" system running in three real projects: a three-layer memory model plus a 200-line extension for automatic injection, with the write path still driven by convention. This is a living document — how it works today, where it has already broken down, and what comes next.'

[cover]
  image = 'https://s3.blog.zeroicey.me/covers/ai-project-memory.jpg'
+++

If you have ever had an AI assistant maintain a project for more than a week, you know the problem: **every new session starts from amnesia**.

It doesn't know the architecture decision you settled last week. It steps on the same deployment pitfall you hit two days ago. Which server directories must never be touched — you end up reminding it verbally, every single time. A bigger context window doesn't help: when the session ends, everything resets to zero.

I built a homegrown "project memory" system for three real projects (a multi-app workbench, a personal journaling service, and a people registry). Nothing mystical about it: at its core it's just **markdown files plus a 200-line pi extension**. Over the past few weeks it has taken part in dozens of production deployments and one hair-raising server config rescue — and it has exposed its own weaknesses along the way.

This post documents its current state — both the parts that work and the parts that have fallen over. **This is a living document**: the system keeps evolving, and I'll revise this post as it does. What you're reading is always the current snapshot.

# The Big Picture: A Three-Layer Memory Model

The design borrows from cache layering, splitting memory by access frequency and cost:

| Layer | Carrier | Nature | How it enters context |
| --- | --- | --- | --- |
| L1 Permanent rules | `.pi/APPEND_SYSTEM.md` | Iron laws, invariants, known pitfalls | Static injection, every session |
| L2 Recent memory | Digests of `.ai/` | What happened lately | Injected per turn by an extension, ≤1.8KB |
| L3 Full retrieval | Full `.ai/` text + event store | Details and history | On-demand `read` / `ctx_search` |

On disk this maps to two directories:

```
project/
├── .pi/                      # agent config
│   ├── APPEND_SYSTEM.md      # L1: permanent rules (incl. known-pitfall section)
│   ├── extensions/
│   │   └── memory.ts         # L2: auto-injection extension
│   └── skills/
│       └── remember-*/       # write-path memory skills
└── .ai/                      # the project memory itself
    ├── README.md             # directory roles + rules + index
    ├── worklog/              # daily log (authoritative history)
    ├── decisions/            # ADRs
    ├── requirements/         # requirement docs (with status lines)
    ├── architecture/         # currently valid architecture docs
    ├── runbooks/             # standard procedures (single source of truth)
    ├── archive/              # dead docs
    └── inbox/                # staging area for raw fragments
```

# The Read Path: Auto-Injection via One Extension

L2 is the engine of the whole scheme. `.pi/extensions/memory.ts` hooks into pi's `before_agent_start` event and appends a recent-memory digest to the system prompt **before every turn**:

```typescript
// simplified core logic
pi.on("before_agent_start", (event) => {
  const digest = buildDigest(root); // ≤1.8KB
  return { systemPrompt: `${event.systemPrompt}\n\n${digest}` };
});
```

The digest contains three things:

- **Titles of the 4 most recent worklog files** (headings only, no body text)
- **Titles of the 5 latest decisions**
- **List of undigested inbox fragments**

Two deliberate design decisions:

**First, restrained injection.** Capped at 1.8KB, headings only. When the model sees a title like "2026-08-23 Admin full re-reskin + disk-full and .env rescue incident", it can judge relevance itself and read the original file if needed. Pouring full text into context would blow up the window within days.

**Second, mtime fingerprint caching.** If new memory is written mid-session, the digest refreshes on the next turn — computed by comparing file mtime lists, recalculating only when something changed. So a worklog written during wrap-up becomes visible to later turns in the same session.

The net effect: from the very first turn of a new session, the model already knows what's been going on and what was recently decided. No more reciting background by hand.

# The Write Path: Skills + Wrap-Up Discipline

Reading is automatic; writing still relies mostly on convention. I created four remember-* skills:

| Skill | Responsibility |
| --- | --- |
| `remember-worklog` | Substantial session work → daily worklog |
| `remember-decision` | Why-level choices (including rejected ones) → ADR |
| `remember-runbook` | Reproducible procedures → runbook, one-line pointer left in worklog |
| `remember-requirement` | Requirement status changes → status table |

Plus a few distillation rules:

- **Dedupe before writing**: same topic exists → update, don't create
- **Mark pitfalls ⚠️; escalate on second strike**: when the same pit shows up twice, distill it into the "Known Pitfalls" section of APPEND_SYSTEM — that's L1, statically injected, seen by every future session
- **Procedures live only in runbooks**: reproducible flows get distilled into runbooks; the worklog keeps a one-line pointer

There's one more safety net underneath: the context-mode extension automatically captures event-level memory (decisions, errors, blockers), searchable via `ctx_search`. The division of labor: **the event store is cheap raw material; `.ai/` is the curated, authoritative version**.

What this looks like in practice — two real examples:

**Example 1**: During a production deployment incident, cleaning up old releases took down the `.env` that every release symlinked to — one service restart away from losing all production config. The rescue (dumping environment variables from `/proc/<pid>/environ` of the live process) went into the day's worklog in full, and the core lesson got distilled into one line: "/proc/<pid>/environ is the last resort for rescuing a running service's config". Next time something similar happens, the model finds a ready-made playbook.

**Example 2**: An ADR about a storage architecture change (moving deletes through signed gateway URLs) recorded not only the adopted approach but two rejected alternatives and why. Two weeks later another module hit a similar problem, and the model cited that ADR's conclusion directly instead of re-walking the dead ends.

# Field Results

The three projects see different usage intensity, which makes a natural comparison:

| Project | Shape | Memory size | Notes |
| --- | --- | --- | --- |
| koma | Multi-app workbench + dual-server ops | 68 worklogs / 604K | Highest intensity, multiple worklogs per day |
| serenique | Full-stack journaling service (4 clients) | 95 worklogs / 1.5M | Memory in git; ~35% of commits touch `.ai/` |
| registry | Bun + React registry | 5 worklogs / 96K | Slim variant: decisions as one file, no archive |

That 35% for serenique deserves a highlight: 176 out of 498 commits over three weeks touched `.ai/`. That ratio tells you memory maintenance isn't decorative — it has become part of the workflow. The cost of that comes later.

# Problems Already Exposed: An Honest List

A few weeks in, these problems were paid for in real hours. This may be the most valuable part of the whole post.

## Problem 1: The write path relies on self-discipline — the biggest hole

The read path is automated; the write path runs on skill descriptions and one line of convention ("wrap-up actions"). Consequences:

- If the model forgets to wrap up, or judges "this wasn't substantial work" → memory silently lost
- You cannot distinguish "nothing worth recording" from "forgot to record"
- No verification of written content — long-term hallucination drift risk

Evidence close at hand: koma's inbox holds a fragment from August 16 that sat undigested for a week, while the convention says "digest then empty".

## Problem 2: "Escalate on second strike" has a circular dependency

Escalating a pitfall to L1 requires recognizing it's the second occurrence. But recognizing the second requires remembering the first — and the first is buried in some worklog file that never gets injected. In other words, **the entry point of this distillation pipeline sits on its least reliable component**. Many pitfalls will just sit in worklogs forever, waiting for a second strike that detection never catches.

## Problem 3: Injection carries titles, not lessons

What changes behavior is concrete instructions ("⚠️ pit + workaround"), but L2 injects titles only. Titles are signposts; they depend on the model choosing to read deeper — and models lack motivation to deep-read historical titles that look unrelated. Also, injection takes newest-first: a critical month-old decision slides out of the window entirely, however relevant it is to the current task.

## Problem 4: One fact stored in five places, consistency maintained manually

One pitfall can simultaneously live in worklog body, README index, ADR, APPEND_SYSTEM pitfalls section, and a runbook. Rules like "update, don't create" are pure discipline with zero mechanism behind them. The most visible symptom: **koma's README has bloated to 20KB** — its "recent work" chain is effectively a secondary summary of worklogs, heavily duplicated, a secretly grown fourth memory layer with no cap and no expiry.

## Problem 5: Three hand-maintained forks drifting apart

memory.ts exists in three copies, already diverging (koma's handles multiple worklogs per day; registry's sorts by mtime). Bug fixes don't propagate between them; in six months they'll be three dialects impossible to merge.

## Problem 6: Durability strategy is a mess

koma's root isn't a git repository — 604K of operational memory sits on a single disk with zero version control. registry gitignores `.ai` — no backup, no history. Only serenique commits it. **For a memory system to treat its own durability this casually is indefensible.**

# Fix Underway: The Write-Path Guard

For problem 1, the first mechanized fix just shipped in koma (today, actually). The idea: make "the moment it should be written" a mechanism:

```
tool_call hook              agent_settled                  forced wrap-up turn
┌─────────────────┐    ┌──────────────────────┐    ┌──────────────────────────┐
│ track edit/write │ →  │ ≥3 code edits?        │ →  │ sendMessage(triggerTurn) │
│ touched .ai/ ⇒   │    │ today's worklog       │    │ inject reminder; model   │
│ already wrapped  │    │ updated? neither ⇒    │    │ must respond: write, or  │
│                  │    │ time to nudge         │    │ explain why not          │
└─────────────────┘    └──────────────────────┘    └──────────────────────────┘
```

Key trade-offs:

- **`agent_settled`, not a session-exit hook**: at exit the process is dying and can't trigger a response — it would degrade into next-session catch-up
- **Force a turn, don't just show a UI notification**: free notifications can't close the loop. Forcing a turn makes "write or not write" an explicit model judgment — even answering "nothing worth recording" is a judgment, not silent loss
- **Never auto-write content**: the guard only guarantees the action gets evaluated; content still comes from the model following templates. Auto-writing would deposit hallucinations — worse than not writing
- Three anti-nag gates: once per session / edit-count threshold / `/memory-guard-skip` opt-out

This guard is brand new and uncalibrated (is the threshold right? what's the false-positive rate?). **After observing 3–5 real sessions**, I'll decide whether to port it to the other two projects.

# Roadmap

In priority order — also the agenda for future updates of this post:

1. **Worklog compaction** — 95 files growing linearly will degrade retrieval quality within a year. Plan: periodic distillation jobs compressing old worklogs into "verified lessons" pages, originals archived
2. **Auto-generated README index** — generated by the extension from filenames + titles, eliminating the manual triple bookkeeping of problem 4
3. **Guard rollout + fork convergence** — once validated, port memory-guard and merge the three memory.ts copies into one shared parameterized extension
4. **Unified durability policy** — commit `.ai/` everywhere (minus secrets), scheduled backups for koma
5. **Subtraction: inbox & requirements** — inbox is largely replaced by context-mode; requirements-as-directory feels heavy for solo projects, evaluate folding status lines into ADR headers

# Takeaways Worth Stealing Today

If you remember only three things:

1. **Layering works**: iron laws static-injected, recent memory injected as digests, everything else retrieved on demand — restrained injection is the foundation. Don't start by stuffing full memory into context.
2. **The read path fully automates with a 200-line extension.** Mechanizing the write path is the hard part — and the current frontier.
3. **Treat the memory system itself as software**: it has bugs, accumulates entropy, needs compaction. It's part of your project, not a plug-and-play add-on.

As for the unfixed problems — README bloat, fork divergence, the second-strike circular dependency — they're sitting right there in plain text, an honest snapshot of the system as it stands. When fixes land, the relevant sections of this post get rewritten, not deleted: a blog post about project memory ought to update itself the way project memory does.
