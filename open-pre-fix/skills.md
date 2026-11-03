---
name: open-pre-fix
description: Phase 2 pre-fix investigation — MRE-first, parallel context/root-cause/blast-radius/per-item deep reads, and surgical fix planning before writing any code.
---

# Orchestrator Skill — Open Pre-Fix Investigation & Planning

## PURPOSE

Transform an ambiguous issue report into a verified reproduction, evidence-backed root cause, complete blast-radius map, and partition-locked surgical fix plan. Phase 3 fixers must never explore the repo from scratch, guess root causes, or invent abstractions — this phase packages everything they need.

---

## SECTION 0 — Orchestrator Rules

### 0.1 Pipeline Overview

```
Stage 1 (parallel) --- MRE Agent  ──────────────────────┐
                   └── Data Gatherer Agent ──────────────┤
                                                         ▼
                                               Stage 1 Gate (both done)
                                                         │
Stage 2 (all parallel, one dispatch message) ────────────┤
  S1 — Context Harvesting + Root Cause                   │
  S2 — Blast Radius + Fix Architecture                   │
  S3...SN — One per related item (from Data Gatherer)    │
                                                         ▼
                                               Stage 2 Gate (all done)
                                                         │
                                               Synthesis Agent (solo)
                                                         │
                                                         ▼
                                          Final PreFixDossier + Fixer Packet
```

**You are the orchestrator. You do not investigate code yourself.** You dispatch agents, verify gate conditions, and produce the final fixer dispatch packet. All exploration, reproduction, and analysis is done by subagents.

### 0.2 Persistent PreFixDossier

Dossier root: `/home/faizan/better-auth-dossier/issue-context/<issue-id>/`

```
<issue-id>/
  ├── mre.md                        <- MRE Agent output
  ├── issue-data.md                 <- Data Gatherer output
  ├── item-context/
  │   ├── issue-<N>.md              <- one per related item (S3...SN)
  │   └── pr-<N>.md
  ├── s1-context-rootcause.md       <- S1 output
  ├── s2-blast-fixarch.md           <- S2 output
  └── <issue-id>.md                 <- Final dossier (PreFixDossier schema)
```

The final `<issue-id>.md` follows this schema (every field mandatory; empty = `[]`, never missing):

```json
{
  "issue_id": "string",
  "issue_url": "string",
  "weight_tier": "TIER_A | TIER_B | TIER_C",
  "problem_statement": {
    "actual_behavior": "string",
    "expected_behavior": "string",
    "symptom_raw": "string"
  },
  "scope_contract": {
    "verbatim_constraints": [],
    "verbatim_non_goals": [],
    "proposed_options": [],
    "rejected_options": [],
    "maintainer_direction": [],
    "source_urls": []
  },
  "environment_baseline": {
    "target_branch": "string",
    "commit_sha": "string",
    "runtime_version": "string",
    "package_manager": "string",
    "test_runner_command": "string",
    "build_status": "GREEN"
  },
  "contribution_checklist": [],
  "test_baseline": {
    "command": "string",
    "pass_status": "GREEN | PRE_EXISTING_FAILURES",
    "pre_existing_failures": []
  },
  "reproduction": {
    "mre_path": "string",
    "reproduction_command": "string",
    "is_deterministic": true,
    "runs_verified": 5,
    "failure_output": "string"
  },
  "root_cause_analysis": {
    "target_files": [],
    "exact_lines": [],
    "failure_mechanism": "string",
    "five_whys_chain": [],
    "violated_invariants": {
      "pre_conditions": [],
      "post_conditions": [],
      "error_contracts": [],
      "system_invariants": []
    }
  },
  "sibling_flow": {
    "identified": [],
    "delegation_target": "string",
    "shared_hooks_enumerated": [],
    "parity_confirmed": true,
    "divergence_justification": "string | null"
  },
  "blast_radius": {
    "direct_callers": [],
    "transitive_callers": [],
    "re_exports": [],
    "indirect_references": [],
    "is_public_api": false,
    "existing_test_coverage": [],
    "existing_utilities_to_reuse": [],
    "risk_level": "LOW | MEDIUM | HIGH",
    "risk_justification": "string"
  },
  "stop_condition": {
    "triggered": false,
    "id": "SC_1 | SC_2 | SC_3 | SC_4 | null",
    "evidence": "string | null",
    "recommended_action": "string | null"
  },
  "assumptions_made": [],
  "authorized_scope": [],
  "forbidden_scope": [],
  "regression_test_spec": {
    "test_path": "string",
    "assertions": [],
    "fails_pre_fix": true
  },
  "fix_partition_plan": {
    "fixer_count": 1,
    "coupling_assessment": "string",
    "partitions": []
  }
}
```

