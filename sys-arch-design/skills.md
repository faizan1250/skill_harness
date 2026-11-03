# Skill — System Design & Architecture Decisions

**Audience: this file is read by both the orchestrator AND every subagent dispatched on a system-design task.** Sections marked `[ORCH]` are for the orchestrator only. Sections marked `[SUBAGENT]` are instructions for whichever subagent is doing that phase's work. Sections with no marker are shared doctrine — everyone internalizes them before doing anything else.

This skill does not replace route skills (OS-Contribution / Personal-Proj) or the Tool Lockdown. The orchestrator still never reads code, never runs commands, never touches files itself. Every fact-finding action — codebase inventory, live web research, benchmark lookups — is a subagent dispatch.

**Orchestrator Skill Access note:** this file must be added to the orchestrator's Skill Access allow-list (alongside the route skills) before it can be read without a per-session explicit user instruction — reading it is not covered by the existing route-skill or `open-*` exceptions. Treat it as read "when the task is a system-design/architecture decision," the same way route skills are read "when the task is OSS/Personal." If it isn't yet on that allow-list, that's a one-line addition to the orchestrator core, not a reason to skip this skill.

---

## Core Doctrine (read first, everyone)

Great system design judgment is not knowing the most technologies. It's knowing which constraints actually bind for *this* project and picking the least complex thing that satisfies them — with the reasoning on paper. A top-tier engineer is distinguished less by what they know and more by what they refuse to do without evidence.

1. **Constraint-first, not technology-first.** Never start from "what should we use" — start from "what has to be true" (scale, latency, consistency, team, budget, compliance, timeline). Technology choice is the output of constraints, not an input. If a subagent or the orchestrator catches itself naming a technology before the constraints sheet exists, stop and go back to Phase 1.
2. **Boring technology bias.** Default to the most proven, most boring option that satisfies the constraints. A novel/trendy choice must justify itself against the boring option in writing — "it's popular," "it's what I know," or "it's modern" are not justifications. Complexity is a cost paid on every future day (on-call load, hiring difficulty, debugging surface), not a one-time cost. The bar for introducing a new moving part should rise with how many moving parts already exist.
3. **No resume-driven development.** Never recommend a technology because it looks good on a portfolio, is currently trending, or because a subagent's search results were full of hype posts. Hype volume is not evidence of fitness. A technology's GitHub star count or "X is dead, switch to Y" blog-post volume says nothing about whether it fits these specific constraints.
4. **Reversibility awareness.** Classify every decision as reversible (cheap to undo — pick fast, move on) or irreversible/expensive-to-undo (data model, core protocol, primary datastore, vendor lock-in — slow down, get more evidence, prefer the option with the better escape hatch). Spend diligence proportional to irreversibility, not uniformly. A caching library choice and a primary-database choice do not deserve the same research depth.
5. **Explicit tradeoffs, no free lunches.** Every option has a cost. If a recommendation doesn't name what it's giving up, it hasn't been thought through — redo it. "Faster AND cheaper AND simpler than the alternative, no downside" is a signal the analysis is incomplete, not that a free lunch was found.
6. **Cite real evidence.** Claims about scale, performance, or fitness must be backed by something a subagent actually found (docs, benchmarks, postmortems, case studies at comparable scale) — not background knowledge that might be stale. See Phase 2.
7. **State the unknowns.** If a constraint needed to decide is missing, that's a question to the user — never a silent assumption. Guessing and presenting it as fact is worse than asking; a wrong guess compounds silently while an asked question costs one reply.
8. **Right-size to what's actually asked.** A "should I use Redis here" question gets a tight, fast answer using this framework's spirit, not the full 4-phase pipeline. A "design the architecture for X" question earns the full pipeline. Match diligence to the size of the decision and its reversibility (point 4), not to a rigid process requirement. See Calibration Examples below.
9. **Known unknowns beat false precision.** When a number can't be pinned down (e.g. exact future traffic), say so and design for an order-of-magnitude range rather than anchoring on a fabricated precise figure. A range with stated confidence is more honest and more useful than a specific number nobody can defend.
10. **Prefer the option with graceful degradation.** Between two options that both satisfy stated constraints, favor the one that fails more gracefully under unexpected load or partial outage — e.g. degrades to slower/stale-but-available over hard-down. This is a tiebreaker, not an override of explicit constraints.

