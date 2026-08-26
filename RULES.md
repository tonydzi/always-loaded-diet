# Always-loaded diet — drop-in rules

Copy the relevant blocks into your agent's rules file (CLAUDE.md / AGENTS.md). Adjust paths and
thresholds to your host; the numbers below are the ones we run.

## For every session (the cheap half)

```markdown
## Always-loaded files: report, never compress
- CLAUDE.md / MEMORY.md / rules files are optimized ONLY by the nightly optimizer job.
  If this session notices bloat: drop ONE line — "⚠️ <file> <size>KB" — and continue the task.
  Compressing or reorganizing someone else's canon mid-task is forbidden.
- Writing a NEW rule is never gated by size: measure (preflight), report the size, write anyway.
  Making room is the optimizer's job, not yours.
- New index lines go ONLY to the "## INBOX" section at the bottom, append-only,
  one line ≤150 chars: `- [Title](file.md) — hook`. The nightly job re-sorts.
- Maintenance actions (backup, reindex, guard sweeps) run at most once per day per node;
  a gate script blocks the second run. Diagnostics of a live incident are never gated.
```

## For the nightly optimizer (the careful half)

```markdown
## Nightly always-loaded optimizer (one writer, once per night)
1. Measure first: current size vs thresholds (CLAUDE.md yellow 100KB / red 120KB;
   MEMORY.md working zone 60-100 lines, hard host cut at 200 lines / 25,000 UTF-16
   units -- units, not bytes: a Cyrillic index budgeted in bytes stops at half its
   real headroom, a CJK one at a third).
2. Fold, don't delete: body text that grew inside the index moves to its topic file;
   the index line shrinks to trigger + gist + pointer.
3. Hub crowded domains: 5+ lines on one domain → one hub file + one index line.
4. Re-sort INBOX lines into their sections; INBOX ends empty.
5. Archive superseded lines to the archive file; before archiving, scan the line's topic file
   tail for pending|BLOCKED|TODO and surface unfinished items instead of burying them.
6. Bump the version header; append one changelog line: what · why · md5 of the file.
7. Never compress wording for its own sake — structure over prose-squeezing; a rewrite that
   saves 5% and risks a rule is a bad trade.
```

## Canary checks (prove your host's limits yourself)

```markdown
- MEMORY.md truncation canary: append a unique line at the very bottom; next session, ask the
  agent to quote it. Missing = your index is over the silent cut — shrink below the limit.
- CLAUDE.md load canary: add a unique marker near the end; ask the agent for it in a fresh
  session. (Measured on our host: no cut up to 401,643 bytes — the cost is tokens, not loss.)
- Session-start cost: log tokens-at-start daily; a steady climb with no new features = bloat tax.
```
