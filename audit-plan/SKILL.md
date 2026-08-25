---
name: audit-plan
description: Rigidly audit a plan, design doc, build spec, RFC, or decision doc (NOT a code diff) for completeness against its source spec, correctness of its claims against the actual codebase, scope creep, internal consistency, decision soundness, and risk/sequencing. Fans out parallel review angles, verifies every factual claim against real code (quoting lines), sweeps for gaps, and ranks findings split into "fix in the plan now" vs "defer to code-review". Use when the user asks to audit / review / sanity-check / "rigid audit" a plan, design doc, build spec, migration-decision doc, or RFC, or to check a plan against its spec. For reviewing an actual code diff or PR, use code-review instead.
---

# audit-plan

Audit a planning/design document with the rigor of a code review, adapted for prose-that-describes-future-code. The payoff: catch design, scope, and correctness errors **while they're still cheap to fix in the plan**, before they become written code that has to be reworked.

## Usage
`/audit-plan <plan-file> [against <spec-or-source>]`
- `<plan-file>` — the plan/design/spec doc to audit.
- `<spec-or-source>` (optional) — the authoritative source the plan must satisfy (a task PDF, kickoff comment, ticket, BRD, upstream spec). If omitted, infer it from the plan (plans usually name their source) or ask the user. The audit is **plan-vs-source AND plan-vs-reality** — without the source you can only do internal consistency.

## When to use / not use
- **USE for:** plan docs, build/implementation plans, design docs, migration-decision docs, RFCs; "audit this plan", "rigid audit", "review my plan vs the spec".
- **DON'T use for:** reviewing an actual code diff or PR → use **code-review**. This skill reviews the *plan*, not the code.

## Method — 3 phases

### Phase 0 — gather
1. Read the plan file in full.
2. Identify + read the authoritative source the plan implements. If a source the plan depends on is **missing** (un-downloaded attachment, absent migration/file), record it as a **BLOCKER** and mark dependent checks "unverifiable" — never guess past it.
3. Identify the repo the plan's claims refer to (and any sibling repos it references).
4. Build an **"already-known / do-not-report"** list (prior findings already applied, known blockers) and pass it to every finder, to keep the run low-noise.

### Phase 1 — fan out finder angles (parallel sub-agents)
Launch **5 independent finder sub-agents in parallel** (Task/Agent tool). If sub-agents aren't available, run the 5 angles sequentially yourself. Each returns up to 8 candidates as JSON `{summary, where, failure_scenario}`. Do not let one angle suppress another. Give every agent the plan path, the source, the repo path, and the do-not-report list.

- **A — Completeness (spec→plan):** map every requirement/instruction in the source to a place in the plan; flag anything missing, under-specified, or only partially covered.
- **B — Correctness (plan→reality):** verify **every** factual/technical claim against the **actual** code/schema — grep and read the repo, quote the proving line. Report only claims that are WRONG, imprecise, or asserted-as-fact-but-unverifiable. **This is the highest-yield angle — the agent must open the files, not trust the plan's own words.**
- **C — Scope + consistency:** unrequested work beyond the source; AND contradictions / stale cross-references *inside* the doc (TL;DR vs body, tables vs text, "open decisions" vs the sections they reference) — especially if the doc has been edited across multiple passes.
- **D — Decision soundness / altitude:** are the chosen approaches the **best** given code + source, not just internally consistent? Flag special-cases that should be shared mechanisms (or vice-versa), and decisions that reuse a weaker concept when a stronger one already exists in the codebase.
- **E — Risk / blockers / sequencing / omissions:** are the blockers real and correctly prioritized? is the ordering sound (hidden deps, migration-number collisions, deploy coupling)? what does a plan of this kind usually need but this one omits (edge cases, perf at real data scale, security/injection, dual-mode parity, test coverage of the risky paths)?

### Phase 2 — verify
Dedupe candidates that point at the same thing. For each survivor, confirm against the code: **CONFIRMED** (name the triggering inputs/state + quote the line), **PLAUSIBLE** (mechanism real, trigger uncertain — say what would confirm), or **REFUTED** (quote the line that disproves it). Drop REFUTED. For a small plan, verify inline; for a claim-dense one, spawn a verifier sub-agent.

### Phase 3 — sweep
One fresh pass over the plan + code for defects **not already found**. Don't re-derive existing ones; return an empty sweep rather than padding.

## Output
Ranked findings, most-consequential first, each: **severity · summary · where (plan section + code `file:line`) · failure_scenario**. Split into two buckets:
- **Fix in the plan now** — design / scope / decision / consistency issues (cheap now; rework if caught after code is written).
- **Defer to code-review** — implementation-robustness items that need real code to check; run **code-review** on the diff later.

End with the **blockers list** (missing inputs that gate parts of the audit). Then offer to apply the "fix now" findings to the plan.

## Sizing
**5 finders + verify + sweep is right for a plan doc.** Add more finder angles only for an unusually large or claim-dense plan. The heavy 15-agent fan-out is for large **code diffs** (that's `code-review`), not plans — over-spawning on a ~100-line doc just re-derives the same context.

## Guardrails
- Never guess on missing inputs — flag them and mark dependent sections unverifiable.
- Prefer concrete, code-backed findings (quote the line) over generic advice.
- Remember the plan is prose about *future* code: the value is catching design/scope/correctness errors before they're written.
