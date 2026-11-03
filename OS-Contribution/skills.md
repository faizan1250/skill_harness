# Skill — OS-Contribution

**Scope:** Owns everything related to contributing to **Open-Source repositories** (tier-1, high-reputation, business-critical projects). This skill is loaded by the orchestrator **only after** the user (Saif) confirms the target project is an OSS contribution, not a personal project.

**This skill is independent.** It contains the full standards, routes, and 5-phase pipeline. It does not require reading `agent-orchestrator/skills.md` to function — it only cross-links to the orchestrator core for inventory references.

**Related skills:**
- Parent / Core: `C:/Users/bari2/Desktop/SKILLS/agent-orchestrator/skills.md` — role, tool lockdown, skill inventory
- Sibling: `C:/Users/bari2/Desktop/SKILLS/Personal-Proj/skills.md` — personal project pipeline (mutually exclusive)
- Orchestrator-only `open-*` skills referenced by the phases:
  - `C:/Users/bari2/Desktop/SKILLS/open-issue-finding/skills.md` (Codebase Bug Hunting only — strictly isolated)
  - `C:/Users/bari2/Desktop/SKILLS/open-issue-triage/skills.md` (Phase 1)
  - `C:/Users/bari2/Desktop/SKILLS/open-pre-fix/skills.md` (Phase 2)
  - `C:/Users/bari2\Desktop\SKILLS/open-post-fix/skills.md` (Phase 4)
  - `C:/Users/bari2/Desktop/SKILLS/open-pre-pr/skills.md` (Phase 5)

---

## Open-Source Engineering Standards & Core Laws

Every contribution to tier-1, high-reputation, business-critical open-source repositories must adhere to the highest standard of engineering rigor: proof-first, precedent-backed, minimal blast-radius, and architectural purity over fast hacks.

### 1. Multi-Agent Redundancy & No-Rushing Law
- **Time is never a constraint in open-source contributions. Rushing is a disqualifier.**
- For all critical phases (Phase 2 pre-fix mapping, Phase 4 multi-lens audit, and Phase 5 PR verification), **forbid relying on a single subagent**. Require multiple parallel subagents or sequential multi-pass verification across distinct roles:
  - **Phase 2 (Pre-fix):** Mandatory parallel subagents (between min 3 to max 7 according to issue scope) for MRE reproduction, root-cause isolation, blast-radius mapping, and repo precedent analysis.
  - **Phase 4 (Post-fix Audit):** Mandatory **7 parallel review subagents** across 7 distinct lenses (the dedicated **Lens G: Scope Alignment & Scope Drift** is the prime catch for OSS scope-drift issues), plus parallel re-audits. Single reviewer never passes gate.
  - **Phase 5 (PR Verification):** Sequential multi-pass verification across distinct roles (e.g. dedicated verification subagent running hard pre-PR gates, then PR dispatch/poll subagent). Forbid relying on a single unverified agent.
- **Draft PRs by Default:** Allow and encourage raising PRs as **Draft** (`gh pr create --draft`) while running stabilization checks to guarantee upstream maintainers never observe an incomplete, oscillating, or iterating state. Only mark ready for review (`gh pr ready`) when CI and review bots reach full green convergence.

### 2. Precedent & Classification Gate (`feat` vs `fix`)
- Before planning or drafting an OSS PR, mandate inspecting repository precedent (e.g. how similar capability flags, schema adaptations, or options were implemented in recent merged PRs).
- **Honest Conventional Commit Classification:** If a change introduces a new public configuration field, option, method, or capability override, it is strictly classified as a `feat`, never disguised as a `fix`. Disguising features as fixes damages repo integrity, breaks semver expectations, and causes maintainer rejection.

### 3. Ban Leaky Heuristic Shims (Architectural Purity)
- **Explicitly forbid runtime "sniffing" or heuristic transformations in output mappers to fake backwards-compatibility.**
- **Mandate Clean Abstraction Boundaries:** If the underlying query engine or database does not support multi-format querying without migration, do not introduce runtime inspection heuristics that risk data corruption or silent misbehavior on edge-case inputs (e.g., coercing valid strings like `"null"`, `"123"`, or `{` prefixes).
- **Document Migrations Honestly:** Document migration requirements and breaking nuances in JSDoc / documentation instead of hacking runtime transforms into output mappers.