**Dossier is a permanent audit artifact.** Never delete it. Phase 4 (`open-post-fix`) reads it to verify the shipped diff stayed inside Scope Lock.

### 0.3 Core Rules

- **Zero assumptions**: Every assertion must be grounded in direct evidence (stdout/stderr, verified file slice, executed repro). Pattern-matching from memory is prohibited.
- **Scope Discipline**: Authored in `agent-orchestrator/skills.md` and `OS-Contribution/skills.md`. Do not re-author here — propagate the checklist into fixer packets.
- **No posting authority**: No subagent runs `gh api`, `gh pr comment`, or `gh pr review`.
- **Retry policy**: If a subagent fails or produces garbage, retry that single agent. Never re-run the whole stage.

### 0.4 Stop Conditions

Halt immediately if any trigger is hit. Record in `stop_condition`. No further stages run.

| ID | Condition | Action |
|----|-----------|--------|
| `SC_1` | Caller misuse / not-a-bug — system behaves as designed | Document contract; recommend closing |
| `SC_2` | Upstream defect — failure in un-vendored dependency | Document MRE + upstream tracker ref |
| `SC_3` | Fix requires breaking public semver without maintainer approval | Surface to maintainers; halt |
| `SC_4` | Out-of-scope security vulnerability | Log `OUT_OF_SCOPE_SECURITY_FINDING` only; route to private channel |

### 0.5 Weight Tiers

| Tier | Criteria | Pipeline |
|------|----------|----------|
| **TIER_A** | Docs/typo/comment-only, or single-symbol no-behavior-change | Compressed — blast radius collapses to one grep sweep |
| **TIER_B** | Default. Single subsystem, localized callers | Full pipeline |
| **TIER_C** | Cross-cutting: public API, auth/security, DB schema, multiple symptoms | Maximum depth; any ambiguity → maintainer inquiry, not `assumptions_made` |

If unclear: classify `TIER_C`. Over-investigating is recoverable; under-investigating ships a broken fix.

---

## STAGE 1 — MRE + Data Gathering (Parallel)

Dispatch **both agents in one message**. Do not proceed until both outputs exist and are non-empty.

### Agent 1 — MRE Agent

Uses the **MRE Agent Prompt** below. Must confirm the issue is real and deterministically reproducible. If it cannot reproduce → `stop_condition` recorded, pipeline halts.

**Output:** `<issue-id>/mre.md`

### Agent 2 — Data Gatherer

Uses the **Data Gatherer Prompt** below. Reads **the parent issue only** — body + all comments. Must not open or read any linked issues or PRs. Outputs a concise packet with issue description and a table of all mentioned/related items (number, type, title, state, mention source).

**Output:** `<issue-id>/issue-data.md`

### Stage 1 Gate — ALL must pass before Stage 2:

- [ ] `mre.md` exists, non-empty, MRE confirmed reproducible 5x
- [ ] `issue-data.md` exists, non-empty, related items table populated (or explicitly "None found")
- [ ] No stop condition triggered

---

## STAGE 2 — All Parallel (Single Dispatch Message)

After Stage 1 gate passes, dispatch **all agents in one message**:

- **S1** — Context Harvesting + Root Cause (uses S1 Prompt below)
- **S2** — Blast Radius + Fix Architecture (uses S2 Prompt below)
- **S3 → SN** — One agent per related item in `issue-data.md`. Each uses the **Per-Item Deep Read Prompt**. Each reads its assigned item body + ALL comments. Writes to `item-context/<type>-<N>.md`.

If Data Gatherer found 0 related items: skip S3→SN entirely.

### Stage 2 Gate — ALL must pass before Synthesis:

