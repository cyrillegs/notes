---
name: tick-milestone
description: Use after a milestone PR merges, to record it in the phase plan — updates the Progress count, the milestone status row, and the build-order checklist, then opens a doc-only PR. Triggers on "tick M<N>", "mark the milestone done", "update the plan", or immediately after the user reports a milestone PR merged.
---

# Tick a milestone

Do this **the same turn the merge is reported** — unprompted. A plan that lags
reality is worse than no plan: the next session trusts it.

## What to update

In `plans/phase-<N>-<slug>.md` (or `plans/backend-service.md`):

1. **The Progress line** — bump the count (`**Overall: 5 / 9 …**`). When the last
   milestone lands, say so loudly: `**Overall: 9 / 9 — PHASE 2 COMPLETE (date).**`
2. **The milestone status row** — `☐ Not started` → `☑ Done`, plus a one-line
   record of what shipped: PR number, merge date, fleet-migration status, and
   the verification that passed.
3. **The build-order checklist** — tick the boxes, and **rewrite any line where
   what shipped differs from what was planned.** This is the important part: M2
   planned `SELECT … FOR UPDATE` and shipped advisory locks (an empty ledger has
   no row to lock). The checklist should say what exists, not what was imagined.
4. **On phase completion** — also update `plans/README.md` (status column +
   phases-complete count) and CLAUDE.md's "Current focus" line.

## Commit + PR

Branch `docs/tick-p<N>-m<M>`, doc-only. The commit message records the
*verification*, not just the tick:

```
Tick Phase 2 M3: Stock Receipt merged + fleet migrated

PR #61 merged; 20260717150000_stock_receipt applied fleet-wide. The RCPT
NumberSeries backfill verified on jane-does-corp (provisioned pre-M3):
series present at current=0. insights_ro on StockReceipt: SELECT ok,
INSERT denied.
```

PR body follows the CLAUDE.md template; mark it **Doc-only** so the reviewer
knows there's no code to read.

## Don't

- Don't batch several milestones into one tick later — the drift is the problem.
- Don't tick from the PR description. Tick from what you *verified*.