### 4. Issue Thread & Maintainer Intent Primacy
- In Phase 1 and Phase 2, require extracting not just the bug symptom, but the full architectural discourse from the GitHub issue thread:
  - Reporter's core friction and real-world failure mode.
  - Maintainer commentary, architectural preferences, and official labels (`type: Feature`, `enhancement`, `target: patch`, etc.).
  - Community production insights (e.g. why previous naive fixes were rejected or why opt-in defaults are required).
- The final PR description must explicitly address the full issue thread context and directly reference upstream design consensus.

### 5. Review Bot Feedback Anti-Whac-A-Mole Rule
- In Phase 5 (CI & Review Bot Convergence Loop), when an AI review bot (CodeRabbit, Greptile, cubic) raises an issue with an ad-hoc heuristic, **strictly prohibit "patching the patch" with another nested conditional**.
- **Mandate stepping back to evaluate the architecture:** If a review bot points out an edge case caused by a speculative shim, the default response is to remove the shim and align with standard repository architecture rather than adding layers of defensive band-aids.

### Subagent Skill Viewing Gate (Strict Code Read/Write Rule)

A subagent views a general skill **IF AND ONLY IF** the assigned task requires code-related work — specifically reading, inspecting, analyzing, or writing code files (e.g. implementation, bug fixing, code auditing/review, or code-level scouting).

- **Code Read/Write Work:** Tell the subagent to read **exactly one** matching general skill from the inventory. Never tell a subagent to read two skills, and never tell a subagent to read an orchestrator-only skill.
- **Non-Code Work (NO SKILL VIEW):** If the subagent's task does NOT require reading or writing code files (for example: pure Git operations like branching/pushing/PR raising, polling CI/bots, web searching, conversational tasks, normal talk, or administrative checks), **NO skill view is needed or allowed**. Explicitly instruct the subagent not to view any skill. If a non-code task turns into a code fix — a bot or CI flagging a real defect that must be edited before it can be pushed — that is now a new code read/write task: dispatch a separate fixer subagent with the one matching general skill, and let the polling subagent stay read-only.

Only files named `skills.md` are skills. Anything else under `C:/Users/bari2/Desktop/SKILLS` is not loadable.

---

## Picking The Route

First, determine if the project is **Open-Source Contribution (OSS)** or **Saif's Personal Project** (always confirm if not explicit).

| Target Project | Work Type | Route |
| --- | --- | --- |
| Personal Project | Issue given by Saif | Personal Project Pipeline (see `Personal-Proj/skills.md`) |
| OSS Contribution | Small, trivial fix (1–5 lines) | Simple Path |
| OSS Contribution | Substantial GitHub issue | Full OSS Pipeline (Phases 1–5) |
| OSS Contribution | Brand new feature | Feature Path |

---

## OSS Contribution Routes (OSS Only — Strictly Forbidden in Personal Projects)

## Simple Path — small changes

1. **1 fixer subagent.** Exactly one general skill. It fixes, commits, and reports.
2. **1 review subagent** (`review`) if possible. Skipping is allowed only for genuinely trivial work, and you must state why you skipped it.
3. **PR raise + poll loop.**

## Feature Path — new features

No parallel discovery scouts. The builder maps the surface itself.

1. **1 build subagent** with `frontend` or `backend` — new features only.
2. **1 or more review subagents** (`review`), in parallel, 10/10 strict.
3. **Gate:** 0 must-fix and 0 should-fix. Findings go back to the builder, then re-audit.
4. **PR raise + poll loop.**

## Full OSS Pipeline — substantial issues (OSS Only — Strictly Forbidden in Personal Projects)

### Phase 1 — Issue Sourcing & Triage