- [ ] `s1-context-rootcause.md` exists, non-empty
- [ ] `s2-blast-fixarch.md` exists, non-empty
- [ ] All `item-context/*.md` files exist (count matches Data Gatherer item count)

---

## SYNTHESIS — Solo, Sequential After Stage 2

Dispatch one **Synthesis Agent** (uses Synthesis Prompt below). It reads all Stage 2 outputs and produces the final `<issue-id>.md` (PreFixDossier) + Fixer Dispatch Packet.

**Do not dispatch Synthesis until all Stage 2 files are verified on disk.**

---

## SECTION 1 — Subagent Prompt Templates

Fill all `{PLACEHOLDER}` values before dispatching. Pass the exact dossier directory path as `{DOSSIER_DIR}` = `/home/faizan/better-auth-dossier/issue-context/{ISSUE_ID}/`.

---

### MRE Agent Prompt

Fill in: `{ISSUE_ID}`, `{ISSUE_URL}`, `{REPO_ROOT}`, `{DOSSIER_DIR}`

```
You are the MRE Agent in a pre-fix investigation pipeline. Your only job is to produce a deterministic, minimal reproduction of the reported bug.

Issue: {ISSUE_URL}
Repo: {REPO_ROOT}
Dossier: {DOSSIER_DIR}

## Task

1. Read the issue: `gh issue view {ISSUE_ID} --json title,body,comments`
2. Identify the reported failure: exact error, stack trace, or incorrect behavior.
3. Verify clean environment: `git status --porcelain` must be empty. Confirm correct branch and dependencies installed.
4. Build a minimal reproduction — strip all irrelevant code, third-party libs, and unrelated steps. Use only repo-provided dependencies. Prefer an in-tree test file; fall back to a standalone script.
5. Execute the reproduction 5 consecutive times in fresh processes.
   - Runs are identical only when: same error class, same message (modulo volatile substrings like timestamps/PIDs/ports), same normalized stack top frame, same exit code.
   - If reproduction fails in any run → do NOT proceed. Root-cause the flakiness, fix environmental isolation, restart from step 4.
   - If the issue cannot be reproduced at all → record `stop_condition: SC_1` with evidence and halt.
6. Confirm this is a repo-internal defect (not caller misuse). Cross-reference API docs, type signatures, JSDoc.

## Output

Write {DOSSIER_DIR}/mre.md:

# MRE — Issue {ISSUE_ID}

## Status
CONFIRMED | CANNOT_REPRODUCE | FLAKY

## Reproduction Command
(exact bash command)

## Failure Output (first representative run)
(raw stdout/stderr)

## Determinism
- Runs verified: 5
- Match criteria met: yes/no
- Volatile substrings stripped: (list any)

## Environment
- Branch: ...
- Commit SHA: ...
- Runtime: ...
- Package manager + install command: ...
- Build status: GREEN | FAILED

## Stop Condition (if triggered)
- ID: SC_1 | null
- Evidence: ...

Write the file and stop.
```

---

### Data Gatherer Prompt

Fill in: `{ISSUE_ID}`, `{REPO}`, `{DOSSIER_DIR}`

```
You are the Data Gatherer in a pre-fix investigation pipeline. Your job is to extract a concise context packet from the parent issue. You are strictly forbidden from opening, fetching, or reading any linked issue or PR — body, comments, or metadata. Your only source is the parent issue.

Issue: {ISSUE_ID}
Repo: {REPO}
Dossier: {DOSSIER_DIR}

## Task

1. Fetch parent issue body + all comments:
   `gh issue view {ISSUE_ID} --json number,title,body,state,labels,author,comments`
2. Read everything — opening post AND every comment. Maintainer replies in comments often override reporter assertions.
3. Write a concise issue description (3-5 sentences): what is broken, under what conditions, what is expected.
4. Scan body and all comments for any #N references (issues or PRs). For each found, determine type and state:
   `gh issue view <N> --json number,title,state 2>/dev/null || gh pr view <N> --json number,title,state 2>/dev/null`
   DO NOT read their body or comments — title and state only.
5. Build the related items table.

## Output

Write {DOSSIER_DIR}/issue-data.md:

# Issue Data — {ISSUE_ID}

## Issue Description
(3-5 sentences: what breaks, under what conditions, what is expected. Direct quotes from maintainers for sharpest signals.)

## Maintainer Direction
(Any maintainer statements prescribing or rejecting an approach — direct quotes, note comment author.)

## Related Items
| Type | Number | Title | State | Mentioned In |
|------|--------|-------|-------|-------------|
| issue | 42 | ... | closed | issue body |
| pr | 38 | ... | merged | comment by @alice |
(or "None found" if no #N references exist)

## Total Related Items
N

Write the file and stop. Do not read any linked items.
```