### Calibration Examples

- *"Should I add a Redis cache in front of this lookup?"* — narrow, reversible, single-axis. Fast pass: confirm the actual bottleneck is read latency/DB load (not guessed), confirm team can operate one more service, give a direct answer with the one-line tradeoff (added ops surface, cache invalidation complexity) named. No need for parallel subagent dispatch across four axes.
- *"We're hitting DB connection limits under load, what do we do?"* — medium. Needs real Phase 1 (what's the actual load pattern, is this a connection-pooling problem or a genuine scale problem) before jumping to "shard the database," which is usually the wrong first move.
- *"Design the backend architecture for a new multi-tenant SaaS product."* — full pipeline. Multiple irreversible decisions (data model, tenancy isolation strategy, core datastore) depend on constraints that don't exist yet (expected tenant count, team size, compliance needs). Skipping Phase 1 here produces an architecture built on guesses.
- *"Our monolith is getting hard to work in, should we move to microservices?"* — full pipeline, brownfield-flavored. The instinct to reach for microservices as the default answer is exactly the over-engineering anti-pattern below — Phase 1 needs to establish what's actually causing the pain (team coordination? deploy coupling? a specific scaling bottleneck?) before any technology or pattern is proposed, because the fix for those is often not the same thing.

---

## The Framework — 4 Phases

Use all four for a real architecture decision (new system, or meaningful change to an existing one). Collapse Phases 1–3 into a fast pass for a narrow, reversible, single-axis question (see Doctrine §8) — but never skip Phase 1 constraint-check or Phase 3 tradeoff-naming entirely; just make them brief.

### Phase 1 — Context & Constraint Extraction `[mostly SUBAGENT, gated by ORCH]`

**`[ORCH]`** Dispatch one or more context-gathering subagents before any option research starts. Don't let research begin on assumed constraints.

**`[SUBAGENT]`** Establish whichever of these actually apply to the ask — don't force irrelevant categories, don't skip relevant ones:

- **Scale:** current and expected load (requests/sec, data volume, users, growth curve). Order-of-magnitude is enough; false precision isn't the goal.
- **Latency & consistency needs:** what's actually user-facing and latency-sensitive vs. backend/batch; where strong consistency is non-negotiable (money, inventory, auth) vs. where eventual is fine.
- **Team:** size, existing skillset, on-call capacity. A two-person team cannot operate the same stack as a 50-person platform org — this is a real constraint, not a lesser concern.
- **Budget & ops tolerance:** managed service vs. self-hosted tolerance, infra spend ceiling if known.
- **Compliance / data residency / security requirements**, if any.
- **Timeline pressure.**
- **If this touches an existing system:** inventory what's already there (current architecture, what's actually causing pain vs. what's just unfamiliar, prior incidents tied to the current design, migration/downtime tolerance, what must NOT break). Get this from subagents reading the actual repo/docs/infra config — never from memory or assumption.

**`[SUBAGENT]`** If the user's prompt doesn't specify enough of the above to constrain the decision, report back to the orchestrator with the specific missing items — don't invent plausible-sounding numbers to fill the gap. A reported gap should name which downstream decision it blocks (e.g. "team on-call capacity unknown — blocks managed-vs-self-hosted call on the datastore axis"), not just list the missing fact in isolation.

**`[ORCH]`** If subagents report missing constraints that materially change the answer, ask the user directly. Do not proceed to Phase 2 on guessed constraints for anything irreversible (Doctrine §4). For reversible, low-stakes gaps, it's acceptable to proceed with a clearly flagged assumption rather than blocking on a round-trip — state the assumption plainly in the constraints sheet so it can be corrected later, never bury it.

**`[SUBAGENT, brownfield]`** When inventorying an existing system, distinguish three things that get conflated constantly: (a) what's actually causing measured pain (an incident, a metric, a missed SLA), (b) what's merely unfamiliar or unloved by the current team, and (c) what's a legitimate future risk with no current symptom. Only (a) justifies urgency. (b) is a bias to flag and discount, not a reason to rebuild. (c) belongs in the revisit-triggers section of the final doc, not the immediate recommendation, unless it's genuinely load-bearing now.

Output of this phase: a short constraints sheet, plain language, no fluff, with any assumptions explicitly flagged as assumptions.

### Phase 2 — Option Space Research `[SUBAGENT, parallel, dispatched by ORCH]`

**`[ORCH]`** Dispatch one subagent per decision axis in parallel (e.g., one for "datastore," one for "messaging/queueing," one for "caching layer," one for "deployment/scaling topology" — only the axes actually relevant to the ask). Each gets the Phase 1 constraints sheet as input.

**Axis-boundary rule (scope isolation):** a Phase 2 subagent's dispatch covers its assigned axis only. If its research surfaces that a *different*, non-dispatched axis is actually relevant (e.g., the caching subagent's findings suggest the real fix is a messaging pattern, not caching), the subagent does not pursue that axis itself — it finishes its assigned axis as scoped, flags the discovery in its report, and the orchestrator decides whether to open a new Phase 2 dispatch for it. This mirrors the core orchestrator's Single-Action Isolation Rule: a dispatch for axis A is not license for axis B, however related it looks mid-research.

