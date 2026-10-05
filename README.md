# always-loaded-diet

**A discipline for the files your agent loads into EVERY session (CLAUDE.md, MEMORY.md, rules files): one nightly writer, measured budgets, pointer-only lines — because every kilobyte you add there is rent you pay on every future session, and because one of these files silently truncates.**

## The pain, in your words

- *"My CLAUDE.md hit 100KB and the agent started ignoring half of it."*
- *"Rules I wrote last month just… stopped existing. Turns out the file gets cut."*
- *"Every session 'helpfully' rewrites and compresses my memory file, and every rewrite loses something."*
- *"Session start costs 100k tokens before any work happens, and it grows every week."*

## The two measurements everything follows from

| File | Measured behavior |
|---|---|
| Auto-memory index (`MEMORY.md`) | **Hard-truncated at 200 lines / 25,000 UTF-16 units** — the tail is silently dropped, no warning, no signal in-session. (Filed upstream as [anthropics/claude-code#79217](https://github.com/anthropics/claude-code/issues/79217)-adjacent behavior; measure it yourself with a canary file.) |
| `CLAUDE.md` | **No truncation observed up to 401,643 bytes** (canary-measured) — it all loads, and you pay tokens for all of it, every session. |

> **The size cap is UTF-16 units, not bytes.** Corrected 2026-08-26 after
> [@DanceNitra caught it](https://github.com/anthropics/claude-code/issues/82056);
> this page said "25KB" before. It only matters for non-ASCII, and it matters a lot:
>
> | script | UTF-8 bytes | UTF-16 units | bytes per unit |
> |---|---|---|---|
> | ASCII | 1 | 1 | 1.00 |
> | Cyrillic / Latin-1 accented | 2 | 1 | 2.00 |
> | CJK (BMP) | 3 | 1 | 3.00 |
> | emoji (astral) | 4 | 2 | 2.00 |
>
> So budgeting a Cyrillic index in bytes stops you at ~12,500 units — **half** the
> headroom you actually have. A CJK index is pruned 3x harder than it needs to be.
> The 200-line cap is exact either way, and for an all-ASCII index bytes and units
> coincide, which is why this went unnoticed here.

So the two files fail differently: MEMORY.md **loses your rules silently**; CLAUDE.md **taxes every session and dilutes attention**, and [RULES.md](RULES.md) treats each with its own diet. Both need a diet, for different reasons.

Our session-start cost measurement in 2026, one machine, one week: median **102,180 tokens** at session start, growing ~20k over 6 days — that growth was almost entirely always-loaded bloat.

## The discipline (what actually works for us)

### 1. One writer per file — sessions report, never compress

The single most important rule. A nightly optimizer (one scheduled job) is the ONLY thing [RULES.md](RULES.md) allows to restructure or compress an always-loaded file. An ordinary session that notices bloat drops
**one report line** ("⚠️ CLAUDE.md 104KB") and moves on. Never inline-compress someone else's
canon mid-task.

Why: (a) concurrent writers on a synced file = conflict artifacts and lost lines; (b) every LLM
rewrite is lossy — dozens of small "helpful" compressions per week quietly bleed rules; (c) we
measured maintenance-as-reflex before instituting this: **296 backup runs and 237 reindex runs in 7 days of 2026, record 52 in a single session** — a once-daily gate killed the parasitic load.

### 2. Budgets with named thresholds

- CLAUDE.md: **yellow at 100KB, red at 120KB**. Grow freely to yellow; past it, the nightly
  optimizer folds detail down. The thresholds live in the guard script, in the file's own header and in [RULES.md](RULES.md) — code mirrors policy, divergence is a bug.
- MEMORY.md: index only — one line per memory, **≤150 chars**, format `- [Title](file.md) — hook`.
  Working zone 60-100 lines; hard ceiling well below the 200-line cut so nothing silently dies, as [RULES.md](RULES.md) spells out.

### 3. Writing a rule is never gated by size

The session that learns a rule MUST write it down even when the file is in the red — [RULES.md](RULES.md) makes the preflight guard MEASURE and report, but never block the write. Making room is the night optimizer's job,
not the writing session's. A rule lost because "the file was too big" costs more than 2KB of bloat.

### 4. Structure: pointer lines, hubs, INBOX

- **Every line is trigger + gist + pointer**, the shape [RULES.md](RULES.md) requires. The body lives one level down (a topic file, a
  docs page); the always-loaded layer holds just enough to know the rule exists and where it lives.
- **Hub pages for crowded domains**, per [RULES.md](RULES.md) — when one domain accumulates 5+ index lines, they fold into
  one hub file with one index line pointing at it (spokes stay findable, index stays lean).
- **INBOX zone, append-only** — [RULES.md](RULES.md) lets new lines land ONLY at the bottom, in a marked INBOX section; the
  nightly job re-sorts them into sections and hubs. Sessions never re-organize the file mid-task.
- **Archive, never delete** — under [RULES.md](RULES.md) superseded lines move to an archive file (still greppable), and the
  tail of anything being archived is scanned for `pending|BLOCKED|TODO` so unfinished work
  surfaces instead of dying with the line.

### 5. Versioning you can audit

The canon file carries a version line in its header (`v4.38.3 · date`), every edit bumps it and
appends one changelog line (what · why · md5 of the file). When a fleet of machines syncs the file, md5 in the changelog is what settles "which version is this node actually running" — the versioning contract is in [RULES.md](RULES.md).

## Self-diagnosis in 30 seconds

```bash
wc -c ~/.claude/CLAUDE.md; wc -l ~/.claude/**/MEMORY.md
```

CLAUDE.md over ~100KB, or MEMORY.md within sight of 200 lines → you are already paying the tax,
and possibly already losing tail lines. Drop a canary line at the very bottom of MEMORY.md and check whether the agent can quote it next session; [RULES.md](RULES.md) calls this the only honest test of the cut.

## What ships here

- [RULES.md](RULES.md) — the discipline above as a drop-in rules file for your agent.
- This README — the measurements and the why.

## FAQ

**Why not just keep the file small by hand?** You will lose. Rules arrive faster than you prune.
The system needs a place where writes are cheap (INBOX, append-only) and a separate scheduled
process that pays the organizing cost once per day.

**Why once per NIGHT, not on every write?** Because the optimizer is lossy and needs review-grade
care; running it 50 times a day multiplies both the token cost and the loss probability. Once a
day, on the whole file, with fresh context, beats 50 micro-compressions.

**Isn't archiving clutter?** The archive is not loaded — it's greppable history. Deleting is how
you find out three weeks later that the deleted line was load-bearing.

**Does this apply beyond Claude Code?** The mechanics (silent truncation limits, always-loaded
token rent) are host-specific; the discipline (one writer, pointer lines, append-only inbox,
measure-not-block) ports to any agent with persistent instruction files.

## Attribution & license

Invented by **Mycroft** (synthetic cofounder) & **Tony** — [Palo Alto AI Research Lab](https://github.com/tonydzi). MIT license.

Siblings: [compact-canon](https://github.com/tonydzi/compact-canon) (what survives `/compact`, measured) · [claw-retro](https://github.com/tonydzi/claw-retro) (the session-close ritual that routes durable rules here) · [break-it-first](https://github.com/tonydzi/break-it-first) (the post-build quality gate).

We hand free working seeds of our lab tooling to engineer-testers — WhatsApp **+1 (341) 222-9178**.

---

<!--ecosystem-map:start-->

## 🧩 One piece of a working system

This repository is one piece lifted out of a live operation: one non-technical founder, an AI
cofounder, and a fleet of machines that reach consensus with each other and wake the human only
for money or the irreversible. It was extracted after it survived production, not written as a
demo — and it runs on its own: nothing here phones home to the rest.

**See how the whole thing fits together → [SYSTEM.md](https://github.com/tonydzi/tonydzi/blob/main/SYSTEM.md)**

<!--ecosystem-map:end-->

## AI contributors

This project is built by a human + AI team, and the git log says so under the rules in [AI-CONTRIBUTORS.md](https://github.com/tonydzi/.github/blob/main/AI-CONTRIBUTORS.md): Claude writes most of the code, Codex and Grok review it, Gemini feeds the research. Each is credited on a commit
**only if its output changed that commit's content** — no decorative credits. Lab-wide
policy, one source for every repo: [AI-CONTRIBUTORS.md](https://github.com/tonydzi/.github/blob/main/AI-CONTRIBUTORS.md).

<!-- READ-WITH-AI:START (generated by read_with_ai.py - do not hand-edit) -->

### READ THIS WITH AI

One click and an agent reads the repo, pulls out the patterns and helps you apply them to your own work.

<a href="https://chatgpt.com/codex?prompt=Read%20this%20repo%3A%20https%3A%2F%2Fgithub.com%2Ftonydzi%2Falways-loaded-diet%20%28%E2%80%9Calways-loaded-diet%E2%80%9D%20-%20Discipline%20for%20the%20files%20your%20agent%20loads%20every%20session%20%28CLAUDE.md%2C%20MEMORY.md%29%3A%20one%20nightly%20writer%2C%20measured%20budgets%2C%20pointer-only%20lines%29.%20Work%20out%20what%20problem%20it%20actually%20solves%2C%20pull%20out%20the%20reusable%20patterns%20and%20help%20me%20apply%20them%20to%20my%20own%20setup.%20Start%20by%20asking%20what%20I%20am%20working%20on."><img alt="Codex - open" src="https://img.shields.io/badge/Codex-open-000000?style=for-the-badge&logo=openai&logoColor=white"></a> <a href="https://chatgpt.com/?q=Read%20this%20repo%3A%20https%3A%2F%2Fgithub.com%2Ftonydzi%2Falways-loaded-diet%20%28%E2%80%9Calways-loaded-diet%E2%80%9D%20-%20Discipline%20for%20the%20files%20your%20agent%20loads%20every%20session%20%28CLAUDE.md%2C%20MEMORY.md%29%3A%20one%20nightly%20writer%2C%20measured%20budgets%2C%20pointer-only%20lines%29.%20Work%20out%20what%20problem%20it%20actually%20solves%2C%20pull%20out%20the%20reusable%20patterns%20and%20help%20me%20apply%20them%20to%20my%20own%20setup.%20Start%20by%20asking%20what%20I%20am%20working%20on."><img alt="ChatGPT - open" src="https://img.shields.io/badge/ChatGPT-open-10a37f?style=for-the-badge&logo=openai&logoColor=white"></a> <a href="https://claude.ai/new?q=Read%20this%20repo%3A%20https%3A%2F%2Fgithub.com%2Ftonydzi%2Falways-loaded-diet%20%28%E2%80%9Calways-loaded-diet%E2%80%9D%20-%20Discipline%20for%20the%20files%20your%20agent%20loads%20every%20session%20%28CLAUDE.md%2C%20MEMORY.md%29%3A%20one%20nightly%20writer%2C%20measured%20budgets%2C%20pointer-only%20lines%29.%20Work%20out%20what%20problem%20it%20actually%20solves%2C%20pull%20out%20the%20reusable%20patterns%20and%20help%20me%20apply%20them%20to%20my%20own%20setup.%20Start%20by%20asking%20what%20I%20am%20working%20on."><img alt="Claude - open" src="https://img.shields.io/badge/Claude-open-d97757?style=for-the-badge&logo=anthropic&logoColor=white"></a>

<details>
<summary>Copy the prompt (works in any agent: Gemini, Grok, a local model, your own CLI)</summary>

```text
Read this repo: https://github.com/tonydzi/always-loaded-diet (“always-loaded-diet” - Discipline for the files your agent loads every session (CLAUDE.md, MEMORY.md): one nightly writer, measured budgets, pointer-only lines). Work out what problem it actually solves, pull out the reusable patterns and help me apply them to my own setup. Start by asking what I am working on.
```

</details>

<sub>— TonyDzi, Palo Alto AI Research Lab · second brain, agent coordination, persistent memory: github.com/tonydzi</sub>

<!-- READ-WITH-AI:END -->
