# Agent Skill — Feature Implementation (Personal Projects Only)

This is an **orchestrator-only** skill, same tier as the `open-*` skills. You read it yourself. You never hand its body to a subagent.

**The user's word is number one**, per the orchestrator core. Everything below is subordinate to any explicit instruction the user gives for the task at hand.

---

## File Write Exception

The orchestrator may write and later delete exactly one file per feature, nowhere else:

```
[project-root]/agents/feat/<feature-slug>.md
```

- `<feature-slug>` — orchestrator names it per feature (kebab-case, derived from the feature name/ticket), never a literal `file.md`.
- Before first write, the orchestrator checks `[project-root]/.gitignore` for an `agents/` entry (or `agents/feat/`). If absent, the orchestrator adds it. This is the only other write exception this skill grants — gitignore maintenance for this one directory, nothing else.
- This file is the **Phase 1 Plan File** (see below). It is written once at the end of Phase 1, read by every subsequent phase's dispatch, and **deleted** once Phase 4 confirms the PR is 100% green with zero bot issues. Do not delete early. Do not keep it past a green PR.
- No other file read/write/edit/delete is permitted under this skill. All Tool Lockdown rules from the orchestrator core still apply in full — the plan file is the sole, narrow exception, exactly as the OSS dossier is for `open-pre-fix`.

---

## Skill Access

Orch agent are prohibited to read these skills , these are for subagents only.

| Skill | Path | Assign in |
| --- | --- | --- |
| frontend | `/home/faizan/SKILLS/frontend/skills.md` | Phase 2, new frontend surface |
| backend | `/home/faizan/SKILLS/backend/skills.md` | Phase 2, new backend/API surface |
| ui-ux | `/home/faizan/SKILLS/ui-ux/skills.md` | Phase 3, feature touches an existing shipped surface |
| issue-fix | `/home/faizan/SKILLS/issue-fix/skills.md` | Phase 3, feature build surfaces a bug in existing code that must be fixed in-line |
| review | `/home/faizan/SKILLS/review/skills.md` | Phase 3, every reviewer lens dispatch |

As with the orchestrator core: you choose the path, you pass it to the subagent, you never read the general skill's body yourself.

---

## Phase 1 — Plan, Scope, Architecture, Edge Cases (CRITICAL PHASE — NO HURRY)

**This is the most critical phase in the entire pipeline.** Every downstream phase inherits Phase 1's blind spots — Phase 2 builds exactly what Phase 1 specifies, Phase 3 only reviews against what Phase 1 wrote down, and a gap here is invisible until production. Speed is not a goal of this phase; completeness is. The orchestrator must not advance, must not accept a thin report, and must not let a subagent rush this to save time. If a single planning subagent's report feels shallow on any section below, redispatch — do not downgrade the bar to move forward.

**Goal:** produce the single verbatim Plan File that Phase 2's build subagent will follow with zero interpretation latitude, written to the standard of a senior/staff-level engineer doing a real design review — not a quick scope note.

1. Confirm the feature request with the user if anything is ambiguous (per "user's word is number one" — do not assume scope).
2. Dispatch 4 dedicated planning subagents in sequence or parallel to produce their respective sections with depth and zero rush (none are optional; if truly not applicable, the subagent must justify briefly):

   - **Subagent 1: Scope & Architecture**
     - **Scope statement** — what is in, and an explicit **out-of-scope list** (mirrors the Scope Discipline spirit from the OSS core: anything not named is not built).
     - **Architecture & design** — approach, affected files/modules (named, not touched), data flow or component tree as applicable, and **why this approach was chosen over plausible alternatives** — a one-line rationale per rejected alternative is enough, but it must be there.

   - **Subagent 2: Data Model, Concurrency & Performance**
     - **Data model & DB impact** — schema changes (new tables/columns/indexes), migration plan and whether it's backward-compatible/reversible, query patterns the feature introduces, and expected read/write volume shape.
     - **Performance considerations** — expected load, any N+1 or hot-path risk, indexing needs, caching strategy if relevant, and any operation that could be slow at scale even if fine at current scale.
     - **Concurrency & race conditions** — anything touched by more than one actor/request at once: locking strategy, idempotency, double-submit handling, ordering guarantees where needed.

   - **Subagent 3: System Impact, Security & Resilience**
     - **System-level impact** — what else in the system this feature touches, interacts with, or could be broken by: upstream/downstream services, shared state, existing feature interactions, third-party integrations.
     - **Security & auth boundaries** — who can call this, what they can see/modify, input validation/sanitization needs, and any new attack surface introduced.
     - **Failure modes** — what happens when a dependency is down, a request times out, or a partial write occurs; whether the feature degrades gracefully or fails loudly, and which is intended.

   - **Subagent 4: Edge Cases & Acceptance Criteria**
     - **Edge case handling plan** — enumerated edge cases and the intended behavior for each: error states, empty states, boundary values, malformed/unexpected input, permission edge cases. This should read as a checklist, not a paragraph.
     - **Acceptance criteria** — concrete, checkable conditions Phase 3 will later grade against, directly traceable to the sections above (every architecture decision, DB change, and edge case above should map to at least one acceptance criterion).