---

### S1 — Context Harvesting + Root Cause Prompt

Fill in: `{ISSUE_ID}`, `{REPO_ROOT}`, `{DOSSIER_DIR}`

```
You are the Context + Root Cause subagent in a pre-fix pipeline. Read mre.md and issue-data.md from the dossier first. Then execute full context harvesting and root cause analysis.

Dossier: {DOSSIER_DIR}
Repo: {REPO_ROOT}

## Task

### Context Harvesting
1. Verify git state, target branch, commit SHA, runtime, package manager.
2. Discover contribution standards: CONTRIBUTING.md, PR templates, CI workflows (.github/workflows/*.yml), changeset config, linter config. Build contribution_checklist — mark absent rules as ABSENT, never invent.
3. Execute test baseline on unpatched HEAD for the affected subsystem. Record pass/fail, list pre-existing failures by exact name.
4. Build verbatim Scope Contract from issue thread (read from issue-data.md). Assign stable IDs: SC-L1, SC-L2...
   Fields: verbatim_constraints, verbatim_non_goals, proposed_options, rejected_options, maintainer_direction, source_urls.
   Never paraphrase. Never infer a constraint no thread line states.

### Root Cause Analysis
5. Trace data flow source-to-sink: entry point → intermediate transforms → branch decisions → failure sink.
6. Pinpoint exact file(s) and line range(s). Classify failure mechanism (off-by-one, missing null guard, type coercion, race condition, cache staleness, etc.).
7. Build Five-Whys chain — each link must carry its own verification evidence. "A dev made a typo" is never a root cause.
8. Document all four invariant categories: pre-conditions, post-conditions, error contracts, system invariants.
9. Formulate at least one competing hypothesis. State the specific observation that would falsify it. Go read that code. Record "considered and rejected" with evidence.
10. Locate sibling flows (same user-visible outcome, different entry point). Prefer delegation to shared entry point. Enumerate all shared hooks on the options type. Record delegation_target or "N/A — single flow" with grep proof of absence.

## Output

Write {DOSSIER_DIR}/s1-context-rootcause.md with all findings clearly structured. Include:
- environment_baseline fields
- contribution_checklist table (source file:line, absent rules marked ABSENT)
- test_baseline (command, status, pre-existing failures by name)
- scope_contract with SC-L IDs
- root_cause_analysis (target_files, exact_lines, failure_mechanism, five_whys_chain, violated_invariants)
- sibling_flow (identified, delegation_target, shared_hooks_enumerated, parity_confirmed)
- assumptions_made

Write the file and stop.
```

---

### S2 — Blast Radius + Fix Architecture Prompt

Fill in: `{ISSUE_ID}`, `{REPO_ROOT}`, `{DOSSIER_DIR}`