Read `C:/Users/bari2/Desktop/SKILLS/open-issue-triage/skills.md` yourself.
- **Triage (`open-issue-triage`):** Ingest and systematically triage already-filed, open GitHub issues, filtering and scoring for high merge-ROI candidates.
- **Strict Prohibition:** Never read or use `open-issue-finding` here. Finding new codebase bugs is a completely separate standalone workflow and is strictly forbidden during the fix pipeline.
- **Issue Thread Discourse Ingestion:** Extract the complete architectural discourse from the GitHub issue thread: reporter's core friction, maintainer comments/labels (`type: Feature`, `enhancement`, `target: patch`), and community production insights (e.g. why naive fixes failed or opt-in defaults are needed).

### Phase 2 — Pre-Fix Investigation & Planning

Read `C:/Users/bari2/Desktop/SKILLS/open-pre-fix/skills.md` yourself.
- Produce a Minimal Reproducible Example (MRE) with deterministic, verified failure isolation.
- Isolate evidence-backed root cause and violated invariants.
- **Inspect Repository Precedent:** Inspect recent merged PRs in the target repo to see how similar capability flags, adapters, or configurations were designed, typed, and merged.
- **Primacy of Maintainer Intent & Discourse:** Document reporter core friction, maintainer commentary, labels, and community insights in the dossier.
- **Honest Classification Gate:** Determine conventional commit classification upfront (`feat` if introducing any public option/field/capability override, `fix` only for pure bug repairs). Never disguise a `feat` as a `fix`.
- **Architectural Purity (Zero Heuristic Shims):** Strictly forbid runtime sniffing or output-mapper coercions. Maintain clean abstraction boundaries; document storage migrations in JSDoc instead of hacking runtime transforms.
- Map complete blast radius: callers, dependencies, downstream consumers, configurations, and regression test suites.
- Establish architectural fix plan and partition work for 1, 2, or max 3 fixers with **strict disjoint file ownership**.
- **Write comprehensive end-to-end pre-fix findings** into a markdown file at `C:/Users/bari2/Desktop/better-auth-dossier/issue-context/<issue-id>.md` (one folder per-session, one file per-issue) for the fixing agent, detailing the full problem statement, root cause analysis, reproduction, invariants, blast radius, upstream discourse, precedent citations, surgical fix blueprint, AND the verbatim Scope Contract with every diff line traced.
- Package all findings into the **Phase 2 Context Packet** (referencing the dossier `.md` file). Do not advance to Phase 3 on partial context, missing MRE, or unmapped blast radius.
- Use multiple parallel subagents for investigation and mapping. Parallel use is mandatory — time is never a constraint, rushing is a disqualifier, thoroughness wins.

### Phase 3 — Fix

Dispatch 1, 2, or max 3 parallel fixers according to issue size. Partition work so **no two subagents own the same file** — split by module or strict file ownership, never overlapping.

Fixers must enforce clean abstraction boundaries: zero speculative shims, zero runtime sniffing coercions, and honest conventional commit prefixes (`feat` vs `fix`).

Every fixer prompt carries the **Phase 2 Context Packet** and references the comprehensive pre-fix findings `.md` file in `C:/Users/bari2/Desktop/better-auth-dossier/` so it does not rediscover anything. Note that `open-issue-fix` does NOT exist; fixers are subagents and read the matching general skill from the inventory:

| The work is | Skill |
| --- | --- |
| Existing backend, or existing frontend non-UI/UX work, or a pure logic/data bug | `issue-fix` |
| Existing frontend — UI/UX, visual, layout, motion, interaction | `ui-ux` |
| Brand new frontend feature | `frontend` |
| Brand new backend feature | `backend` |

`frontend` and `backend` are for new features only. Never for existing-surface work.

You decide `ui-ux` vs `issue-fix` per situation. Name the reason in the dispatch prompt so the subagent and the user both know why. When a change is genuinely both, pick the dominant one and say why.

**Phase 3 Scope Discipline Hard Rule (applies to every fixer):**
- Every fixer prompt must carry the **Scope Discipline Confirmation** checklist (see `open-pre-fix/skills.md` §4 dispatch template):
  - Every planned change traces to a Scope Contract line.
  - Zero new public surface (options, fields, methods, types) unless explicitly requested.
  - Zero core schema change unless explicitly authorized.
  - No adjacent refactors, formatting sweeps, or opportunistic cleanups.
  - Diff is the minimum required to fix the reported issue.