3. The orchestrator synthesizes the 4 subagent outputs verbatim into `[project-root]/agents/feat/<feature-slug>.md`. The orchestrator does not edit, soften, or reinterpret the subagents' content — it is a transcription, not a rewrite. If any subagent's report is thin, generic, or skips a section without justification, **redispatch that specific subagent** rather than filling gaps yourself or accepting a weaker plan to save a round trip.
4. **Mandatory red-team pass.** If the Plan File's Data Model & DB Impact, Concurrency & Race Conditions, or System-Level Impact sections describe any of: a new/altered schema, more than one actor touching shared state, or a dependency on another service/module — this is not optional. Dispatch a red-team subagent whose only job is to red-team the synthesized plan's architecture, DB, and concurrency sections: find what's missing, what could fail under load, what the plan assumed without stating. The orchestrator folds confirmed findings into the Plan File before Phase 2. Skip this pass only when none of the three trigger conditions apply, and record in the Plan File's Scope section one line stating why it was skipped.
5. **Depth check, not presence check.** "Contains all sections" is not satisfied by a section merely existing. For each section, the orchestrator checks it against one concrete test: could a build subagent with zero other context implement that section's requirement correctly from this text alone? A section that is a single vague sentence ("handle errors gracefully," "should scale fine") fails this test regardless of heading. Redispatch naming the specific section and specific gap — not a generic "add more detail" request.
6. **Retry cap.** Up to 2 redispatches per planning subagent for the same gap. If a 3rd redispatch would be needed, stop and escalate to the user with: what's missing, what the subagent produced across all attempts, and why it's insufficient. Do not loop a 4th time.
7. **Hard gate.** Phase 2 cannot start until the Plan File passes the depth check on all sections, and the red-team pass has run or was explicitly waived with a stated reason. No parallel start, no partial handoff, and no advancing "to not waste time" — time spent here is the entire point of this phase.

---

## Phase 2 — Build

1. Dispatch a build subagent with: the full, verbatim contents of the Plan File, and the exact path to the one general skill matching the feature's shape, per the Skill Access table. If the Plan File's design spans both a new frontend surface and a new backend/API surface, dispatch two build subagents in sequence within this phase (backend first, since frontend typically consumes it) — not in parallel, since the frontend subagent needs the backend subagent's actual interface, not the planned one, to avoid building against a guess.
2. Instruct the build subagent to **strictly follow the Plan File** — no scope expansion beyond it, no silent deviation from the edge case plan. Any case where the subagent believes the plan is wrong or incomplete must come back as a flagged report to the orchestrator, not a unilateral change.
3. The build subagent's report must end with a structured **Scope Affirmation**, not prose: explicit yes/no answers to (a) zero out-of-scope changes beyond the Plan File's scope statement, (b) zero deviation from the edge case plan without a flagged justification, (c) zero schema/data-model change beyond what Phase 1 specified. Any "no" must list the specific deviation and reason. A report without this block is incomplete and is redispatched on that basis alone, before the orchestrator even evaluates the work.
4. **Retry cap.** Up to 2 redispatches for the same undisclosed-scope or deviation issue. On a 3rd occurrence, escalate to the user rather than redispatching again — do not loop a 4th time.
5. **Hard gate.** If the Scope Affirmation shows any unresolved "no," or is missing, do not advance — redispatch with the discrepancy named, or escalate to the user per the retry cap if it looks like the Plan File itself was wrong.

---

## Phase 3 — Post-Build Review (Execution vs Phase 1 Plan Audit)

**Subagent count:** constant **4 review subagents** dispatched in parallel or sequence, directly mirroring the 4 planning domains established in Phase 1. Every review subagent is mandatory — no guessing lens count, zero under-scoping.

**Core Mandate:** Phase 3 subagents do not perform generic rubber-stamp reviews. Their primary directive is to audit the actual **execution (code & diffs)** directly against the **Phase 1 Plan File** (`[project-root]/agents/feat/<feature-slug>.md`), verifying that what was planned was implemented verbatim with zero unauthorized scope drift or skipped commitments.

The 4 dedicated review subagents (each dispatched as a dedicated `review`-skill subagent with the Phase 1 Plan File, acceptance criteria checklist, and the Phase 2 build report/diff):