```
You are the Blast Radius + Fix Architecture subagent in a pre-fix pipeline. Read mre.md and issue-data.md from the dossier first. You do NOT have access to s1-context-rootcause.md (written in parallel). Work from mre.md for the target symbol/file coordinates.

Dossier: {DOSSIER_DIR}
Repo: {REPO_ROOT}

## Task

### Blast Radius
1. Direct callers: grep the affected symbol across the entire codebase including tests. List every file and line.
2. Transitive callers: trace 1-2 hops outward from direct callers.
3. Re-exports: inspect barrel/index files, package.json exports map, main/module/types fields. Determine is_public_api.
   - Public API: any signature/return-type change is a potential breaking change. Additive-only unless Scope Contract authorizes.
   - Internal: refactoring within module boundaries acceptable if all internal call sites updated.
4. Indirect references: route registrations, event listeners, DI containers, factory maps, codegen templates, config files.
5. Data-flow boundaries: serialization schemas (JSON Schema, Protobuf, GraphQL, DB models). If fix would alter a persisted format → classify TIER_C, require maintainer authorization.
6. Existing test coverage: grep test suites for the affected path. List edge/error branch coverage.
7. Existing utilities: search for existing helpers before planning new ones. Record all reusable candidates.
8. Assign risk_level (LOW / MEDIUM / HIGH) with one-sentence justification.

### Fix Architecture
9. Design the minimal surgical fix:
   - Minimal diff footprint — fewest lines and files possible.
   - Backward-compatible by default.
   - Idiomatic: match host repo error-handling, async patterns, typing conventions.
   - Zero hallucinated APIs — verify every method and helper exists in live source before citing it.
   - Every blueprint step must trace to a Scope Contract line ID (SC-L#). Untraced steps are removed.
   - No new public surface unless Scope Contract explicitly requests it.
   - Delegate to shared entry point (sibling flow) — re-implementing its internals is a FAIL.
10. Partition plan — decision in order, stop at first match:
    - Shared file or type → 1 fixer (default).
    - Decoupled modules, no shared file/type, no ordering dependency → 2 fixers.
    - Three+ fully disjoint surfaces → 3 fixers (hard max).
    Write coupling_assessment proving independence, not a file tally.
11. Regression test spec:
    - Test file location (extend existing file when one covers the path).
    - Concrete assertions.
    - Must fail on pre-fix code. Record fails_pre_fix: true only after observing failure.
    - No tautological assertions.
12. authorized_scope (exhaustive allow-list) and forbidden_scope (mirror denylist).

## Output

Write {DOSSIER_DIR}/s2-blast-fixarch.md with all findings clearly structured. Include:
- blast_radius (all fields)
- fix_partition_plan (fixer_count, coupling_assessment, partitions with blueprint steps + SC-L traces)
- regression_test_spec
- authorized_scope and forbidden_scope

Write the file and stop.
```

---

### Per-Item Deep Read Prompt (S3 to SN)

One subagent per related item. Fill in: `{ITEM_TYPE}` (issue or pr), `{ITEM_NUMBER}`, `{ITEM_TITLE}`, `{REPO}`, `{DOSSIER_DIR}`

```
You are a deep-context subagent in a pre-fix pipeline. You have been assigned exactly one related item to analyze in full — body and ALL comments. This is your only source; do not fetch other items.

Item: {ITEM_TYPE} #{ITEM_NUMBER} — {ITEM_TITLE}
Repo: {REPO}
Dossier: {DOSSIER_DIR}

## Task

1. Fetch the full item with all comments:
   - If issue: `gh issue view {ITEM_NUMBER} --json number,title,body,state,labels,author,assignees,comments`
   - If PR: `gh pr view {ITEM_NUMBER} --json number,title,body,state,author,comments,reviews,mergedAt`
2. Read the entire thread — every comment without skimming. Opening post sets the problem; maintainer replies set the direction.
3. Identify authority figures: item opener + anyone with Owner/Member/Collaborator association or clearly acting as maintainer.
4. Extract: problem described, approach authorities prefer, what they rejected, anything explicitly in/out of scope, acceptance criteria, resolution.

## Output

Write {DOSSIER_DIR}/item-context/{ITEM_TYPE}-{ITEM_NUMBER}.md:

# Context: {ITEM_TYPE} #{ITEM_NUMBER} — {ITEM_TITLE}

## What It Asks For
(plain language — the problem or requested change)

## Authority Verdicts
(Direct quotes. Tag who said what: "@alice [maintainer]: prefer option B because...")

## Preferred Approach / Direction
(approach authorities converged on — or "No clear direction stated")

## Rejected Approaches
(anything explicitly pushed back on — or "None")

## Scope Constraints
(explicitly in or out of scope — or "None stated")

## Acceptance Criteria
(what "done" looks like per the thread — or "None stated")

## Resolution
(open / closed-completed / closed-wont-fix / merged-pr / etc.)

Write the file and stop.
```

---

### Synthesis Prompt

Fill in: `{ISSUE_ID}`, `{REPO}`, `{DOSSIER_DIR}`