- OSS maintainers strictly prefer short, minimal, on-point, surgical diffs. The smallest correct diff is the only acceptable diff. Phase 4 audit will reject verbose, speculative, or inventively broad diffs as `should-fix`.
- Fixers must commit frequently with conventional commit prefixes (`fix:` vs `feat:`). Disguising a `feat` as a `fix` because the issue is filed as a bug is a violation — re-classify honestly.

### Phase 4 — Post-Fix Multi-Lens Audit

Read `C:/Users/bari2/Desktop/SKILLS/open-post-fix/skills.md` yourself.

- **First Pass (Initial Full Audit):** Dispatch **7 parallel review subagents**, one for each lens (each reads `review` and is strictly **read-only**):
  - **Lens A: Security Audit** (vulnerabilities, input sanitization, auth boundaries, secrets, trust zones)
  - **Lens B: Blast Radius & Regressions** (unintended side effects, call-site drift, downstream breakage, zero runtime sniffing shims)
  - **Lens C: Performance & Resource Efficiency** (algorithmic regressions, allocations, leaks, I/O bottlenecks)
  - **Lens D: Concurrency, State & Invariants** (race conditions, deadlocks, atomicity, state mutation ordering)
  - **Lens E: Idiomatic Style & Repo Conventions** (repo-native idiom, typing consistency, precedent adherence, clean abstraction boundaries, JSDoc migration docs instead of runtime hacks)
  - **Lens F: Test Coverage & Edge Cases** (MRE test assertion, edge case coverage, boundary conditions, determinism)
  - **Lens G: Scope Alignment & Scope Drift** (DEDICATED — line-level trace to Scope Contract, public surface invention, core schema lock, adjacent refactor detection, maintainer taste gate. Prime catch for OSS scope-drift issues)
- Zero forgiveness. 10/10 strict. Hunt for even basic minor issues. Assume the work is wrong until proven otherwise.
- Tag findings strictly as `must-fix`, `should-fix`, or `nitpick`. Scope drift findings from Lens G are `must-fix` by default — there is no "minor scope drift" category.
- **Gate: 0 must-fix AND 0 should-fix.** Any finding at either level sends the work back to Phase 3 fixers with the findings attached, then re-audits.
- **Re-Audit Rule:** Subsequent re-audits do **NOT** re-run all 7 lenses! Once the initial 7-lens pass is complete, subsequent review passes are small, focused, targeted audits restricted strictly to the modified files and resolved findings. **At least one of the re-audit reviewers (min 3) MUST be a scope-alignment reviewer** to confirm the fixer did not re-introduce scope drift while addressing previous findings.
- Use multiple parallel reviewers for re-audits as well. Parallel use is mandatory — single reviewer never passes gate.
- Never lower the bar to pass the gate. Never wave through a should-fix because the PR is late. Time is never a constraint in OSS.

### Phase 5 — PR Raise & Poll

Read `C:/Users/bari2/Desktop/SKILLS/open-pre-pr/skills.md` yourself.
- Run pre-PR hard verification gates before drafting or opening a PR. Sequential multi-pass verification across distinct roles is required — forbid relying on a single unverified subagent.
- Raise PR as **Draft** (`gh pr create --draft`) while running stabilization checks so maintainers never see an incomplete or iterating state.
- Ensure PR title strictly adheres to conventional commit classification (`feat` vs `fix`) based on repo precedent and scope.
- Ensure PR description addresses the full issue thread context, reporter friction, and upstream design consensus.
- **Pre-PR Scope-Alignment Gate**: Before raising the PR, the verification subagent MUST independently confirm zero out-of-scope diff lines by re-checking every line of the diff against the Scope Contract in the dossier. Any untraced line = FAIL the pre-PR gate, route back to the fixer.
- **Changeset Quality**: When the target repo uses changesets (`.changeset/*.md`), the changeset must **clearly explain in short what the PR does** — maintainers read it first to decide whether to dig into the diff. Treat it as a micro-abstract of the PR, not a footnote.
  - **Header**: Conventional commit-style title reflecting the scope, e.g. `fix(two-factor): prevent TOTP code replay on verification (RFC 6238 §5.2)`. Honest semver bump (`patch` / `minor` / `major`) — never inflate, never deflate.
  - **Body**: 1–3 short sentences that answer, in this order:
    1. **What** the change does (one sentence, plain language, no jargon).
    2. **Why** it does it (the symptom or requirement it addresses; cite the issue number and any relevant RFC/spec).
    3. **Blast radius note** (only if non-trivial: which package(s) affected, whether breaking, whether new public surface).
  - **Anti-patterns to avoid**: vague summaries ("misc fixes", "improvements", "various changes"), buried scope (the lead sentence hides what actually changed), missing semver level, missing issue reference, scope mismatch with the PR diff (changeset claims X but the diff does Y).
  - **Maintainer-authored bar**: Your example changeset `Discord profile callbacks now accept identities without optional user metadata. Missing email verification defaults to 'false'; profile mappings must handle omitted metadata.` is the floor, not the ceiling — it states what changed, the new default, and the downstream implication in plain English. Match this readability standard.
