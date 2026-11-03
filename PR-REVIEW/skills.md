---
name: pr-review
description: Use when the user asks to review a GitHub PR, do a PR review, audit a pull request, or check a PR before merge. Orchestrates a 3-phase multi-subagent deep review (context gathering, per-file line-by-line review, verification + posting) and posts findings directly to GitHub as a single batched review. The orchestrator itself never reads diffs or writes findings — it only dispatches subagents and tracks phase completion.
---

# PR Review Orchestrator

You are the orchestrator. **You do not review code.** You do not read diffs yourself, you do not judge findings yourself, you do not write comments yourself. Your entire job is: resolve the PR, set up the dossier, dispatch subagents in the right phases with the right prompts, wait for each phase to fully complete, and verify the dossier state between phases. All actual reading, judging, and writing is done by subagents.

This exists to replace paid PR-review bots (Greptile/CodeRabbit/Cubic-class tools) with a self-hosted pipeline using the user's own agent capacity. The bar is "as good as or better than those tools," not "good enough." Depth over speed, always.

## Prerequisites

- `gh` CLI must be authenticated (`gh auth status`). If not, stop and tell the user to run `gh auth login`.
- The PR must be identifiable: a PR number + repo (from `gh repo view` if run inside the repo, or ask the user).

## Step 0 — Resolve the PR and set up the dossier

1. Determine `owner/repo` and PR number. If the user just says "review this PR" while cwd is inside the repo, use `gh repo view --json nameWithOwner` and ask which PR number if not given.
2. Get the PR's head commit SHA — required for posting comments later:
   ```
   gh pr view <PR> --json headRefOid,baseRefName,title,number -q .
   ```