**`[SUBAGENT]`** For your assigned axis:

- Live web research for *current* best practice — training-data knowledge of "what's good" can be stale; verify against recent sources (docs, engineering blogs, benchmark posts, comparable-scale case studies) rather than reciting from memory.
- Surface 3–5 real candidate options. Skip options that fail an obvious hard constraint (e.g., a tool with no managed offering when ops tolerance is near zero) — don't carry dead-on-arrival options forward just to pad the list.
- For each candidate, report: what it's good at, what it costs (money, ops burden, learning curve, lock-in), known failure modes or scaling ceilings, and who else runs it at comparable scale (with source).
- Flag hype vs. substance explicitly — "trending in search results" is not the same as "proven fit." Say so if that's what you're seeing. Weight a vendor's own benchmark post lower than an independent third-party comparison or a public postmortem; vendors benchmark their own strengths.
- Note reversibility: how hard is it to migrate away from this choice later? Lock-in isn't binary — note specifically what would need to change (data export path, API surface, proprietary query language) to leave.
- Where possible, find the failure story, not just the success story: a known outage, a scaling ceiling someone hit, a "why we moved off X" post. The failure modes matter more than the feature list for a constraint-driven decision, because features rarely differentiate at the shortlist stage — operational behavior under stress does.
- Don't stop at the first comfortable answer. If the first search mostly surfaces marketing pages or top-of-funnel "best X for Y in 2026" listicles, treat that as a prompt to search again with sharper terms (specific competitor names, "postmortem," "incident," "vs," the actual scale numbers from Phase 1) rather than reporting shallow findings back.

Output of this phase: per-axis option lists with sourced pros/cons, fed back to orchestrator. Each candidate should carry at least one specific, attributable source — not an unsourced general impression.

### Phase 3 — Tradeoff Synthesis & Elimination `[ORCH, or a dedicated synthesis SUBAGENT]`

This is reasoning over already-gathered material, not fact-finding — it's fine for the orchestrator to do this directly, or dispatch a synthesis subagent if the option space is large. **If dispatched to a subagent, that subagent reasons only over the Phase 2 reports it was handed — it does not go back to the web or reopen research.** If synthesis surfaces a real evidence gap (missing data needed to break a tie, a claim that needs stronger sourcing), it reports the gap back to the orchestrator as a finding, same as a Phase 1 gap — the orchestrator then decides whether to reopen a scoped Phase 2 dispatch for it, rather than the synthesis subagent quietly expanding its own scope into new research.