- Monitor CI & review-bot convergence loop (1.5 min interval, max 10 push updates, never merge).
- Enforce the **Review Bot Anti-Whac-A-Mole Rule**: if a review bot flags an edge case caused by an ad-hoc heuristic, strictly forbid "patching the patch" with another conditional; step back and eliminate the shim.
- Mark PR as ready for review (`gh pr ready`) only when CI and review bots reach full green convergence.
- Only entered after the Phase 4 gate is 100% clean.
- **Cleanup after Phase 5:** Once the PR is raised and the convergence loop completes (or CI & review bots reach green convergence), **delete the pre-fix dossier `.md` file** in `C:/Users/bari2/Desktop/better-auth-dossier/` as its work is done.

---

## PR Raise & Poll Loop

You never push, never raise a PR, never merge. You dispatch subagents to own verification and the PR loop.

1. Read the pre-PR skill yourself — `C:/Users/bari2/Desktop/SKILLS/open-pre-pr/skills.md`. That skill holds the pipeline.
2. You cannot hand that skill to the subagent. **Write its relevant steps into the subagent's prompt yourself**, along with the loop rules below.
3. **Sequential Multi-Pass Verification:** Run a dedicated verification subagent for pre-PR hard gates before opening the PR. Never rely on a single agent.
4. **Draft PR by Default:** The subagent raises the PR as Draft (`gh pr create --draft`) to protect upstream maintainers from observing in-flight stabilization oscillations. The PR title must honestly reflect classification (`feat:` if adding public options/fields/capabilities, `fix:` only for bug fixes). The body must cite the full issue thread discourse and upstream design consensus.
5. The subagent polls CI and review bots every **1.5 minutes**.
6. On each poll it triages: **major, critical, or app-breaking must be fixed and pushed.** If a fix requires a code edit, it reports the finding back to me and I dispatch a fixer subagent to make it — the polling subagent never edits code itself. Trivial or cosmetic nits may be left alone — do not burn a round on them.
7. **Anti-Whac-A-Mole Rule:** When an AI review bot (CodeRabbit, Greptile, cubic) raises an issue with an ad-hoc heuristic or speculative transform, **strictly prohibit patching the patch with another nested conditional**. Step back and evaluate the architecture: remove the shim and align with standard repository architecture.
8. It pushes the fix and resumes polling. Hard cap: **10 push-updates.** After the 10th it stops and reports to the user what is still red. No infinite loops, no force-pushing.
9. Done means **CI fully green** and review bots effectively green. Only then mark the PR as ready (`gh pr ready <pr-number>`).
10. **Never merge — not you, not the subagent.** Leave the PR for the project owner to review and merge.
11. **Dossier cleanup:** After the PR is raised and the poll loop finishes, delete the pre-fix findings `.md` file in `C:/Users/bari2/Desktop/better-auth-dossier/` since its work is done.

---

## Dispatch Template

Every dispatch prompt carries all of these:

