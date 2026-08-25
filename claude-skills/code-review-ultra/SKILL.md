---
name: code-review-ultra
description: Deep, recall-maximizing multi-agent code review of a diff/PR, run LOCALLY via parallel sub-agents — the same finder-fan-out → verify → sweep technique behind the cloud `/code-review ultra`, for when cloud ultra isn't on your plan or you want a heavy local pass. Fans out up to 9 independent review angles (5 correctness + 3 cleanup + 1 altitude), verifies each candidate against the real code (quoting lines), sweeps for gaps, and returns ≤15 ranked findings split correctness-vs-cleanup. Use when the user asks for an "ultra review", "ultrareview", "deep/thorough/rigorous code review", "review my diff/PR for every bug", or maximum bug recall on changed code. For a quick lighter pass use the bundled `/code-review`; for a plan/design doc use `/audit-plan`.
---

# code-review-ultra

A deep, **recall-first** code review of the changed code, run **locally** by fanning out independent finder sub-agents — the same multi-agent method the cloud `/code-review ultra` uses, minus the cloud. Use it when cloud ultra is unavailable (plan-gated / billed) or you just want a heavy local pass. The goal at this level: **catch every real bug** — a missed bug ships, so err toward surfacing.

> This skill does **NOT** call the cloud. It replicates the technique with local sub-agents. If you actually want the cloud run, that's `claude ultrareview` from a terminal (user-triggered + billed) — out of scope here.

## Usage
`/code-review-ultra [<PR# | branch | path | "HEAD">] [against <base>]`
- **no arg** → review the current branch vs its upstream/base, plus uncommitted working-tree changes.
- a **PR number / branch / file path** → review that target instead.
- `against <base>` → override the diff base (e.g. `against main`).

## When to use / not use
- **USE for:** "ultra review", "deep/rigorous/thorough code review", "find every bug in this diff", max recall before a risky merge.
- **DON'T use for:** a quick low-noise pass → use the bundled **`/code-review`** (low/medium effort). Reviewing a **plan / design doc** (prose about future code) → use **`/audit-plan`**. The genuine cloud ultra → `claude ultrareview` in a terminal.

## Method — 4 phases

### Phase 0 — gather the diff
Run `git diff @{upstream}...HEAD` (fall back to `git diff main...HEAD`, then `git diff HEAD~1`). **Also run `git diff HEAD`** and include uncommitted working-tree changes — ultra reviews often run before the commit. If a PR#/branch/path was passed, review that instead. This unified diff is the **review scope**. Build an **"already-known / do-not-report"** list (fixes already applied, known/accepted issues) and pass it to every finder to keep the run low-noise.

### Phase 1 — fan out finder angles (parallel sub-agents)
Launch independent finder sub-agents **in parallel** (Agent tool). Each surfaces **up to 8 candidates** as JSON `{file, line, summary, failure_scenario}`. Do **not** let one angle suppress another — if two flag the same line for different reasons, keep both. Give every agent the diff, the repo path, and the do-not-report list. Brief each to **open the files** (read the enclosing function of each hunk; grep callers) — never trust the diff alone.

**5 correctness angles:**
- **A — line-by-line scan.** Read every hunk, then the enclosing function (bugs in unchanged lines of a touched function are in scope). Per line: what input/state/timing/platform makes it wrong? Inverted/off-by-one conditions, null/undefined deref, missing `await`, falsy-zero checks, wrong-variable copy-paste, swallowed errors, unescaped regex metachars.
- **B — removed-behavior auditor.** For every DELETED/replaced line, name the invariant it enforced, then find where the new code re-establishes it. If it doesn't: a dropped guard / error path / narrowed validation / deleted test covering a real case.
- **C — cross-file tracer.** For each changed function, grep its callers — does the change break a call site (new precondition, changed return shape, new exception, ordering/timing dep)? Check callees too: does a parallel change make a call unsafe?
- **D — language-pitfall specialist.** The classic footguns of the diff's language/framework (JS falsy-zero / `==` / closure-captured loop var; Python mutable-default / late-binding closure; Go nil-map / range-var capture; SQL injection; timezone/DST; float equality). Flag instances the diff introduces.
- **E — wrapper/proxy correctness.** When the diff adds/modifies a wrapper (cache, proxy, decorator, adapter): does every method route to the wrapped instance (not back through a registry/session/global → re-entry/recursion)? Does it forward all methods callers use?

**3 cleanup angles + 1 altitude** (hunt cleanup in the changed code; in `failure_scenario` state the concrete cost — what's duplicated/wasted/harder to maintain):
- **Reuse.** New code re-implementing something the codebase already has — grep shared/utility/adjacent modules and name the existing helper.
- **Simplification.** Redundant/derivable state, copy-paste-with-variation, deep nesting, dead code left behind. Name the simpler form.
- **Efficiency.** Wasted work the diff adds: redundant compute / repeated I/O, sequential independent ops, blocking work on startup/hot paths, closures keeping large scope alive. Name the cheaper alternative.
- **Altitude.** Is each change at the right depth, or a fragile bandaid? Special-cases layered on shared infra signal the fix isn't deep enough — prefer generalizing the mechanism over adding special cases.

### Phase 2 — verify (1-vote, 3-state)
Dedupe candidates pointing at the same line/mechanism (keep the most concrete). For each survivor, confirm against the code — one verifier vote, one of:
- **CONFIRMED** — name the triggering inputs/state + the wrong output/crash; quote the line.
- **PLAUSIBLE** — mechanism real, trigger uncertain (timing/env/config); say what would confirm it.
- **REFUTED** — factually wrong or guarded elsewhere; quote the line that proves it.

This is **recall mode**: a single non-REFUTED vote carries the finding. Don't drop on uncertainty. For a small diff, verify inline; for a large/claim-dense one, spawn a verifier sub-agent.

### Phase 3 — sweep
One **fresh** finder with the verified list: re-read the diff + enclosing functions for defects **not already listed**. Don't re-derive existing ones. Focus on what the first pass misses: moved/extracted code that dropped a guard, second-tier footguns (dataclass default evaluated once, `hash()` non-determinism, lock-scope shrink, predicate methods with side effects), setup/teardown asymmetry in tests, flipped config defaults. Up to 8 more; return empty rather than pad.

## Output
Ranked findings, most-severe first, at most **15**. Each: **`file:line` · summary · failure_scenario** (concrete inputs/state → wrong output/crash). **Correctness always outranks cleanup/altitude** when the cap forces a cut. Present as a readable ranked list (or a JSON array if asked). If nothing survives verification, say so plainly. Then **offer to apply** the clear high-severity fixes (and fold deferred/judgment items into a doc or a follow-up).

## Sizing & cost
Right-size the fan-out to the diff — heavy multi-agent runs cost real tokens:
- **Tiny diff (≤ ~150 lines / 1–2 files):** 3–4 finders covering the correctness + cleanup concerns; verify inline.
- **Medium:** 5–6 finders.
- **Large / multi-file / security-sensitive:** the full 9 angles + a verifier sub-agent + the sweep.
Don't spawn 9 agents on a 40-line change — they just re-derive the same context. Match the spend to the risk.

## Guardrails
- **Recall over precision** — at this level a missed bug matters more than a false positive; surface, then let verify filter.
- Finders must **open the files** (enclosing function, callers), not review the diff in isolation.
- Every finding **quotes the proving line** and gives a concrete `failure_scenario` — no generic advice.
- Pass the **do-not-report list** so iterative re-reviews don't re-flag applied fixes.
- **Local only** — this never invokes cloud ultra; for that, the user runs `claude ultrareview`.