1. Cross-reference every Phase 2 candidate against the Phase 1 constraints sheet.
2. Eliminate options that violate a hard constraint — state the specific constraint violated, not a vague "doesn't fit." A hard constraint is one where failing it isn't a matter of degree (e.g. "no managed PostgreSQL offering in required region" for a hard compliance requirement); soft preferences (team mildly prefers X) go in the matrix as a weighted factor instead, not an elimination.
3. For survivors, build a ranked tradeoff matrix: axis (cost, ops burden, latency fit, consistency fit, team fit, reversibility, scaling ceiling) × candidate.
4. If this touches an existing system, add a cost-of-change row: migration effort, risk, downtime, what breaks.
5. Explicitly flag when two options are close — don't force false confidence where the evidence doesn't support a clean winner. A legitimate output of this phase is "two viable options, here's the one deciding factor that would break the tie" rather than a single recommendation manufactured from thin evidence.
6. Weight axes by what Phase 1 actually flagged as binding, not evenly. If the constraints sheet shows team size is the dominant constraint, team-fit and ops-burden should dominate the ranking — don't let scaling-ceiling comparisons between options drive the recommendation when nothing in Phase 1 suggested that scale is the live risk.
7. Sanity-check the surviving leader against Doctrine §2 and §3 before finalizing: if the top-ranked option is also the newest/trendiest candidate, re-examine whether that happened on the evidence or because Phase 2 research skewed toward hype sources (Doctrine §3). This isn't a bias against new technology — it's a check that the win is real.
8. **Source conflicts:** if two Phase 2 reports cite conflicting evidence for the same claim (e.g. one source says a tool handles a given scale comfortably, another documents it failing at that scale), do not average or silently pick one. Prefer the more specific, more recent, more comparable-scale source; if neither clearly wins, state the conflict explicitly in the matrix rather than resolving it silently, and let it lower the confidence level in Phase 4 rather than being hidden.

### Phase 4 — Decision Doc `[ORCH assembles]`

Final output to the user, in this shape:

1. **Recommended default** — the single pick, stated plainly, with the 2–4 sentence "why" tied directly to Phase 1 constraints (not generic praise).
2. **Ranked matrix** — the full Phase 3 matrix, so the user can see the reasoning, not just trust it.
3. **Rejected alternatives** — each with the specific reason it lost, in one line. This is what proves the recommendation wasn't the only thing considered.
4. **Risks & revisit triggers** — what could make this wrong later, and the concrete signal that means "come back and reassess" (e.g., "revisit if write volume exceeds X/sec" or "revisit if team grows past Y engineers").
5. **Path to implementation** — naturally either a target-state build-out (new system) or a phased migration plan (existing system: what changes first, what stays, rollback points) — whichever the Phase 1 context actually called for. Do not invent a migration plan for a greenfield ask or a from-scratch plan for a brownfield one. For a migration, sequence by risk: the step that's cheapest to reverse and gives the earliest real signal goes first, not necessarily the step that delivers the most value first.
6. **Confidence level** — one of **High** (direct comparable-scale evidence, no unresolved source conflicts, constraints sheet had no load-bearing assumptions), **Medium** (reasonable evidence but at least one flagged assumption or a non-exact scale/stack match), or **Low** (thin precedent, an unresolved Phase 3 source conflict, or a load-bearing constraint that had to be assumed rather than confirmed). State which one and the one-line reason — not a vague gesture at confidence. A confident-sounding doc built on thin evidence is more dangerous than one that's honest about its limits.

Keep the doc itself dense — this is a reference artifact, not a narrative essay. Prefer the matrix and bulleted reasoning over prose paragraphs; the user should be able to scan it in under two minutes and audit the reasoning in five.

---

## Anti-Patterns — reject these even under time pressure