1. **Review Subagent 1: Scope, Architecture & Design Fidelity** — compares the executed code line-by-line against Phase 1 Subagent 1's specifications (Scope Statement, In/Out-of-Scope boundaries, affected files/modules, and architectural pattern). Verifies no unauthorized scope creep occurred and no extra files were modified beyond the design.
2. **Review Subagent 2: Data Model, Concurrency & Performance Audit** — audits the execution against Phase 1 Subagent 2's specifications (schema/DB migrations, query patterns, indexing, hot paths/caching, and race condition/locking/idempotency mechanisms).
3. **Review Subagent 3: System Impact, Security & Resilience Verification** — verifies the execution against Phase 1 Subagent 3's specifications (system-level blast radius/shared state regressions, auth & access boundaries, input validation/sanitization, failure modes, timeouts, and graceful degradation).
4. **Review Subagent 4: Edge Cases & Acceptance Criteria Verification** — checks every single enumerated edge case and failure scenario from Phase 1 Subagent 4 against the actual code. Verifies that every acceptance criterion is concretely satisfied and testable.

Rules:
- **All 4 Subagents Mandatory:** All 4 review subagents run on every feature. If a subagent's domain has sections justified as non-applicable in Phase 1, that subagent confirms that no code was introduced in that area.
- **Auditing Execution to Plan:** Each review subagent reports line-level pass/fail per item from the Phase 1 plan it audits, not an abstract summary.
- **Scope Affirmation Verification:** Each review subagent report must independently validate or contest the Phase 2 build subagent's **Scope Affirmation** block — verifying zero undisclosed scope drift or deviation.
- **Hard gate:** Any review subagent reporting a `must-fix` sends the task back to a Phase 2 redispatch (never a quiet patch by the reviewer).
- **Re-run rule, not judgment call:** After a Phase 2 fix, re-run only the specific review subagent(s) that failed, plus any subagent whose audited items overlap the files touched by the fix. Subagents with zero file overlap are not re-run.
- **Retry cap:** Up to 2 fix-and-re-review cycles per `must-fix` finding. On a 3rd occurrence of the same finding, escalate to the user rather than cycling again.
- Phase 4 cannot start until all 4 review subagents report clean with uncontested Scope Affirmations.

---

## Phase 4 — PR + CI/Bot Convergence Polling

1. Dispatch a subagent, using the `issue-fix` skill path (it is the general skill covering git/gh actions in the inventory), to raise the PR.
2. **Polling loop:** dispatch a polling subagent, same `issue-fix` skill path, that checks CI status and review-bot output every **1.5 minutes**, looped, until CI finishes and all bots have returned 100% of their responses (not partial — a bot that's still "in progress" does not count as answered). Instruct it explicitly to query CI and each bot's actual current state on every loop iteration — not to cache or assume continuity from the prior check.
3. If CI fails or a bot raises issues: dispatch a fix subagent against the specific failure/issue, push, and **restart the 1.5-minute polling loop from zero** — do not resume a stale count or assume partial re-checks suffice.
4. Repeat step 3 until the PR is 100% green on CI **and** 0 outstanding bot issues simultaneously.
5. **Stall cap.** If a single CI failure or bot issue survives 3 fix-and-poll cycles without resolving, or the polling loop runs past 2 hours of elapsed time without reaching green, stop and escalate to the user with current CI/bot state rather than continuing to loop.
6. **CI/Bot Green-Claim Verification Rule (mirrored from orchestrator core):** when the polling subagent reports 100% green and 0 issues, do not accept this at face value. Dispatch a separate confirmation subagent — same `issue-fix` skill path, but explicitly instructed to independently query CI's and each bot's status itself from scratch, not to re-read or trust the polling subagent's last report. If the confirmation contradicts the claim, resume the fix-and-poll loop (subject to the stall cap above) — do not treat the original claim as provisionally true while you wait for confirmation.
7. Only once the independent confirmation agrees the PR is 100% green with 0 bot issues: delete the Plan File at `[project-root]/agents/feat/<feature-slug>.md`. This is the only point in the pipeline where deletion is authorized.

---

## Summary of Hard Gates

| Transition | Gate |
| --- | --- |
| Phase 1 → 2 | Plan File passes depth check across all 4 planning subagent domains; red-team pass run or waived with stated reason |
| Phase 2 → 3 | Build report's Scope Affirmation is present and fully "yes" |
| Phase 3 → 4 | All 4 review subagents report clean with uncontested Scope Affirmations |
| Phase 4 → done | Independent confirmation agrees: 100% CI green, 0 bot issues |

No phase may be skipped. No phase may run in parallel with another (within-phase sequential sub-dispatches, as in Phase 2's backend-then-frontend build, are not a violation of this). Each gate is binary — partial pass redispatches, it does not advance.

**Escalation is not a failure state.** Every retry cap above routes to the user, not to a silent stall or an infinite loop. If the orchestrator finds itself about to redispatch past a stated cap for any reason, that is itself a bug in execution — stop and escalate instead.