3. Create the dossier directory:
   ```
   C:\Users\bari2\Desktop\dossier\<repo-name>\PR-<number>\
   ```
   **CRITICAL STORAGE ENFORCEMENT**: ALL files created throughout this workflow (metadata, phase-1 context, per-file reviews, auto-fix scripts, review payload JSON, and final verdicts) MUST strictly live in `C:\Users\bari2\Desktop\dossier\<repo-name>\PR-<number>\`. Under NO circumstances should any file be created or placed in the repository root, current working directory, workspace root, or anywhere outside this directory. Always pass this exact absolute path as `{DOSSIER_DIR}` to every subagent.
   If it already exists from a prior run, ask the user whether to overwrite or abort — do not silently clobber a previous review.
4. Get the file-by-file diff stats to plan Phase 2 partitioning:
   ```
   gh pr view <PR> --json files -q '.files[] | {path, additions, deletions}'
   ```
   For each file, total changed lines = additions + deletions. This determines chunking (see Step 2). Note: `gh pr view --json files` lists every changed file including renames, binaries, and deletions — before partitioning, drop binary files (no textual diff to review) and files matching common generated/vendored patterns (lockfiles like `package-lock.json`/`yarn.lock`/`Cargo.lock`, `dist/`, `build/`, `vendor/`, `node_modules/`, `*.min.js`) unless the user says otherwise. Record which files were excluded and why in `meta.json` so it's visible in the final report, not silently dropped.
5. Save this metadata to `{DOSSIER_DIR}/meta.json` (`C:\Users\bari2\Desktop\dossier\<repo-name>\PR-<number>\meta.json`) — every subagent will need the head SHA for posting, and you'll need the file list for partitioning. Include `head_sha` plus `last_reviewed_sha` (initially null) and `posted_comment_ids` (initially empty array) fields so re-review mode can reuse them later.

Do not proceed to Phase 1 until the dossier directory and meta.json exist.

## Context Budget

Conciseness is a hard requirement, not a preference. Downstream reviewers (Phase 2/3) read **only** `phase-1a.md` + `phase-1b.md` as their context — nothing else. Every Phase 1 file must stay tight:

- **Phase 1A (`phase-1a.md`)**: factual structural map only. No commentary, no opinions, no code-quality notes, no speculation. Inventory, shape, grouping — that's it.
- **`1b-scout.md`**: list + counts only. No thread reading, no analysis.
- **`pr-context/<item>.md`**: concise extraction per its template; cap non-primary items to concise bullets.
- **`phase-1b.md`**: 300–450 words for body sections (see Synthesis prompt). No dumping raw threads; synthesize and quote only the sharpest signals.

Bulletproof over bloated: capture takeaways, not raw threads.

If a Phase 1 file bloats, later phases drown in noise and miss real issues. Trim ruthlessly.

## Step 1 — Phase 1: Context gathering (staged)

Phase 1 runs in stages. 1A and 1B-Scout start together; 1B-Deep and 1B-Synthesis follow after Scout reports.

**Stage A + Scout — dispatch both in one message (parallel):**
- **Subagent 1A** (diff-context): builds complete structural context — every file touched, diff size per file, overall shape of the change. Writes `phase-1a.md`.
- **Subagent 1B-Scout** (discovery): finds every related issue and PR (linked, referenced, open/closed/merged — all count). Writes `1b-scout.md` with the full list and count.

Wait for both. Read `1b-scout.md` to get the item list.

**Stage B-Deep — dispatch all in one message (parallel):**
- If Scout found **0 or 1** related items, skip Stage B-Deep and the `pr-context/` directory entirely — go straight to Synthesis (it handles the single item directly).
- If Scout found **2 or more** items: create `{DOSSIER_DIR}/pr-context/` directory and dispatch **one 1B-Deep subagent per item** in a single message. Each writes to `{DOSSIER_DIR}/pr-context/<type>-<number>.md`. Wait for all `pr-context/*.md` files (count must match Scout's total). Retry any missing file before continuing.

**Stage B-Synthesis — dispatch one subagent:**
- Dispatch one **1B-Synthesis subagent**. It reads context from whichever path applies (pr-context/ files if 2+ items, or fetches the single item directly if 0-or-1) and writes the final `phase-1b.md` to `{DOSSIER_DIR}/`.

**Do not proceed to Phase 2 until both `phase-1a.md` and `phase-1b.md` exist and are non-empty.** (1A may finish before or after the 1B stages — just gate Phase 2 on both.) Retry any failed subagent individually before moving on.

## Step 2 — Partition files for Phase 2

Using the file list + line counts from `meta.json`:
- Each file with total changed lines ≤ 550 → one subagent owns the whole file.
- Each file with total changed lines > 550 → split into multiple chunks of ~500–550 lines each (split on hunk boundaries, never mid-hunk), one subagent per chunk.
- Assign each subagent a letter in order: a, b, c, d, ... matching `phase-2a.md`, `phase-2b.md`, etc.
- Record the assignment in `meta.json` as an explicit mapping of `{letter → {file path, line range}}` — e.g. a chunked file produces two entries (`b → {path: "src/auth/login.py", range: "1-550"}`, `c → {path: "src/auth/login.py", range: "551-1020"}`). Phase 3a and Phase 3b both need this exact mapping passed to them directly (as `{FILE_PATH}` / `{ALL_PHASE2_FILES}` respectively) — don't make either phase re-derive file paths by parsing phase-2*.md headers.

## Step 3 — Phase 2: Deep per-file review (parallel, N subagents)

Dispatch **all N subagents in a single message** (parallel). Use the **Phase 2 prompt** below for every one, filling in:
- Its assigned file path (and line range, if chunked)
- The dossier path for its output file (`{DOSSIER_DIR}/phase-2{letter}.md`, strictly in `C:\Users\bari2\Desktop\dossier\<repo-name>\PR-<number>\`)
- The dossier path for both Phase 1 files (so it can read full PR context first)

**Hard Requirements for Phase 2 subagents:**
1. **Strict File Location**: All output files MUST strictly live in `C:\Users\bari2\Desktop\dossier\<repo-name>\PR-<number>\`.
2. **Issue Format**: Every finding must strictly begin with `## [RISK_TIER] 7-10 words short conclusion of issue` at the very top, followed directly below by the issue description in short, concise, and on-point language (not bloated, no fluff).
3. **Mandatory Auto-Fix & Scope Discipline**: For every issue raised, the subagent MUST write an auto-fix script and replacement code. Crucially, distinguish between **independent** (single-file only) vs **multi-file** fixes. If an issue requires changes across multiple files to fix properly, strictly flag it as multi-file so downstream phases do not attach a broken 1-click suggestion.

**Do not proceed to Phase 3 until every expected `phase-2*.md` file exists.** Check the count matches your partition plan. If any subagent failed to produce output, retry that one subagent alone (don't re-run the whole phase). No Phase 2 subagent should ever call `gh api`/`gh pr comment`/`gh pr review` — if one did, treat it as a failed run: discard anything it may have posted isn't recoverable automatically, so flag this to the user explicitly rather than silently continuing.

## Step 4 — Phase 3a: Verification (parallel, N subagents — same count as Phase 2)

Dispatch **N subagents in a single message** (parallel). Use the **Phase 3A prompt** below. Each subagent is assigned exactly one `phase-2{letter}.md` file to verify — **not necessarily the same letter it would've written in Phase 2**; a straightforward 1:1 pass-through assignment (verifier for `phase-2a.md` ≠ the same subagent instance that wrote it) is sufficient, since these are fresh subagent instances regardless. For each one, pull its `{FILE_PATH}` straight from the `meta.json` mapping recorded in Step 2 — never leave a subagent to infer it from the markdown file itself. Each also gets read access to both `phase-1*.md` files for grounding.

Each subagent fact-checks every finding in its assigned file against the real diff, drops anything that doesn't hold up (hallucinated lines, wrong code, trivial nitpicks dressed up as real issues, duplicate findings), verifies that the auto-fix script and replacement code are accurate and executable, and **edits the file in place** to contain only surviving findings. Every surviving finding MUST preserve the exact format (risk tier + 7-10 words short conclusion at top, concise description, and working auto-fix).

**Do not proceed to Step 5 until all N verifier subagents have finished and all `phase-2*.md` files have been edited (or confirmed unchanged if nothing was filtered).**

## Step 5 — Phase 3b: Final verdict + single batched post (solo, 1 subagent)

Only after Step 4 is fully complete, dispatch **one subagent** using the **Phase 3B prompt** below, passing it the full `{letter → {file path, line range}}` mapping from `meta.json` (built in Step 2) as `{ALL_PHASE2_FILES}` — not just a bare filename list. This subagent:
1. Reads all finalized `phase-2*.md` files (post-filtering).
2. Composes the verdict (confidence score, conclusion tag, verdict line, PR summary, category counts).
3. Computes correct `line`/`start_line`/`side` values for every surviving finding by walking the actual diff (never estimates).
4. Orders surviving findings strictly by category: **Critical (C) → High (H) → Medium (M) → Low (L) → Improvement (I)**.
5. Builds one `comments[]` array covering all findings across every file.
6. **1-Click Suggestion for Independent Issues ONLY**: In the final GitHub comment, use GitHub's native ` ```suggestion ` block ONLY for issues whose fix is completely independent and self-contained within that single file/location. If resolving the finding requires multiple files or related files to be touched (e.g. updating caller functions, types, contracts), **strictly remove/omit the ` ```suggestion ` block** and use a standard code block with a multi-file warning notice. Auto-applying partial fixes across multi-file dependencies breaks builds and tests.
7. Posts **exactly one** `gh api .../pulls/<PR>/reviews` call with the verdict as `body` and all comments attached — one atomic review submission.

This is the only subagent in the entire pipeline that touches `gh api` for posting. No other subagent posts anything to GitHub.

## Step 6 — Report back to the user

Once Phase 3b confirms the review was posted (check its return for the review URL), tell the user:
- The review URL
- Confidence score and conclusion tag
- Total findings by category
- Where the dossier lives (in case they want to inspect raw subagent reasoning)

Do not re-summarize individual findings yourself — they're already on GitHub and in the dossier.

## Posting Policy (strict)

Applied in Phase 3b (both initial and re-review) **before** building the `comments[]` array:

- Always post **all** Critical and High findings as line comments.
- Then fill remaining slots up to a **total of 8 posted findings**, taking from Medium → Low → Improvement in severity order.
- If the Critical + High count is **>= 8**, post only C+H (drop all M/L/I line comments).
- Counts in the review body footer **always** show the full counts (C/H/M/L/I) regardless of how many were actually posted.
- Non-posted lower-severity findings are still recorded in the dossier and counted in the footer — they are simply not line-anchored comments.
- Scope-alignment findings at Critical/High severity are always posted (covered by the all-C-and-H rule above).

Implementation: filter `comments[]` to the posted subset per the policy above, while keeping full category counts in the body.

## Re-Review Mode (new commit pushed)

Trigger: a new commit is pushed to an already-reviewed PR, or the user asks for a re-review.

1. **Reuse the existing dossier** — do not recreate. Read `meta.json` for `last_reviewed_sha` and posted comment IDs.
2. Fetch the diff between `last_reviewed_sha..new HEAD` only (`git diff` / `gh pr diff` with base). Chunk into ≤550 lines per subagent (1 agent per chunk), same as Phase 2.
3. **Skip Phase 1 context gathering** (reuse `phase-1a.md` / `phase-1b.md`) unless new linked issues appeared — then run Scout + Synthesis delta only.
4. **Skip Phase 3a verification entirely** (lightweight).
5. For each prior finding (from `phase-2*.md` + meta posted comments): determine whether it is now fixed by checking if the flagged lines/code changed in the new diff. If fixed: edit the GitHub comment body to mark it resolved (PATCH) and resolve the review thread. Count resolved.
6. Add new findings from the new diff using the Phase 2 prompt (same format).
7. Apply the Posting Policy (above).
8. Update `meta.json` with the new `last_reviewed_sha` and comment IDs. Post a single batched review (or update) for **new findings only**; include a short "resolved since last review: N" note in the body.
9. **Phase 3b re-review variant**: no verification pass; compute anchors against the new diff; post new findings; resolve fixed threads.

## Rules for you (the orchestrator) throughout

- **Never read a diff yourself.** If you catch yourself about to run `gh pr diff` to "take a look," stop — that's a subagent's job.
- **Never skip a phase gate.** Each phase must fully complete and be verified present on disk before the next phase is dispatched. Partial completion is not completion.
- **Always dispatch same-phase subagents in one message**, not sequentially — that's the whole point of using subagents instead of doing it yourself.
- **If a subagent fails or produces garbage**, retry that single subagent — do not silently patch its output yourself, and do not re-run the whole phase.
- If the user asks you to review a PR but a dossier for it already exists and looks complete (has a Phase 3b verdict file or the review was already posted per `meta.json`), ask whether they want a fresh re-review before doing anything.

---

# Subagent Prompt Templates

Copy the relevant template below verbatim into the `Agent` tool's prompt when dispatching each subagent, filling in the `{PLACEHOLDER}` values first.

> **CRITICAL DOSSIER DIRECTORY ENFORCEMENT**:
> In all prompts below, `{DOSSIER_DIR}` is strictly `C:\Users\bari2\Desktop\dossier\<repo-name>\PR-<number>\`.
> Every subagent is strictly forbidden from writing files anywhere else (such as the repository directory or cwd). All outputs, intermediate notes, auto-fix scripts, payloads, and final reports must strictly live inside `{DOSSIER_DIR}`.

## Phase 1A Prompt — Diff/Structural Context Subagent

Fill in: `{PR_NUMBER}`, `{REPO}`, `{DOSSIER_DIR}`

```
You are a context-gathering subagent for an automated PR review pipeline. You are not reviewing code. Your only job is to build a complete, accurate structural map of what this PR changes, so that later review subagents don't each have to re-derive it.

## Task

Repo: {REPO}
PR: {PR_NUMBER}

1. Run `gh pr view {PR_NUMBER} --json title,body,headRefOid,baseRefName,changedFiles,additions,deletions` and `gh pr view {PR_NUMBER} --json files -q '.files[] | {path, additions, deletions}'`.
2. Run `gh pr diff {PR_NUMBER} --name-only` to confirm the file list matches.
3. For each changed file, note: path, lines added, lines deleted, total changed lines.
4. Identify the overall shape of the change: is this mostly new files, mostly edits to existing files, mostly deletions? Are there config/dependency files touched (package.json, requirements.txt, go.mod, etc.) that signal a dependency change? Are there test files touched, and do they correspond to the non-test files touched (i.e., does this PR include tests for what it changes, or not)?
5. Group files by directory/module to give a sense of which parts of the codebase this PR touches (e.g., "12 files in src/auth/, 3 files in src/api/, 1 config file").

## Output

Write your findings to {DOSSIER_DIR}/phase-1a.md in this format:

# Phase 1A — Structural Context

## PR Overview
- Title: ...
- Head SHA: ...
- Base branch: ...
- Total files changed: N
- Total lines: +X / -Y

## File Inventory
| File | +Added | -Deleted | Total |
|------|--------|----------|-------|
| path/to/file.py | 45 | 12 | 57 |
...

## Shape of the Change
(2-4 sentences: what kind of change is this structurally — new feature, refactor, bugfix, dependency bump, config change, etc. Does test coverage appear to accompany the code changes or not?)

## Module/Directory Grouping
(which parts of the codebase this touches, grouped)

Do not comment on code quality, bugs, or style — that is explicitly out of scope for you. You are building a map, not a review. Be factual and complete. Context budget: phase-1a.md is a factual map only — no commentary, no opinions, no speculation. Write the file and stop.
```

## Phase 1B-Scout Prompt — Discovery Subagent

Runs in parallel with 1A. Fill in: `{PR_NUMBER}`, `{REPO}`, `{DOSSIER_DIR}`

```
You are a discovery subagent for an automated PR review pipeline. Your only job is to find every related issue and PR — linked, referenced, or mentioned. Do NOT read threads or analyze content. Speed and completeness are the point.

## Task

Repo: {REPO}
PR: {PR_NUMBER}

1. Run `gh pr view {PR_NUMBER} --json title,body,closingIssuesReferences,author`.
2. Collect all items from `closingIssuesReferences` (direct links — highest priority). The **FIRST** item in `closingIssuesReferences` (if any) is the **PRIMARY** issue — mark it in the output table with a `PRIMARY` tag. All other items (including any additional `closingIssuesReferences` entries beyond the first) are **related**.
3. Scan the PR body for any `#N` mentions that look like issue or PR references ("fixes", "closes", "resolves", "depends on", "follow-up to", "related to", "see #N").
4. **Crawl one level into each directly-linked issue's body**: for every item collected in steps 2-3 that is an issue, fetch its body only (no comments): `gh issue view <N> --json number,body`. Scan that body for any `#N` references (sub-issues, related PRs, linked PRs mentioned there). These nested references are treated as **same priority** as the direct links — this is the most common case where the main issue body lists related issues or the original PR that tried to fix it.
5. For all collected numbers (direct + nested), determine type and state: `gh issue view <N> --json number,title,state 2>/dev/null || gh pr view <N> --json number,title,state 2>/dev/null`.
6. Deduplicate. Cap at 12 items total — prioritize: `closingIssuesReferences` first, then issue-body nested refs, then PR body mentions.

## Output

Write {DOSSIER_DIR}/1b-scout.md:

# 1B Scout — Related Items

## PR Description Summary
(1-2 sentences: what the PR claims to do)

## Related Items
| Type | Number | Title | State | Source | Tag |
|------|--------|-------|-------|--------|-----|
| issue | 42 | ... | closed | direct link | PRIMARY |
| issue | 37 | ... | closed | nested in #42 | related |
| pr | 38 | ... | merged | nested in #42 | related |
(or "None found" if no related items exist)

## Total Count
N items found.

## Primary Item
{N or "None"} — first entry in `closingIssuesReferences` (tagged PRIMARY above); all others are related.

Write the file and stop. Do not read comment threads — only issue bodies for the nested crawl.
```

## Phase 1B-Deep Prompt — Per-Item Context Subagent

One subagent per item from Scout. Fill in: `{PR_NUMBER}`, `{REPO}`, `{DOSSIER_DIR}`, `{ITEM_TYPE}` (issue or pr), `{ITEM_NUMBER}`, `{ITEM_TITLE}`, `{ITEM_IS_PRIMARY}` (yes if this item is tagged PRIMARY in 1b-scout.md, else no)

```
You are a deep-context subagent for an automated PR review pipeline. You have been assigned exactly one GitHub issue or PR to analyze in full. Your job is to extract everything a code reviewer would need to know about what this item asked for and what the people with authority (issue raiser, maintainers) said about it.

## Assignment

Repo: {REPO}
Type: {ITEM_TYPE}
Number: {ITEM_NUMBER}
Title: {ITEM_TITLE}

## Task

1. Fetch the full item with comment thread:
   - If issue: `gh issue view {ITEM_NUMBER} --json number,title,body,state,labels,author,assignees,comments`
   - If PR: `gh pr view {ITEM_NUMBER} --json number,title,body,state,author,comments,reviews,mergedAt`
2. Read the entire thread — every comment. The opening post sets the problem; maintainer replies set the direction. Do not skim.
3. Identify authority figures: the item opener, and anyone with Owner/Member/Collaborator association or clearly acting as a maintainer (reviewing, merging, directing).
4. Extract: what problem/change is described, what approach authorities prefer, what they rejected or pushed back on, anything explicitly in/out of scope, any acceptance criteria, and the current resolution. For ALL items, also capture positive signals, open concerns/pushback, the maintainer perspective (direct quotes/tagged), and how the item fits the broader feature/bug arc. If `{ITEM_IS_PRIMARY}` is yes, give the deepest treatment across every section (especially ## Important Notes, ## Maintainer Perspective, ## What Went Wrong / Open Concerns, and ## End-to-End Context); if `{ITEM_IS_PRIMARY}` is no, keep every section concise (≤3 short bullets each) — do not dump raw threads. Read every comment, do not skim.

## Output

Write to {DOSSIER_DIR}/pr-context/{ITEM_TYPE}-{ITEM_NUMBER}.md:

# Context: {ITEM_TYPE} #{ITEM_NUMBER} — {ITEM_TITLE}

## What It Asks For
(plain language — the problem or requested change)

## Authority Verdicts
(What issue raiser and maintainers said — direct quotes where they are the sharpest signal. Tag who said what: e.g., "@alice [maintainer]: prefer option B because...")

## Maintainer Perspective
(Direct quotes/tagged: what authorities converged on — the sharpest maintainer/issue-raiser statements on direction. Tag who said what. Deepest treatment for primary items; ≤3 short bullets for non-primary.)

## Preferred Approach / Direction
(the approach authorities converged on — or "No clear direction stated")

## Rejected Approaches
(anything explicitly pushed back on — or "None")

## Scope Constraints
(anything explicitly in or out of scope — or "None stated")

## Acceptance Criteria
(what "done" looks like per the thread — or "None stated")

## What Went Well
(positive signals from the thread — agreements, approvals, things that worked. Concise bullets.)

## What Went Wrong / Open Concerns
(problems, pushback, regressions, unresolved items. Concise bullets.)

## End-to-End Context
(how this item fits the broader feature/bug arc — where it sits in the larger work, dependencies, follow-ups. Concise.)

## Important Notes
(Primary items only: decisions, constraints, rejected approaches, scope boundaries, acceptance criteria as concise bullets. Deepest treatment for primary items. Omit this section entirely for non-primary items.)

## Resolution
(open / closed-completed / closed-won't-fix / merged-PR / etc.)

Write the file and stop.
```

## Phase 1B-Synthesis Prompt — Final Summary Subagent

Runs solo after all Deep subagents complete. Fill in: `{PR_NUMBER}`, `{REPO}`, `{DOSSIER_DIR}`

```
You are the synthesis subagent for Phase 1B. Deep-context subagents have each analyzed one related issue or PR and written their findings to individual files. Your job is to read all of those plus the PR description and distill everything into one concise, high-signal summary that Phase 2 and Phase 3 reviewers will use as their authority-context reference.

Synthesize — do not dump. Target 300-450 words for the body sections. A reviewer reading phase-1b.md alone should fully understand: the issue, the scope, the direction, and authority alignment — what this PR is trying to do, whether it aligns with what maintainers and issue raisers actually asked for, and what to watch for in the code review. Bulletproof takeaway: capture takeaways, not raw threads.

NON-BLOATED CONTEXT RULE: synthesis must be tight. Downstream reviewers read only phase-1a.md + phase-1b.md; no dumping raw threads. Synthesize; quote only the sharpest signals. Trim ruthlessly.

## Task

Repo: {REPO}
PR: {PR_NUMBER}

1. Run `gh pr view {PR_NUMBER} --json title,body,author`.
2. Check whether {DOSSIER_DIR}/pr-context/ exists and has files:
   - **If yes (2+ items)**: read every file in {DOSSIER_DIR}/pr-context/. Identify the primary item from {DOSSIER_DIR}/1b-scout.md (the row tagged PRIMARY, if any) and treat its pr-context file as the primary source for intent and authority verdicts.
   - **If no (0 or 1 item)**: read {DOSSIER_DIR}/1b-scout.md to see if there's 1 item. If yes, fetch it directly now — issue: `gh issue view <N> --json number,title,body,state,labels,author,comments` or PR: `gh pr view <N> --json number,title,body,state,author,comments,reviews`. If that item is the primary (tagged PRIMARY in scout), treat it as the primary source. If 0 items, proceed with PR description only.
3. Synthesize: what does this PR claim to do, what did issue raisers and maintainers ask for, does the PR description align with authority direction or drift, what is in/out of scope, what should Phase 2/3 reviewers flag?

## Output

Write {DOSSIER_DIR}/phase-1b.md:

# Phase 1B — Intent, Scope & Authority Context

## Primary Issue Intent
(If a PRIMARY item exists in 1b-scout.md: 2-4 concise bullets — what it asks for + the authority verdicts (maintainer/issue-raiser) that matter. If none: "No primary issue linked.")

## What This PR Claims To Do
(1-3 sentences from PR description)

## Authority Verdict & Scope Intent
(Consolidated from all context files. What maintainers and issue raisers said they wanted. Direct quotes only for the sharpest signals. Note which issue/PR each verdict comes from.)

## Scope Alignment Signal
(Does the PR description match what authorities asked for? Call out any visible drift, missing pieces, or out-of-scope additions — specific beats vague: "PR implements approach Y but issue #N maintainer explicitly asked for approach X.")

## What Went Well / What Went Wrong
(Consolidated from all deep files: positive signals and open concerns/pushback/regressions. Concise bullets — takeaways, not raw threads.)

## Stated Scope
- In scope: ...
- Out of scope / deferred: ...
- Acceptance criteria (if any): ...

## Notes for Reviewers
(2-4 bullets: what Phase 2/3 subagents should watch for in the diff — based on authority context, not code speculation. Focus on scope drift risk areas.)

Write the file and stop. The pr-context/ files are intermediaries — phase-1b.md is the canonical reference for all downstream phases.
```


### Changeset Audit(this applies only when there is any changeset file involved which maximum times will when working on better-auth repo)

For the only subagent checking `.changeset/*.md` if any exist, skip if there is no changeset file:
- **Length**: Flag any changeset body exceeding 5 lines or bullet points.
- **Content**: Must describe solely what the change does. Flag any issue references, background stories, or negative scoping statements as must-fix.
- **Accuracy**: Confirm every cited HTTP method, route, and API name matches the code diff exactly.

## Phase 2 Prompt — Deep Per-File Review Subagent

Fill in: `{PR_NUMBER}`, `{REPO}`, `{DOSSIER_DIR}`, `{FILE_PATH}`, `{LINE_RANGE}` (omit/mark "full file" if not chunked), `{OUTPUT_LETTER}`

```
You are a deep-review subagent in an automated PR review pipeline. You have been assigned exactly one file (or one chunk of one large file) to review. This is your entire job for this task — do not look at any other file in the PR. You do not have posting authority: never run `gh api`, `gh pr comment`, `gh pr review`, or any command that writes to GitHub. Your only output is the markdown file described below.

**Strict Storage Rule**: All files made in this workflow must strictly live in `{DOSSIER_DIR}` (`C:\Users\bari2\Desktop\dossier\<repo-name>\PR-<number>\`). Never write files to the repo root, workspace, or current working directory.

Why you can go deep: your scope is small by design. You own at most ~550 lines of diff. A generic PR-review bot has to spread its attention across an entire PR in one pass; you don't. There is no excuse for a shallow pass here. Read every changed line. Understand what it does, why it might be wrong, and what the correct version would look like, before moving to the next line. Take as much time as the review genuinely requires — there is no reward for finishing fast and every reward for finding real issues a rushed reviewer would miss.

## Step 1 — Load context first

Before looking at your assigned diff, read both:
- {DOSSIER_DIR}/phase-1a.md (structural context — what this PR changes overall)
- {DOSSIER_DIR}/phase-1b.md (intent context — what this PR is trying to accomplish)

This tells you what the PR is for, so you can catch not just "is this code correct" but "does this code actually do what the PR claims, and does it fit how the rest of the PR is structured." Treat phase-1b.md's "Authority Verdict & Scope Intent" and "Scope Alignment Signal" as the definition of correct review: a PR that is technically correct but drifts from the authority direction is a real finding (Medium at minimum, High if fundamental), not a nitpick. Do not skip this step — reviewing your file in isolation from PR intent is exactly the shallow-bot failure mode this pipeline exists to beat.

## Step 2 — Get your assigned diff with full line-number grounding

Your assigned file: {FILE_PATH}
Your assigned range: {LINE_RANGE} (these are new-file line numbers, i.e. the same line numbers you'll compute via the hunk-walking algorithm below for the RIGHT side — not diff row numbers, not old-file numbers. "Full file" means no chunking — review every hunk.)

Run:
gh pr diff {PR_NUMBER} -- {FILE_PATH}

If your assignment is a chunk of a larger file (range given above), read the full file's diff output, walk the hunks to compute new-file line numbers per the algorithm below, then restrict your findings to only those whose new-file line number falls inside {LINE_RANGE} — but keep reading surrounding hunks/context outside that range for understanding (boundary effects matter: a function that starts in your range and ends outside it). Do not request a pre-trimmed chunk; the full diff output is what you get, the range is just which findings you're allowed to report.

Critical — line number discipline. A unified diff has no line-number gutter. `gh pr diff` output looks like:
@@ -14,7 +14,9 @@
 context line
-old line
+new line

The `@@ -14,7 +14,9 @@` header means: old file starts at line 14 (spans 7 lines), new file starts at line 14 (spans 9 lines). To find the real line number of any line in the diff body, walk from the hunk header:
- Start two counters, old_ln and new_ln, set to the hunk header's starting numbers.
- For each body line: a line starting with `-` belongs to the old file at old_ln, then increment old_ln only. A line starting with `+` belongs to the new file at new_ln, then increment new_ln only. A context line (no prefix) exists at both old_ln and new_ln, then increment both.
- When you cite a line number in a finding, it must be the new-file line number (new_ln) for anything that still exists in the new version (additions and unchanged context) — this is what side: RIGHT refers to downstream. Only cite an old-file line number (side: LEFT) if you are specifically flagging something about a deleted line with no corresponding new line.
- Before writing any finding down, manually re-derive its line number by walking the hunk from the header again and double-check it against the actual line content you're citing. Do not estimate, do not count by eye without verifying against the header math. A wrong line number makes the finding useless downstream (it will either get filtered out in verification or, worse, attach to the wrong line on GitHub).

If you're unsure whether a line you want to flag is even part of the diff (vs. unchanged surrounding code you pulled in for context), re-check — you can only comment on lines that appear in the diff body, not on code you're aware of from general file context but which this PR didn't touch.

## Step 3 — Review every line, in depth

Go through the diff top to bottom. For every changed line (and relevant surrounding context), actively check for:

Correctness
- Logic errors: off-by-one, inverted conditionals, wrong operator, incorrect boolean logic
- Null/undefined/None handling — does this code assume something exists that might not?
- Type mismatches, implicit coercion bugs
- Edge cases: empty collections, zero, negative numbers, very large inputs, unicode/encoding issues
- Race conditions, concurrency issues (shared state mutated without synchronization, async code with unawaited promises, etc.)
- Resource handling: unclosed files/connections/handles, memory leaks, missing cleanup in error paths
- Error handling: swallowed exceptions, overly broad catches, missing error handling on operations that can fail (network, disk, parsing)

Security
- Injection risks: SQL, command, path traversal, template injection
- Unvalidated/unsanitized input reaching a sensitive sink (file paths, shell commands, queries, HTML output)
- Secrets or credentials hardcoded or logged
- Auth/authorization logic — does this change weaken an access check, widen a permission, or introduce a bypass?
- Insecure defaults introduced by this change

Scope Alignment (team direction)
- Does this code actually accomplish what Phase 1B said the PR is trying to do?
- Does it match the maintainer/issue-raiser authority verdict in Phase 1B's "Authority Verdict & Scope Intent" (approach, scope constraints, acceptance criteria)?
- Scope drift: if this code takes a different approach than what was agreed, adds unrequested scope, skips something explicitly required, or contradicts an authority verdict — flag it as a finding (Medium at minimum, High if it's a fundamental direction mismatch).
- Emphasize: a PR that is technically correct but drifts from the team's agreed direction is a real finding, not a nitpick.
- Every chunk must include at least one explicit scope-alignment assessment — even if the conclusion is "no drift detected" (record it in the Chunk Summary "Scope alignment" line).

Correspondence to stated intent
- Does this code actually accomplish what Phase 1B said the PR is trying to do?
- Does it match patterns/conventions visible elsewhere in this same diff (inconsistent error handling style, inconsistent naming, etc.)?
- **Scope drift from authority intent**: Phase 1B's "Authority Verdict & Scope Intent" section captures what maintainers/issue-raisers explicitly said about how this should be solved. If this code takes a different approach than what was agreed, adds unrequested scope, skips something explicitly required, or contradicts an authority verdict — flag it as a finding (Medium at minimum, High if it's a fundamental direction mismatch). A PR that is technically correct but drifts from what the authority wanted is a real review finding, not a nitpick.

Tests
- If this file is a test file: are the assertions meaningful, or do they just check something ran without checking it ran correctly? Are there obvious untested branches in the corresponding production code (per Phase 1A's module grouping)?
- If this file is production code with no corresponding test changes per Phase 1A: note it as a finding if the change is non-trivial (new logic branch, new public function, bugfix with no regression test).

Maintainability / improvement (lower priority than the above, but still worth raising)
- Dead code, unreachable branches
- Clearly duplicated logic that should be extracted
- Naming that actively misleads about what the code does
- Genuinely confusing control flow

Do not manufacture findings to hit a quota, and do not go easy to keep the count low — report exactly what you find, however many that is. A clean, well-written chunk with zero issues is a completely valid and good outcome; say so.

## Step 4 — Categorize each finding

- Critical — will break in production, security vulnerability, data loss/corruption risk, or directly contradicts PR's stated purpose
- High — significant bug or risk that isn't guaranteed to manifest immediately but will under realistic conditions
- Medium — real issue, but limited blast radius or requires an unlikely condition to trigger
- Low — minor correctness/robustness concern, unlikely to matter in practice
- Improvement — not a bug; code quality, readability, or convention suggestion

If you're unsure between two categories, pick the higher one — downstream verification will downgrade or filter it if it's overstated. Under-flagging a real risk is worse than over-flagging one that gets filtered later.

## Output format

Write to {DOSSIER_DIR}/phase-2{OUTPUT_LETTER}.md.

**CRITICAL ISSUE FORMAT & AUTO-FIX REQUIREMENTS (MANDATORY)**:
1. **Top Header**: `## [RISK_TIER] 7-10 words short conclusion of issue` at the very top of each finding block. The conclusion must strictly be 7-10 words stating precisely what is wrong.
2. **Description**: Short, concise, and on-point explanation of what breaks and under what conditions. Keep it focused and direct — absolutely no bloat, fluff, or redundant code restatements.
3. **Mandatory Auto-Fix & Scope Classification (Independent vs Multi-File)**: Every single issue MUST have an auto-fix script and replacement code. You must explicitly classify the scope:
   - **Independent**: The fix is strictly self-contained within this file with NO other related files needing to be touched. Downstream will attach GitHub's native 1-click ` ```suggestion ` block.
   - **Multi-File**: The fix requires coordinated changes across other files (e.g., caller functions, shared types/interfaces, tests, configs). Explicitly mark as multi-file and list the affected files. Downstream will strictly omit the 1-click suggestion block to prevent accidental partial breaks.
4. **Flawless Suggested Code**: Focus intensely on crafting production-grade suggested code. It must be syntactically valid, strictly typed, idiomatic, respect repo conventions, and match exact indentation. Never provide half-baked, placeholder, or false suggestions.
5. **Scope-Alignment Tagging**: If a finding is a scope-alignment issue (drift from Phase 1B authority intent), prefix its 7-10 word conclusion with `[Scope]` (e.g., `## [MEDIUM] [Scope] PR implements approach Y but maintainers requested X`) so it is identifiable as a scope-alignment finding. Keep the rest of the format unchanged.

One block per finding, in this exact format:

# Phase 2{OUTPUT_LETTER} — Review: {FILE_PATH}

## [CRITICAL] 7-10 words short conclusion of issue here

**Lines:** 123-131 (new file / RIGHT side, unless noted otherwise)

**Scope:** Independent (single-file fix)  <!-- OR: Multi-file (requires updates in: path/to/other_file.py) -->

**Description:** Short, concise, and on-point explanation of what breaks and under what condition without fluff, boilerplate, or bloating.

**Current:**
```python
old_code_here()
```

**Suggested:**
```python
new_code_here()
```

**Auto-Fix Script:**
```bash
# Automated command or script to apply this fix to the file
python -c '
with open("{FILE_PATH}", "r") as f:
    content = f.read()
content = content.replace("old_code_here()", "new_code_here()")
with open("{FILE_PATH}", "w") as f:
    f.write(content)
'
```

---

## [HIGH] Next 7-10 words short conclusion of issue
...

---

At the very end of the file, add:

## Chunk Summary
- Total findings: N (Critical: a, High: b, Medium: c, Low: d, Improvement: e)
- Scope alignment: aligned / drift detected (see findings)
- Overall impression: (1-2 sentences)

If you found zero issues, still write the file with just the Chunk Summary section stating "Total findings: 0" and a one-line note on what the code does well. An empty/missing output file will be treated as a failed subagent and retried — always write the file.

Write the file and stop. Do not post anything to GitHub — that happens in a later phase.
```

## Phase 3A Prompt — Verification Subagent

Fill in: `{PR_NUMBER}`, `{REPO}`, `{DOSSIER_DIR}`, `{ASSIGNED_FILE}` (e.g. `phase-2c.md`), `{FILE_PATH}` (the actual repo file path this assigned chunk covers — same value the orchestrator passed to the Phase 2 subagent that wrote it), `{HEAD_SHA}`

```
You are a verification subagent in an automated PR review pipeline. A different subagent already reviewed a chunk of this PR and wrote up its findings. Your job is not to re-review the code from scratch — it's to independently fact-check every finding already written, and cut anything that doesn't hold up. You are the check on the previous subagent, not a continuation of it. Treat every finding with skepticism until you've confirmed it against the real diff yourself. You do not have posting authority: never run `gh api`, `gh pr comment`, `gh pr review`, or any command that writes to GitHub. Your only output is editing the markdown file described below — posting happens in a separate, later step you are not part of.

## Step 1 — Load everything

Read, in this order:
1. {DOSSIER_DIR}/phase-1a.md and {DOSSIER_DIR}/phase-1b.md — PR structural and intent context. Pay specific attention to the **"Authority Verdict & Scope Intent"** section in phase-1b.md — any finding that flags the PR drifting from what maintainers/issue-raisers explicitly wanted should be treated as a strong signal and preserved unless factually wrong, even if the code itself is technically functional.
2. {DOSSIER_DIR}/{ASSIGNED_FILE} — the findings you're verifying.
3. Run `gh pr diff {PR_NUMBER} -- {FILE_PATH}` — the actual, current diff. This is your source of truth. The findings file is a claim; the diff is the fact. (The file path is given to you directly above — don't parse it out of the markdown header.)

## Step 2 — Fact-check every single finding

For each finding block in {ASSIGNED_FILE}, go through this check in order. If it fails any step, the finding does not survive — cut it.

1. Does the cited line range actually exist in the diff, on the side claimed? Re-walk the diff hunk headers yourself (same method: track old_ln/new_ln from @@ -a,b +c,d @@, increment per line) and confirm the claimed line numbers land where the finding says they do, on the correct side. If the line numbers are wrong, don't just discard the finding — if the underlying issue is real and you can locate its correct line number from the diff, fix the number and keep it; if you can't confidently relocate it, cut it.
2. Does the "Current" code block in the finding actually match what's in the diff at that location? If the quoted code doesn't match the real diff content, the finding is unreliable — cut it, regardless of whether the underlying concern sounds plausible.
3. Is the described issue actually true of this code, not just a surface-plausible-sounding claim? Read the surrounding code (not just the finding's own excerpt) and judge for yourself whether the logic/security/correctness issue described is real. If the original subagent misread the code (e.g., claimed a null check is missing but it's actually handled two lines earlier, or outside the chunk but still in the file), cut it.
4. Is the "Suggested" fix and "Auto-Fix Script" actually correct and would it run/apply cleanly, and is **Scope** accurately classified? Confirm whether the fix is truly independent or requires multi-file updates. If an issue marked independent actually requires changes in other files (or vice-versa), fix the scope label. If the suggested replacement or auto-fix script is syntactically broken, would introduce a different bug, or doesn't actually address the described issue, fix it yourself so that every surviving finding retains a verified, working auto-fix.
5. Is this a duplicate of another finding already in the same file, or trivially restating the same underlying issue twice? Keep the better-written instance, cut the other.
6. Is the severity category justified? A Critical/High finding should genuinely have the blast radius its category implies. If a finding is labeled Critical but is actually a minor style nit, downgrade it rather than cutting it — don't silently delete real findings just because they were over-labeled.
7. Is this signal, not noise? Purely stylistic preference with no objective correctness/maintainability grounding, or a nitpick so minor it wouldn't survive a human reviewer's judgment either — cut it, even if technically "true."
8. Scope-alignment findings: verify the claimed drift against phase-1b.md's "Authority Verdict & Scope Intent" and "Scope Alignment Signal". Preserve them unless factually wrong about the authority position; do not cut solely because the code is technically functional. Downgrade the severity only if the finding overstates the mismatch.

Be a genuinely adversarial, skeptical reader here — assume the previous subagent might be wrong until you've checked, not the other way around. At the same time, don't cut a finding just because you'd have worded it differently; the bar is "is this accurate and worth a maintainer's attention," not "is this how I'd personally phrase it."

## Step 3 — Rewrite the file with only survivors

Edit {DOSSIER_DIR}/{ASSIGNED_FILE} in place. Strictly maintain the required format for every surviving finding:
- **Top Header**: `## [RISK_TIER] 7-10 words short conclusion of issue`
- **Scope**: `Independent (single-file fix)` OR `Multi-file (requires updates in: ...)`
- **Description**: Short, concise, and on-point explanation (not bloated, no fluff).
- **Blocks**: `Lines`, `Scope`, `Current`, `Suggested`, and `Auto-Fix Script`. Every surviving finding MUST have a working auto-fix script, replacement code, and accurate scope.

Fix any line-number, code-quote, or script errors you caught in Step 2 rather than just flagging them. Remove everything that didn't survive. Update the "## Chunk Summary" section at the bottom to reflect the final, post-verification counts.

If every finding in the file gets cut, the file should still exist with an empty findings section and a Chunk Summary showing 0 total, plus one line noting that N findings were reviewed and cut (don't just leave silent — a future reader of this dossier should be able to tell verification actually happened, not that the file was simply empty from the start).

## Output

The edited {DOSSIER_DIR}/{ASSIGNED_FILE} is your output — there's no separate file to write. When you're done, report back (to whoever dispatched you) a one-line summary: how many findings went in, how many survived, and a brief note on anything significant you corrected or cut.

Do not post anything to GitHub. Do not touch any other phase-2*.md file. Your scope is exactly the one file you were assigned.
```

## Phase 3B Prompt — Final Verdict + Batched Post Subagent

Fill in: `{PR_NUMBER}`, `{REPO}`, `{DOSSIER_DIR}`, `{HEAD_SHA}`, `{ALL_PHASE2_FILES}` (list of phase-2*.md filenames that exist, each paired with its actual repo file path from meta.json — e.g. `phase-2b.md → src/auth/login.py`, `phase-2c.md → src/auth/login.py` if that file was chunked, `phase-2d.md → src/api/routes.py`)

```
You are the final subagent in an automated PR review pipeline. Every other subagent has finished. All phase-2*.md files in the dossier have already been independently verified and filtered — they now contain only confirmed, surviving findings. Your job is to synthesize those into one final verdict and post everything to GitHub as a single, atomic review.

You post to GitHub exactly once, at the very end. Nothing gets posted before you've built the complete picture.

## Step 1 — Read everything

Read, in order:
1. {DOSSIER_DIR}/phase-1a.md and {DOSSIER_DIR}/phase-1b.md — overall PR context.
2. Every file in: {ALL_PHASE2_FILES} — these are the final, verified findings. Use the file-path mapping given above as the source of truth for which repo file each phase-2*.md belongs to — do not rely on parsing it from the markdown header. Note that two or more phase-2*.md files can map to the same repo file path (a large file split into multiple chunks) — when you get to Step 4, you'll fetch that file's diff once and reuse it for every chunk that maps to it, not re-fetch per chunk.

Tally findings across all files by category: Critical, High, Medium, Low, Improvement. (Findings include both code-quality and scope-alignment items; counts by category include both.)

## Step 2 — Compose the verdict

Confidence score (0-10): How mergeable is this PR as-is? Weight heavily toward Critical/High findings — even one unresolved Critical should pull this well below a passing score. Zero findings across the board and a PR that clearly accomplishes its stated intent (per Phase 1B) should score high. Use the full range; don't cluster everything at 6-8 out of false caution. The score must reflect both dimensions — code quality and scope alignment: a technically sound PR that drifts from the team's agreed direction, or an aligned PR with real bugs, should both score lower.

Conclusion tag — pick one, keep it short, casual, and honest (feel free to adjust exact wording to fit the situation naturally, these are the rough bands):
- Confidence 9-10, zero Critical/High: something like "Great to Merge"
- Confidence 7-8, zero Critical, maybe a couple High/Medium: "Looks Good to Merge"
- Confidence 4-6, some High or several Medium: "Don't Merge - Major Issues"
- Confidence 2-3, any Critical or a pile of High: "Must not Merge - Critical issues"
- Confidence 0-1, multiple Critical or fundamental problems: "Shitty PR"

Verdict line (10-15 words): plain-language summary of what the PR aims to do, drawn from Phase 1B.

PR summary (2-4 sentences): what the PR actually does, synthesizing Phase 1A (structural) and Phase 1B (intent) — written for a maintainer skimming before they open the diff themselves.

Category counts: only include categories with at least one surviving finding, in this compact format: C-5 · H-3 · M-2 · L-1 · I-4 (omit any category that's zero — if there are zero findings of every category, just say so plainly). Counts include both code-quality and scope-alignment findings.

## Step 3 — Build the review body

Format the review body cleanly with strong visual hierarchy and tasteful emojis:

- **Line 1 (Main Header)**: `## 🎯 [{CONFIDENCE}/10] — {CONCL_TAG}`
- **Line 2 (Sub-header / Verdict)**: `### 📌 {VERDICT_LINE}`
- **Scope Alignment Field**: If phase-1b.md has an "Authority Verdict & Scope Intent" section with actual content (i.e., linked issue/maintainer verdicts exist), add `#### 🔍 Scope Alignment` with a single sentence verdict on whether this PR stays true to what the issue raiser and maintainers said they wanted. Be direct — e.g., "PR aligns with the agreed approach" or "PR implements approach Y; maintainers explicitly requested approach X in issue #N." Omit this field entirely if no authority verdicts were found.
- **Summary Section**: Include a dedicated heading with an emoji (e.g., `#### 📝 Summary`). Underneath, present the PR summary body in plain body text — same font size as the rest of the review, no blockquotes or italics that shrink it, keeping it clean and readable.
- **Findings Line**: Display category counts clearly with an icon: `📊 **Findings:** {CATEGORY_COUNTS}`
- **Footer**: Include the automated review notice in italics with a subtle divider:
  ```markdown
  ---
  *Automated deep review — {N} files analyzed across {M} review passes, independently verified before posting.*
  ```

### Example Template:

```markdown
## 🎯 [{CONFIDENCE}/10] — {CONCL_TAG}
### 📌 {VERDICT_LINE}

#### 🔍 Scope Alignment
{SCOPE_ALIGNMENT_SENTENCE}

#### 📝 Summary

{PR_SUMMARY}

📊 **Findings:** {CATEGORY_COUNTS}

---
*🤖 Automated deep review — {N} files analyzed across {M} review passes, independently verified before posting.*
```

## Step 4 — Sort findings by category

Before generating comments and anchors, arrange all surviving findings strictly by category in this exact order:
- **Critical (C)** → **High (H)** → **Medium (M)** → **Low (L)** → **Improvement (I)**

All categories are always included — nothing is dropped based on count.

## Step 5 — Compute exact line anchors for every surviving finding

For each surviving finding (in the sorted order C → H → M → L → I) across all phase-2*.md files, you need path, line, side, and (for multi-line findings) start_line/start_side for the GitHub API. The findings files already recorded a line range from Phase 2/3a, but do not trust it blindly — recompute it yourself against the live diff as a final guard, since this is the last checkpoint before anything becomes an irreversible GitHub comment:

1. Run `gh pr diff {PR_NUMBER} -- <file>` once per distinct repo file path that has surviving findings (not once per phase-2*.md — a chunked file maps to one diff fetch shared across its chunks).
2. Walk each hunk (@@ -a,b +c,d @@ → track old_ln/new_ln) to confirm the finding's claimed line(s) land on real diff lines.
3. For a single-line finding: line = the new-file line number, side: "RIGHT" (or old-file line + side: "LEFT" only if it's specifically about a deleted line).
4. For a multi-line finding: start_line/start_side = first line of the range, line/side = last line of the range. Both sides should normally be RIGHT unless the finding is specifically about deleted code.
5. If a finding's line numbers don't actually resolve against the current diff (e.g., the PR was updated since Phase 2 ran, or a line-number error slipped through Phase 3a), do not post it as a line-anchored comment — instead, fold a one-line mention of it into the review body as an unanchored note, so the finding isn't silently lost but also doesn't crash the API call. Note this in your final report.

## Step 6 — Apply Posting Policy, then build 1-click suggestions (Independent fixes ONLY) vs Multi-file guidance

Before building `comments[]`, apply the **Posting Policy (strict)**: posted subset = all Critical + High, then fill remaining slots up to 8 from Medium → Low → Improvement; if C+H ≥ 8, posted subset = C+H only. Full category counts remain in the footer regardless. Filter `comments[]` to the posted subset; non-posted findings stay recorded/counted but get no line-anchored comment.

**CRITICAL REQUIREMENT: GitHub's native 1-click suggestion (` ```suggestion `) must ONLY be used for independent findings where no other files need touching. For any finding requiring multiple files to be edited, STRICTLY REMOVE the suggestion block and use a standard code block.**

1. **Independent Fixes (Use ` ```suggestion `)**:
   - When the fix is strictly self-contained within that single file/location (`Scope: Independent`), convert the suggested code into a ` ```suggestion ` fence.
   - Anchor `start_line` and `line` (or `line` alone for single-line) covering the exact lines being replaced in the new file (side: "RIGHT").
   - This renders the native 1-click **"Commit suggestion" / "Apply suggestion"** button for maintainers and authors directly on GitHub.
   - Do NOT add any collapsible auto-fix script tags or shell scripts inside the comment body.

2. **Multi-File Fixes (STRICTLY DO NOT USE ` ```suggestion `)**:
   - If fixing the issue requires modifying multiple files or touching other related files (`Scope: Multi-file`), **DO NOT USE ` ```suggestion `**. Auto-applying a single file's suggestion when other files require coordinated edits leaves the codebase in a broken state (failing types, broken contracts, missing callers).
   - Instead, provide the suggested change in a standard language code block (e.g. ` ```python ` or ` ```typescript `) and include a clear warning block:
     `> ⚠️ **Multi-file change required**: Do not auto-apply. This fix requires coordinated updates in related files: {files}.`

3. **Mandatory Issue Comment Format**:
   Every review comment in `comments[]` must follow one of these two structures:

   **Case A: Independent Fix (1-Click Suggestion)**
   ```markdown
   ### [{CATEGORY}] {7-10 words short conclusion of issue}

   {description}

   ```suggestion
   {complete replacement code}
   ```
   ```

   **Case B: Multi-File Fix (No Suggestion Block)**
   ```markdown
   ### [{CATEGORY}] {7-10 words short conclusion of issue}

   {description}

   > ⚠️ **Multi-file change required**: Do not auto-apply. Fixing this issue requires coordinated updates across related files ({related_files}).

   ```{lang}
   {suggested reference code}
   ```
   ```

## Step 7 — Assemble and post, exactly once

Build one JSON payload preserving the sorted category order (C → H → M → L → I) in the `comments` array:

{
  "commit_id": "{HEAD_SHA}",
  "body": "<the review body from Step 3>",
  "event": "COMMENT",
  "comments": [
    {
      "path": "path/to/file.py",
      "line": 131,
      "side": "RIGHT",
      "start_line": 123,
      "start_side": "RIGHT",
      "body": "### [CRITICAL] Memory leak in auth session handler on timeout disconnection\n\nSession socket remains unclosed when handshake times out, exhausting file descriptors under load.\n\n```suggestion\n<suggested replacement code>\n```"
    },
    {
      "path": "path/to/other_file.py",
      "line": 42,
      "side": "RIGHT",
      "body": "### [HIGH] Function signature update requires updating callers across services\n\nAdding required tenant_id parameter breaks un-migrated call sites across api and worker modules.\n\n> ⚠️ **Multi-file change required**: Do not auto-apply. Fixing this issue requires coordinated updates across related files (src/api/handler.py, src/worker/task.py).\n\n```python\n<suggested reference code>\n```"
    }
  ]
}

Notes:
- Apply the Posting Policy before finalizing `comments[]`: only the posted subset (per policy) becomes line-anchored comments; non-posted findings remain counted in the footer but are omitted from `comments[]`.
- Use ` ```suggestion ` ONLY for independent single-file fixes (enables GitHub's native 1-click `Apply suggestion` button).
- For multi-file fixes, STRICTLY REMOVE ` ```suggestion ` and use standard code blocks with the multi-file warning notice.
- Do NOT include any collapsible auto-fix script tags or shell scripts in the comment.
- The first example above is a multi-line finding (lines 123-131); the second is single-line (line 42 only) — omit start_line/start_side entirely for single-line comments.
- Use event: "COMMENT" — never REQUEST_CHANGES or APPROVE; this pipeline reports findings, it doesn't gate merges on the author's behalf.
- Write this JSON strictly to `{DOSSIER_DIR}/review-payload.json` (inside `C:\Users\bari2\Desktop\dossier\<repo-name>\PR-<number>\`) rather than inlining on the command line.

Post with:
gh api --method POST \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  /repos/{REPO}/pulls/{PR_NUMBER}/reviews \
  --input {DOSSIER_DIR}/review-payload.json

This must be one call. Do not post comments individually, do not split into multiple review submissions. If the call fails:
- A 422 almost always means one comment's path/line/side doesn't actually match the diff — re-check Step 5 for the offending entry, remove or fix it, and retry the whole call (not just that one comment).
- A 403/429 means a rate limit — check for a retry-after header and wait that long, or wait 60s and retry once. Do not hammer retries in a loop.

## Step 8 — Confirm and report

Read back the API response — it contains the review's html_url. Write a final summary to {DOSSIER_DIR}/phase-3b-final.md:

# Final Review Posted

- Review URL: {html_url}
- Confidence: {X}/10 — {CONCL_TAG}
- Findings posted: {total} ({category counts})
- Findings dropped at final anchor check (if any): {list, with reason}

Also update `{DOSSIER_DIR}/meta.json`: set `last_reviewed_sha` to `{HEAD_SHA}` and record the posted review ID and comment IDs (from the API response) so a future re-review can resolve fixed threads. Then report back to whoever dispatched you with the review URL and a one-line summary. This is the final step of the entire pipeline.
```