1. The task, scoped tightly enough for one subagent to finish alone.
2. Skill instruction: If and only if the subagent's task requires reading or writing code files, provide the exact absolute path of the **one** general skill to read. If the task is non-code work (e.g. pure Git operations, PR polling, web search, normal talk), explicitly state that no skill view is needed.
3. The repo path.
4. The full context packet — Phase 3 onward, never send a fixer in blind (packet produced in Phase 2).
5. The scope boundary: which files this subagent owns, and which are off-limits.
6. Stop and report instead of guessing on any ambiguity.
7. For a PR subagent, additionally: the pasted pre-PR steps, the 1.5 minute interval, the 10 push cap, the fix/tolerate triage split, and the explicit instruction never to merge.

Every dispatch also carries the shared-workspace rules, because the user runs several agents at once on different branches:

- Work only on your assigned branch. Never switch, checkout, rebase, merge, reset, revert, or clean another agent's work.
- Touch only the files this task owns. Anything unfamiliar in the tree belongs to another agent.
- **Never `git stash`.** Not to save work, not to unblock yourself, not ever.
- Dirty tree you did not create → leave it and report it to the user. Do not repair it.
- Before committing, confirm staged files are only yours. Never sweep up paths you did not touch.

When you parallelise, the file-ownership split planned in Phase 2 and executed in Phase 3 is what keeps agents off each other. Parallel subagents that share a file will collide — split the work, never the intent.

If two branches are ever going to be live at once, they belong in separate worktrees or clones, not one shared checkout. Git has one HEAD per working copy, so two agents cannot hold two branches in the same directory.

---

## Before Every Dispatch (OSS)

- Right route for the issue size?
- Does the subagent's task require reading or writing code files? If yes, assigned exactly one general skill with full absolute path. If no (Git ops, PR polling, web search, normal talk), confirmed NO skill view is assigned.
- Task scoped so one subagent can finish it?
- No two parallel subagents own the same file?
- Fixers carry the full context packet from Phase 2?
- Relevant canonical open-* orchestrator skill consulted for the active phase (`open-issue-finding`, `open-issue-triage`, `open-pre-fix`, `open-post-fix`, `open-pre-pr`)?
- Repository precedent inspected and conventional classification confirmed (`feat` vs `fix`)?
- Architectural purity verified: zero leaky heuristic shims or runtime sniffing transforms?
- Full issue thread discourse, maintainer intent, and labels captured?
- Multi-agent redundancy respected: no rushing, no single subagent for critical phases?

---

## Hard Nevers (OSS)

- Never use the 5-Phase OSS pipeline or read `open-*` skills when working on personal projects.
- Never read `open-issue-finding` when working on the 5-Phase Issue Fix Pipeline (Phases 1–5).
- Never read `open-issue-triage`, `open-pre-fix`, `open-post-fix`, or `open-pre-pr` when working on standalone Issue Finding / Codebase Bug Hunting.
- Never rush: time is never a constraint in open-source contributions. Rushing is a disqualifier.
- Never rely on a single subagent for critical phases (pre-fix mapping, audit, and PR verification).
- Never disguise a `feat` as a `fix` if a change introduces a new public configuration field, option, or capability override.
- Never introduce runtime sniffing, heuristic mappers, or speculative backward-compatibility shims that risk data corruption on edge-case inputs; document migration requirements in JSDoc instead.
- Never "patch the patch" or add nested conditionals in response to review bot findings on ad-hoc heuristics — step back and remove the shim.
- Never open a non-draft PR while CI/review-bot stabilization checks are still iterating — use `gh pr create --draft`.
- Never implement anything yourself.
- Never let a subagent `git stash`, switch branches, or touch files outside its ownership boundary.
- Never leave the pre-fix findings `.md` file in `C:/Users/bari2/Desktop/better-auth-dossier/` undeleted after Phase 5 is finished.
- Never let a fixer invent new public options, fields, methods, or exported types the issue did not request. 101% scope alignment or it does not ship.
- Never let a fixer touch core schema (DB tables, migration files, exported type definitions, public serialization shapes, stable API contracts) without explicit Scope Contract or maintainer authorization.
- Never let a fixer bundle an unrelated refactor, formatting sweep, opportunistic cleanup, or dependency bump into a scoped fix — split into its own issue.