- Recommending a technology before constraints are established.
- A "pros and cons" list where every option somehow has the same number of pros (sign that fitness wasn't actually weighed against constraints).
- Citing popularity, GitHub stars, or hype as the primary justification.
- Silent assumptions about scale, team, or budget presented as if they were given.
- A migration plan for a system that doesn't exist yet, or a greenfield plan that ignores a running production system.
- Over-engineering for hypothetical scale the project has no evidence it will reach (Doctrine §2 — boring technology bias applies to architecture patterns too: don't reach for microservices/event-sourcing/multi-region because a greenfield project *might* need it someday without evidence it will).
- Under-engineering past a stated hard constraint (compliance, consistency) to save complexity.
- Treating "what's causing pain" and "what's unfamiliar to the current team" as the same thing in a brownfield inventory (see Phase 1 brownfield note) — this is the single most common source of unjustified rewrites.
- A tradeoff matrix where every axis is weighted equally regardless of what Phase 1 actually flagged as binding (see Phase 3 point 6) — this quietly launders an unweighted gut preference into something that looks rigorous.
- Presenting a two-way toss-up as a confident single recommendation because a clean answer feels more useful than an honest one (Phase 3 point 5, Phase 4 point 6).
- Recommending the newest/trendiest surviving option without the Doctrine §2/§3 sanity check (Phase 3 point 7).
- A subagent chasing a discovery outside its assigned Phase 2 axis instead of flagging it and letting the orchestrator open a new scoped dispatch (Phase 2 axis-boundary rule) — this is scope drift, same violation the core orchestrator's Single-Action Isolation Rule exists to prevent.
- A synthesis subagent quietly going back to the web mid-Phase-3 instead of reporting the evidence gap back (Phase 3 intro) — synthesis reasons over what it was handed, it doesn't reopen research on its own authority.
- Silently resolving a source conflict between two Phase 2 reports instead of surfacing it and letting it affect confidence level (Phase 3 point 8).

---

## Dispatch Checklist `[ORCH]`

Before closing out a system-design task, confirm:

- [ ] Constraints sheet exists and came from subagents/user, not assumption
- [ ] Every recommended option has sourced evidence behind it (Phase 2)
- [ ] Every surviving option in the matrix has a named tradeoff, not just the winner
- [ ] Rejected alternatives each have a one-line reason
- [ ] Revisit triggers are concrete, not vague
- [ ] If existing system involved: migration path present; if greenfield: no invented migration path
- [ ] Matrix axes were weighted by what Phase 1 flagged as binding, not applied evenly by default
- [ ] If the winning option is also the newest/trendiest, the Doctrine §2/§3 sanity check was actually done, not skipped
- [ ] Confidence level is stated as High/Medium/Low with a one-line reason, not a vague gesture
- [ ] Brownfield only: pain-vs-unfamiliarity distinction was made explicitly, not assumed
- [ ] No Phase 2 subagent chased an out-of-scope axis instead of flagging it (axis-boundary rule)
- [ ] No synthesis subagent reopened web research mid-Phase-3 instead of reporting the gap
- [ ] Any Phase 2 source conflicts were surfaced explicitly, not silently resolved
- [ ] This skill's path is on the orchestrator's Skill Access allow-list for this task type

---

## Example: Compressed Walkthrough

To make the four phases concrete, here's how they'd run (abbreviated) for: *"Our Postgres-backed order service is slow under Black-Friday-level traffic, what should we do?"*

- **Phase 1:** Subagent inventories current schema, query patterns, and actual slow-query logs (not guessed) from the repo/infra docs. Finds the bottleneck is a handful of unindexed hot queries plus connection-pool exhaustion, not raw data volume. Team is 4 engineers, no dedicated DBA, downtime tolerance near zero during the sale window. Flags: exact peak RPS unknown — asked to user rather than guessed, since it's load-bearing for the decision.
- **Phase 2:** Parallel subagents research (a) query/index optimization and pooling (PgBouncer, read replicas), (b) caching layer options, (c) "should this become a different datastore entirely" as a sanity-check axis, each with sourced comparable-scale evidence. Axis (c) comes back weak — no evidence in Phase 1 that this is a datastore-fit problem rather than a query/pooling problem.
- **Phase 3:** Axis (c) is down-weighted hard per Phase 3 point 6/7 — it's the trendiest-sounding option but the evidence doesn't support it being the actual fix, and recommending a datastore migration for a query-indexing problem would be textbook over-engineering (Anti-Patterns). Indexing + pooling + a targeted read-replica survive as the ranked matrix leaders, weighted heavily on reversibility and small-team ops burden given the 4-engineer, no-DBA constraint.
- **Phase 4:** Recommended default is indexing fixes + PgBouncer, with a read replica as the next step if that's insufficient — not a datastore swap. Rejected: full migration to a different datastore (reason: no evidence the current engine is the bottleneck; high irreversibility for unproven benefit). Revisit trigger: "if indexed query latency is still above target after pooling fix, re-open the datastore-fit question with real post-fix numbers." Confidence: **High** — direct evidence from slow-query logs, no unresolved source conflicts in Phase 2, only one flagged assumption (peak RPS) and it wasn't load-bearing for the final pick.