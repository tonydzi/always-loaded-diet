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
| Auto-memory index (`MEMORY.md`) | **Hard-truncated at 200 lines / 25KB** — the tail is silently dropped, no warning, no signal in-session. (Filed upstream as [anthropics/claude-code#79217](https://github.com/anthropics/claude-code/issues/79217)-adjacent behavior; measure it yourself with a canary file.) |
| `CLAUDE.md` | **No truncation observed up to 401,643 bytes** (canary-measured) — it all loads, and you pay tokens for all of it, every session. |

So the two files fail differently: MEMORY.md **loses your rules silently**; CLAUDE.md **taxes every
session and dilutes attention**. Both need a diet, for different reasons.

Our session-start cost measurement, one machine, one week: median **102,180 tokens** at session
start, growing ~20k over 6 days — that growth was almost entirely always-loaded bloat.

## The discipline (what actually works for us)

### 1. One writer per file — sessions report, never compress

The single most important rule. A nightly optimizer (one scheduled job) is the ONLY thing allowed
to restructure/compress an always-loaded file. An ordinary session that notices bloat drops
**one report line** ("⚠️ CLAUDE.md 104KB") and moves on. Never inline-compress someone else's
canon mid-task.

Why: (a) concurrent writers on a synced file = conflict artifacts and lost lines; (b) every LLM
rewrite is lossy — dozens of small "helpful" compressions per week quietly bleed rules; (c) we
measured maintenance-as-reflex before instituting this: **296 backup runs and 237 reindex runs in
7 days, record 52 in a single session** — a once-daily gate killed the parasitic load.

### 2. Budgets with named thresholds

- CLAUDE.md: **yellow at 100KB, red at 120KB**. Grow freely to yellow; past it, the nightly
  optimizer folds detail down. The thresholds live in the guard script AND in the file's own
  header — code mirrors policy, divergence is a bug.
- MEMORY.md: index only — one line per memory, **≤150 chars**, format `- [Title](file.md) — hook`.
  Working zone 60-100 lines; hard ceiling well below the 200-line cut so nothing silently dies.

### 3. Writing a rule is never gated by size

The session that learns a rule MUST write it down even when the file is in the red — a preflight
guard MEASURES and reports, but never blocks the write. Making room is the night optimizer's job,
not the writing session's. A rule lost because "the file was too big" costs more than 2KB of bloat.

### 4. Structure: pointer lines, hubs, INBOX

- **Every line is trigger + gist + pointer.** The body lives one level down (a topic file, a
  docs page); the always-loaded layer holds just enough to know the rule exists and where it lives.
- **Hub pages for crowded domains** — when one domain accumulates 5+ index lines, they fold into
  one hub file with one index line pointing at it (spokes stay findable, index stays lean).
- **INBOX zone, append-only** — new lines land ONLY at the bottom, in a marked INBOX section; the
  nightly job re-sorts them into sections and hubs. Sessions never re-organize the file mid-task.
- **Archive, never delete** — superseded lines move to an archive file (still greppable), and the
  tail of anything being archived is scanned for `pending|BLOCKED|TODO` so unfinished work
  surfaces instead of dying with the line.

### 5. Versioning you can audit

The canon file carries a version line in its header (`v4.38.3 · date`), every edit bumps it and
appends one changelog line (what · why · md5 of the file). When a fleet of machines syncs the
file, md5 in the changelog is what settles "which version is this node actually running".

## Self-diagnosis in 30 seconds

```bash
wc -c ~/.claude/CLAUDE.md; wc -l ~/.claude/**/MEMORY.md
```

CLAUDE.md over ~100KB, or MEMORY.md within sight of 200 lines → you are already paying the tax,
and possibly already losing tail lines. Drop a canary line at the very bottom of MEMORY.md and
check whether the agent can quote it next session.

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

Its closest neighbours in the **memory** layer: [`sqlite-graph-memory`](https://github.com/tonydzi/sqlite-graph-memory) · [`second-brain-starter-kit`](https://github.com/tonydzi/second-brain-starter-kit) · [`voice2brain`](https://github.com/tonydzi/voice2brain)

<!--ecosystem-map:end-->

## AI contributors

This project is built by a human + AI team, and the git log says so: Claude writes most of
the code, Codex and Grok review it, Gemini feeds the research. Each is credited on a commit
**only if its output changed that commit's content** — no decorative credits. Lab-wide
policy, one source for every repo: [AI-CONTRIBUTORS.md](https://github.com/tonydzi/.github/blob/main/AI-CONTRIBUTORS.md).