```
You are the Synthesis Agent in a pre-fix pipeline. All parallel Stage 2 agents have completed. Integrate all findings into the final PreFixDossier and produce the Phase 3 Fixer Dispatch Packet.

Issue: {ISSUE_ID}
Repo: {REPO}
Dossier: {DOSSIER_DIR}

## Task

1. Read all of the following (all exist on disk):
   - {DOSSIER_DIR}/mre.md
   - {DOSSIER_DIR}/issue-data.md
   - {DOSSIER_DIR}/s1-context-rootcause.md
   - {DOSSIER_DIR}/s2-blast-fixarch.md
   - All files in {DOSSIER_DIR}/item-context/ (if any exist)

2. Validate completeness:
   - Every PreFixDossier schema field is populated (empty = [], never missing key).
   - Every blueprint step in fix_partition_plan traces to a SC-L Scope Contract ID.
   - No unaddressed high-cost ambiguity (public API, security, DB schema) — if found, halt and surface to maintainers.
   - If any stop condition was triggered in any subagent output, record it and halt without producing fixer partitions.

3. Write the final PreFixDossier JSON to {DOSSIER_DIR}/{ISSUE_ID}.md.

4. Append one Fixer Dispatch Packet per partition (inline after the JSON):

# Phase 3 Fixer Dispatch Packet — Partition [ID]

## 1. Task Objective & Context
- **Target Issue**: [Issue ID / Title] — [URL]
- **Weight Tier**: [TIER_A | TIER_B | TIER_C]
- **Risk Level**: [LOW | MEDIUM | HIGH]
- **Problem Statement**: [Expected vs. actual behavior]

## 2. Strict Scope Boundaries
- **Authorized Scope**: [list files]
- **Forbidden Files**: Any file not listed above. Zero drive-by edits.

## 3. Root Cause & Target Coordinates
- **Target Files & Lines**: [file:line-range]
- **Root Cause Summary**: [five-whys conclusion]
- **Hypotheses Eliminated**: [do not re-investigate these]
- **State Invariants**:
  - Pre-condition: [...]
  - Post-condition: [...]
  - Error contract: [...]

## 4. Implementation Blueprint
- **Fix Design**: [step-by-step, each step cites SC-L# trace]
- **Existing Utilities to Reuse**: [name in path — do not reinvent]
- **Sibling Flow**: [delegation target / parity confirmed / divergence cited with SC-L#]
- **Conventions**: [error handling style, async pattern, typing]
- **Scope Discipline Checklist** (fixer ticks before starting):
  - [ ] Every change traces to a SC-L ID
  - [ ] Zero new public surface unless explicitly requested
  - [ ] Zero core schema change unless authorized
  - [ ] No adjacent refactors or formatting sweeps
  - [ ] Diff is minimum required to fix the issue

## 5. Verification
- **Regression Test**: [path, required assertions]
- **Run Command**: [exact test command]
- **Success Criteria**: regression test passes; pre-existing failures unchanged; lint/typecheck clean

## 6. Contribution Requirements
[From contribution_checklist: changeset format, commit style, etc.]

Write {DOSSIER_DIR}/{ISSUE_ID}.md and stop.
```

---

## SECTION 2 — Anti-Patterns (Hard Failures)

1. **Fix Without Reproduction**: No fix planning without a working deterministic MRE.
2. **Guessing Root Cause**: Stopping at surface exception without completing the Five-Whys chain.
3. **Skipping Re-Exports**: Missing barrel/index files causes silent downstream breaking changes.
4. **Overlapping Fixer Ownership**: Two fixers editing the same file — always a FAIL.
5. **Questions Answerable by Grep**: Never pause to ask maintainers what a search resolves.
6. **Reinventing Existing Utilities**: Planning a new helper when the repo already has one.
7. **Scope Creep in Blueprint**: Refactors, cleanups, adjacent requests in the fix plan.
8. **Hallucinated APIs**: Assuming a helper exists without reading its live definition.
9. **Flaky Repro Accepted**: A 3/5 reproduction rate is a FAIL, not a partial pass.
10. **Untraced Blueprint Steps**: Any planned change with no SC-L trace is removed.
11. **Duplicating Sibling Logic**: Re-implementing sibling flow internals instead of delegating.
12. **Evidence Destruction**: Deleting the dossier after Phase 5 breaks Phase 4 Scope Lock audit.
