---
name: open-pre-pr
description: Run a strict, stage-gated adversarial audit on a finished code fix before opening a pull request against any production repository, then manage the automated review-bot and CI feedback loop until all CI checks pass and bot comments are resolved (never merging the PR). Use this whenever a fix is implemented and about to be raised as a PR, whenever asked to "prepare this for review," "audit this diff," "get this PR-ready," or "raise the PR," and whenever an open PR has new comments from any review bot (CodeRabbit, Greptile, Reviewdog, Danger, etc.) or failing CI checks. Do not open a PR, and do not treat a PR as finished once bots or CI have run, without going through this pipeline first.
---

# Open Pre-PR Audit Pipeline

## Why this exists

Diff-only review structurally misses two classes of bug — missing correctness or security invariants that depend on code outside the diff, and blast-radius damage 2–3 hops from the changed symbol — and that blind spot exists in every codebase, not just auth libraries. AI-authored code is also fluent enough to get rubber-stamped despite carrying more issues per PR than human code, at higher severity. This pipeline exists to structurally prevent both failures: it forces full-repo context before any lens runs, forces every lens to produce evidence of real adversarial work (not a skim), and refuses to call a PR "done" the moment it's opened, since that's before any external reviewer — bot or human — has actually tested it.

## Ground rules (apply to every stage)

- **No fixed subagent count anywhere in this pipeline.** The number of subagents per lens, per chunk, or per stage is an orchestrator judgment call based on diff size and risk, not a parameter of this skill. Never hardcode a count.
- **Gates are binary.** A stage is green or it is not. "Mostly passing," "minor issues noted," or "should be fine" are not green. Do not advance on a judgment call that a stage is close enough.
- **No stage starts before the previous stage is green**, including re-entry after a fix loop — Stage 3 is not a formality, it is a full re-run.
- **A clean result is never self-certifying.** A lens or a full pipeline pass that reports "no issues" must also report what it checked and at what granularity. An unexplained "no issues found" is treated as an incomplete report, not a pass — a well-prepared PR passing cleanly is the expected outcome, not proof the audit was thorough.
- **Chunk large diffs.** No single lens pass should ingest more than roughly 200–400 lines of diff in one shot — detection quality measurably degrades above that density regardless of model capability. Split by logical unit (file/module), run each lens per chunk, then synthesize across chunks before declaring the lens green.
- **Bulletproof PR Bar (upstream standard).** Strict, clean, minimal, professional edits only. Even minor loophole blocks PR — no "minor so ship it". If it would not survive upstream maintainers, it does not raise.
- **Mandatory Parallelism.** Use multiple parallel subagents , 3 parallel subagents is preferred for this pahse , its min count , u can increase no if there is a need like if the issue is bigger . Time is unlimited — thoroughness wins. Single-lens single-pass sign-off is a loophole.

---

## STAGE 0 — Contribution Guidelines Discovery (mandatory, runs first)

**Every OSS repository has its own submission rules, and they are not guessable.** A repo may require
a changeset for every `packages/**` edit, enforce Conventional Commit titles with a workflow that
hard-fails, demand a linked issue, require a specific label from `labeler.yml`, mandate a
breaking-changes declaration, forbid squash merges, or target a release branch other than `main`.
Carrying another project's rules into this one is how contributions get rejected — or worse, get
approved and reverted later.

**Do not hardcode any repository's rules from memory, or from a different project. Discover them,
write them down, and treat the written list as binding.**

### Sources to read — all of them, in this priority order

1. `CONTRIBUTING.md`, `.github/CONTRIBUTING.md`, `docs/CONTRIBUTING.md`, `CONTRIBUTING.*`
2. `.github/PULL_REQUEST_TEMPLATE.md` / `.yml`, `.github/PULL_REQUEST_TEMPLATE/`, or the repo's
   configured GitHub template
3. `.github/workflows/*.yml` — **read the triggers and the fail-fast steps.** Workflows are the only
   rules actually *enforced*; prose in `CONTRIBUTING.md` is advisory. A workflow named something
   like "Verify Changesets", "Validate PR title", "Release Notes Draft", or "Semantic Pull Request"
   defines a hard gate even when `CONTRIBUTING.md` never mentions it. The `on:` block also tells you
   what triggers each gate — a `push`-only workflow will never run for your PR at all.
4. `.changeset/config.json` and `.changeset/` — which packages require a changeset, allowed
   dependency graphs, and whether `patch` is capped on the default branch. Many repos forbid
   `minor`/`major` on `main`, which silently forces a public API addition into a `patch` release —
   surface that as a finding to the maintainer, don't fight the policy.
5. `.github/labeler.yml` or `labeler.yml` — the exact label set and path→label mapping
6. `CODE_OF_CONDUCT.md`, `SECURITY.md`, `docs/` contribution pages
7. The repo's own recently **merged** PRs — `gh pr list --state merged --limit 30` — as empirical
   evidence of what actually lands: title style, body structure, label set, linked-issue format.
   When prose and practice disagree, practice is the real rule.

### Conflict resolution — record which side you followed

- **Enforced workflow > prose.** If `CONTRIBUTING.md` says one thing and a workflow hard-fails on
  something else, the workflow is the rule.
- **More specific > more general.** `packages/x/CONTRIBUTING.md` beats the root `CONTRIBUTING.md`.
- **Recent merged PRs > both**, when prose, workflow, and practice all disagree.
- Never resolve a conflict silently — write down what conflicted and which side you followed.

### Produce a written artifact — `contribution_checklist`

Discovery is not finished until it exists as a table. One row per rule, nothing summarised:

| # | rule | source (file:line) | how to satisfy it | verified how |
|---|------|-------------------|-------------------|---------------|
| 1 | `packages/**` changes require a changeset | `.changeset/config.json:3`, `CONTRIBUTING.md:88` | add a changeset file under `.changeset/` | file present in the diff, name matches the repo's required pattern |

Extract at minimum, when present — do not limit yourself to this list:
changeset requirement and bump policy · PR title / Conventional Commit validation · required labels
and the labeler mapping · linked-issue or issue-first requirement · test requirements and any
`@see` / issue-link convention on regression tests · breaking-changes declaration · base branch
targeting (the default branch may not be `main`) · DCO / sign-off · contributor licence agreement
· required docs or changelog updates · required reproduction or regression evidence in the body ·
any label marking a PR as semver-significant · any workflow that hard-fails and therefore blocks merge.

**If a source does not exist, record the row as ABSENT.** Do not invent a guideline and do not
assume a rule from a different repository. "This repo has no PR template" is a finding about the
repo, not a gap in your work.

### Carry it through every stage

The checklist is created once, here, and handed verbatim to Stages 1–5 — carried forward as a binding
definition of done across the entire lifecycle. Any later stage may **append** a row when
it discovers a rule this pass missed; additions are appended, never silently dropped, and a
disagreement between the checklist and reality is itself a finding.

**A rule discovered here that this skill never mentions is still binding.** This document supplies
defaults and failure modes; the repository supplies the actual contract. Where they disagree, the
repository wins.

---

## STAGE 1 — Pre-Audit Context Assembly

**Objective:** assemble everything an audit lens needs so that Stage 2 never operates diff-only and never has to rediscover basics mid-audit.

Assemble and hand forward as a single context package:

1. **The original issue/request** the fix addresses, verbatim.
2. **The full diff**, plus — critically — the surrounding code the diff doesn't show: the complete body of every changed function/file, not just the hunks.
3. **A call/dependency graph slice around every changed symbol.** Build or query the repo's dependency graph (import/call/inheritance edges) and do a depth-bounded traversal (roughly 5–10 hops) backward from each changed symbol (who calls this?) and outward from its callers. Flag any changed symbol with disproportionate fan-in ("god node" — e.g. shared middleware, a widely-imported utility) as automatically high-risk regardless of how small the edit looks. Small diffs to high-fan-in code are exactly the case where "the change is small so risk must be low" fails.
4. **Existing tests** that cover the changed code, and whether the fix added/updated tests for the new behavior.
5. **Project conventions**: the `contribution_checklist` from Stage 0 is attached and every row is
   treated as a binding requirement, plus any sibling-code conventions the diff should match (e.g.
   how similar routes/handlers in this repo already implement the same control). If this stage
   discovers a rule Stage 0 missed, append it to the checklist rather than absorbing it silently.
6. **A written author-intent narrative**, forced at this stage, not skipped: **what** changed, **why**, and **how it was verified** (what was actually run, not "tests pass"). This measurably lowers the defects a later reviewer finds, because it forces the author to justify each hunk before anyone goes looking for problems.

**Stage 1 is green when:** every item above exists as a concrete artifact (not "the author will remember this") and is handed to Stage 2 as one package — except the intent narrative, which is withheld from the adversarial lenses themselves (see Stage 2 information-asymmetry rule) and used only by the orchestrator for its own gap-checking.

---

## STAGE 2 — Multi-Lens Strict Adversarial Audit

**Objective:** run the fix through every lens that top PR-bots and top human reviewers converge on, each one actively hunting a reason to reject.

### Posture, non-negotiable for every lens

- Each lens's subagent(s) operate as a hostile reviewer, not a collaborator: "assume this will be attacked / assume you have zero tribal knowledge / assume you are trying to break this in production."
- **Do not give adversarial lenses the Stage 1 intent narrative.** Review the artifact in isolation, the way a future attacker or an unfamiliar maintainer would — they won't have the author's stated intent either. The orchestrator uses the narrative separately, afterward, only to check for gaps between what was claimed and what each lens actually found.
- **Separate finding from fixing.** A lens reports and evidences a problem; it does not patch it inline. Mixing the two makes it impossible to tell afterward whether the original work was sound.
- **Every lens must produce one of two things, never silence:** (a) at least one specific, evidenced finding, or (b) an explicit "nothing found" statement that names exactly what was checked and at what granularity. A lens that returns a bare "LGTM" has failed to run and must be re-run — an instructed-to-find-problems agent that finds literally nothing has probably not looked hard enough, and a silent pass carries no information either way.
- **Order matters for security-sensitive, high-fan-in, public-API, or migration changes: write the failure narrative before the checklist.** Have the lens first write out, in plain prose, how this change ships and breaks production ("X returns 500 on every request because...", "a user reaches another user's data because..."), and only then run its structured checklist. Narrative-first surfaces scenarios a checklist alone won't anticipate; checklist-first tends to produce shallow, generic findings.

### Required lenses

| Lens | Persona | Core question |
|---|---|---|
| Security / Auth Auditor | Security Auditor / Saboteur | Will this be exploited, and does it regress a hardcoded security control? |
| Blast-Radius / Impact Analyst | — | What breaks 2–3 hops from this change that the author's own mental model doesn't reach? |
| Correctness / Functionality Skeptic | Skeptic | Does this do what was intended *and* what the user needs, including edge cases and concurrency? |
| API-Contract / Breaking-Change Reviewer | Architect | Does this change a public surface, and who downstream (visible or invisible) consumes it? |
| Readability / Complexity Reviewer | New Hire / Minimalist | Can someone with zero tribal knowledge follow this, and is any of it unjustified complexity? |
| AI-Specific-Failure-Mode Reviewer | — | Does this carry a bug class that specifically comes from AI-authored code rather than human code? |
| PR-Hygiene / Commit Reviewer | — | Is the presentation (description, commits, scope) itself acceptable, independent of the code? |

The **Security/Auth Auditor** and **Blast-Radius** lenses carry extra weight: these two categories are the documented, convergent blind spot of diff-only review (human and bot alike). Any repo with security-sensitive code paths (auth, payments, data access, crypto) shares this risk profile — a regression here has outsized blast radius for every downstream consumer by construction.

### Security — hardcoded floor (inlined; applies regardless of repo domain)

This floor is carried directly in this skill so a missed read of any external reference file can never leave the single most safety-critical lens undefined. Any confirmed violation below is an automatic REJECT, regardless of what any other lens concludes.

The security floor covers two tiers: **General OSS Security Patterns** (mandatory across every repository, including libraries, CLI tools, and background workers) and **Authentication, Session & Cryptographic Controls** (mandatory whenever auth, session, or crypto surfaces exist).

#### 1. General OSS Security Patterns (Mandatory for ALL codebases)

- **Command Injection Prevention:**
  - Never interpolate or concatenate user/untrusted input into shell command strings.
  - Subprocess execution (`exec`, `spawn`, `popen`, system calls) must pass arguments as discrete arrays/vectors with shell interpretation disabled (`shell: false`).
  - If dynamic shell invocation is strictly unavoidable, inputs must be strictly validated against rigid character allowlists (e.g., `^[a-zA-Z0-9_-]+$`).
- **Path Traversal Checks:**
  - All filesystem operations (`fs.readFile`, `path.join`, `path.resolve`, archive extraction, stream destinations) using user-influenced paths must verify that the canonicalized target stays strictly within the intended root directory.
  - Enforce boundary assertions (e.g., verifying `path.relative(root, resolved)` does not start with `..` and is not absolute) and reject unescaped `..`, null bytes (`%00`), and unexpected symlinks.
- **SSRF (Server-Side Request Forgery) Mitigation:**
  - Outgoing HTTP/network requests (`fetch`, `axios`, webhook triggers, URL importers) consuming user-provided URLs must strictly allowlist protocols (`http:` and `https:` only).
  - Explicitly block or filter internal, private, loopback, and cloud metadata network destinations (e.g., `127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.169.254`, `::1`, `localhost`).
  - Protect against DNS rebinding / TOCTOU IP resolution when making security-critical fetch requests.
- **Prototype Pollution Guards:**
  - Recursive merge, deep clone, or dynamic property-setting functions (e.g., `setByPath`, query string parsers, recursive dictionary extends) must explicitly reject or sanitize prototype poisoning keys (`__proto__`, `constructor`, `prototype`).
  - Use `Object.create(null)` for pure dictionaries or use native maps where appropriate.
- **ReDoS (Regular Expression Denial of Service) Prevention:**
  - Regular expressions evaluated against untrusted input must not contain catastrophic exponential or polynomial backtracking patterns (nested quantifiers like `(a+)+`, overlapping alternations like `(a|ab)+`).
  - Enforce strict input length bounds prior to regex evaluation or use safe, linear-time regex engines/timeouts where available.
- **Secret & Token Leakage Prevention:**
  - Zero hardcoded secrets, private keys, API keys, bearer tokens, or database credentials anywhere in source code or committed test fixtures (use mock placeholders like `test-token-dummy` or mock env vars).
  - Zero leakage of sensitive data (passwords, tokens, authorization headers, private keys, PII) into application logs, console output, exception stack traces, or client-facing unhandled error payloads.

#### 2. Authentication, Session & Cryptographic Controls (Mandatory when auth/session surfaces exist)

- **CSPRNG Identifiers:** Session/auth tokens, CSRF tokens, and nonce identifiers are generated via CSPRNG (>=64 bits entropy) — never `Date.now()`, `Math.random()`, or any non-cryptographic PRNG.
- **Session Fixation Prevention:** A new session ID is issued and the old one invalidated immediately on login (session regeneration) — repopulating a pre-login session ID under the authenticated user is session fixation.
- **Cookie Hardening:** Every auth cookie sets `HttpOnly`, `Secure`, and `SameSite=Strict` (or `Lax` with a written justification), plus a bounded `maxAge`.
- **JWT Integrity:** JWT verification explicitly allow-lists algorithms and always calls `verify()`, never `decode()`-only, for any value feeding an auth decision; `exp`, `iat`, `nbf`, `aud`, and `iss` claims are all checked.
- **Password Hashing:** Password hashing uses bcrypt (cost >=12) or Argon2 only — MD5/SHA-1/SHA-256/DES/3DES/RC4 for password storage is an automatic fail.
- **IDOR / Ownership Predicates:** Every data-mutating or data-returning route traces back to a shared, centralized authorization check with an explicit ownership predicate on the resource — "is this user authenticated" alone, with no check on *this specific resource ID*, is IDOR.
- **Cross-File Consistency:** The diff's security controls (middleware, hashing cost, session duration, authorization pattern) match what every sibling route/handler in the repo already uses.

**Note on applicability:** The **General OSS Security Patterns** apply universally to every repository and can never be skipped. If a repo has no authentication, session, or cryptographic surface (e.g., a pure CLI parser or math utility), only the auth/session-specific controls may be marked N/A, and that exemption must be explicitly documented at Stage 1 with written justification. Never skip the Security lens as a whole.

### Scope-creep check (runs as part of the PR-Hygiene lens, not as a line-count gate)

Line count alone is never grounds to flag a PR — "patch too large" is a near-irrelevant rejection reason on its own. What correlates with real rejections is unrelated changes bundled into the diff. For every changed file/hunk, require a one-line tie-back to the Stage 1 issue/intent. Any hunk that can't be tied back gets flagged for removal or a split PR, regardless of how small it is.

### Triage — mandatory before any finding leaves Stage 2

Raw lens output over-reports by design (an instructed-to-find-problems agent finds problems whether or not they're real). Before anything is acted on:

1. Every finding gets an explicit verdict: **PASS** (not a real issue, dismissed), **CONTESTED** (plausible, needs a second lens or human judgment), or **REJECT** (confirmed, blocks merge).
2. Every PASS and CONTESTED verdict carries a one-line, falsifiable, code-referencing rationale — "dismissed because X" is not sufficient; it must point at the specific line/behavior that resolves the concern.
3. REJECT is reserved for findings with clear evidence; a REJECT from the Security lens against the hardcoded checklist above is never downgraded by another lens's disagreement.
4. **CONTESTED is a staging state, not a resting state.** It must be escalated before Stage 2 can go green: spawn one additional, independent pass with no visibility into the original finding, the dismissal rationale, or which lens raised it, and have it adjudicate the evidence directly against the diff. That pass resolves to PASS or REJECT — never back to CONTESTED. If the adjudicator is itself unsure, treat it as REJECT: an unresolved ambiguity on a merge-blocking question defaults to the stricter outcome, not the more convenient one.

**Stage 2 is green when:** every lens has reported (finding or explicit clean statement), every finding has a triage verdict, zero REJECT-level findings remain open, and zero CONTESTED findings remain unescalated.

**Stage 2 is NOT green, full stop, if any one of these is true regardless of what other lenses concluded:**
- Any hardcoded item in the Security checklist above is a confirmed regression.
- Any lens returned silence instead of a finding-or-explicit-clean-statement.
- Any hunk fails the scope-creep tie-back check.
- Any CONTESTED finding has not been escalated to a resolving PASS/REJECT verdict.

**Any single REJECT sends the fix back out of this skill's scope** — flag exactly what needs fixing and why, and do not proceed to Stage 3. This skill does not implement the fix; it gates on whether one is needed.

---

## STAGE 3 — Convergence Check

**Objective:** confirm all lenses are green **simultaneously, in the same pass** — not green at different times across different fix rounds.

After any fix-back-and-forth triggered by Stage 2, **fully re-run Stage 2 from scratch** against the updated diff. Do not reuse prior lens results, even for lenses that weren't touched by the latest fix — a fix for one lens's finding can silently reopen another lens's earlier clean verdict (e.g., a correctness fix that adds a new code path the Blast-Radius lens hasn't seen). "Green across different passes at different times" does not satisfy this stage; only a single pass where every lens is green at once does.

**Stage 3 is green when:** one complete Stage 2 run, against the current diff, returns zero open REJECTs and zero unescalated CONTESTED findings across all lenses. Loop Stage 2 → fix → Stage 3 until this holds. Do not proceed to Stage 4 on anything short of this.

**Stall safeguard:** if the identical REJECT (same lens, same underlying issue) survives three full Stage 2 → fix → Stage 3 cycles, stop looping — that's a signal the fix isn't actually addressing the finding, not that the finding is wrong. Escalate: report the finding and all three attempted fixes, and why each one failed to clear it, to the orchestrator/human for a decision, rather than attempting a fourth blind fix-and-rerun.

---

## STAGE 4 — PR Raise

**Objective:** only once Stage 3 is unanimous green, produce the actual PR.

### PR description — use this template, every field required

- **What**: the change itself, in the author's own words (not a restatement of the diff).
- **Why**: the reasoning a diff can't convey. Must actually match the Stage 1 intent narrative *and* the final diff — re-verify it against the diff just before submission, since a description that's stale relative to the final code is rated worse by reviewers than no description at all.
- **Linked issue**: explicit reference (`Fixes #...`), not prose-only.
- **Breaking changes**: an explicit field that cannot be left blank — either the literal `NONE` or a structured note of what breaks and for whom. Silently hoping nobody notices a breaking change is not an option this template allows.
- **Testing performed**: concrete — what was actually run, against what inputs/scenarios — not "tests pass."
- **Risk tier** (Low/Medium/High/Critical): auto-High if the diff touches security-sensitive code (auth/session/crypto/payments/ACLs), a public API signature, or a database migration, regardless of line count.
- **Rollback plan**: required whenever Risk tier is High/Critical, or the change is non-trivially reversible. The bar differs by which of those is true:
  - *Non-trivially reversible* (migrations, destructive data changes, infra/permission changes): name the exact revert steps, the estimated time-to-revert, and confirm the rollback was actually exercised (e.g. the down-migration run against a test/staging DB) — not just written down. `git revert` alone does not undo a destructive migration.
  - *High/Critical risk but trivially revertible by `git revert`* (e.g. a risky code-only change touching no migration or infra): name the revert command and estimated time-to-revert. Do not require a rehearsed dry-run here — there's nothing to exercise beyond the revert itself, and demanding one doesn't match the actual risk.
- **Reviewer notes**: point a human reviewer at the riskiest part of the diff directly, don't make them find it.
- **API delta** (required if any public export/signature changed): explicit added/removed/changed list, plus a note on downstream impact assuming invisible external consumers exist, not just in-repo ones.

### Commit hygiene — verify before pushing

- Every commit is atomic: apply the "no 'and'" test — if the commit message needs "and" to describe it ("fixes A and B"), split it.
- Conventional Commits format (`type(scope): summary`; `feat`/`fix`/`refactor`/`chore`/`docs`/`test`/`revert`).
- Imperative, present tense ("add X," not "added X"/"adds X").
- Message explains **why**, not a restatement of what the diff already shows.

**Stage 4 is green when:** the PR is open, every template field above is filled with real content (not a placeholder), commit hygiene checks pass, and the PR number/URL is recorded for Stage 5.

---

## STAGE 5 — Post-Raise Bot-Feedback & CI Convergence Loop

**This stage is not optional and the PR is not done when Stage 4 ends.** CI checks and automated review bots respond on staggered timelines — typically 4–8 minutes after a push — so a single check after raising the PR is insufficient. This stage runs an explicit poll-until-stable loop across both **automated CI status checks** and **external review bots** (CodeRabbit, Greptile, Reviewdog, Danger, etc.).

> [!CRITICAL]
> **MANDATORY RULE: NEVER MERGE.**
> Under no circumstances does the agent merge the PR, enable auto-merge, or execute `gh pr merge`. The PR must be left open for the human repository maintainer to review and merge. The agent's responsibility ends strictly when the PR reaches green convergence.

### Concrete `gh` CLI Polling Commands (copy-pasteable)

Use these concrete commands to inspect PR status, CI checks, reviews, and bot comments:

1. **Comprehensive Snapshot (Status Checks Rollup, Reviews, Top-Level Comments):**
   ```bash
   gh pr view <number> --json statusCheckRollup,reviews,comments
   ```

2. **CI Check Runs Status (Terminal Overview):**
   ```bash
   gh pr checks <number>
   ```
   Or query individual check states programmatically:
   ```bash
   gh pr view <number> --json statusCheckRollup --jq '.statusCheckRollup[] | {name: (.name // .context), status: (.status // .state), conclusion: .conclusion}'
   ```

3. **Bot PR Reviews (Approval / Changes Requested States & Review Summaries):**
   ```bash
   gh api repos/{owner}/{repo}/pulls/{number}/reviews
   ```

4. **Bot Inline Diff Comments (File & Line Specific Suggestions):**
   ```bash
   gh api repos/{owner}/{repo}/pulls/{number}/comments
   ```

5. **Issue & PR Conversation Comments (Top-Level Bot Overview Comments):**
   ```bash
   gh api repos/{owner}/{repo}/issues/{number}/comments
   ```

### Identify configured bots and CI workflows first

Before the loop starts, inspect the repository configuration:
- Examine `.github/workflows/*.yml` to identify all required CI checks (build, test, lint, typecheck).
- Inspect bot configs (`.coderabbit.yaml`, `reviewdog.yml`, `Dangerfile`, `.greptile/`, etc.).
- Enumerate the expected bots and workflows explicitly — this defines the baseline for convergence.

### Loop Parameters

- **Initial Poll Delay:** 60 seconds after `git push` before the first poll, giving GitHub Actions and bot webhooks time to register.
- **Poll Interval:** 1.5 minutes (90 seconds) between successive status checks.
- **Stabilization Window:** Minimum 480 seconds (8 minutes) elapsed since push **AND** at least 3 consecutive stable polls (no change in bot comments or review states) before bot feedback is considered settled. Slowest bots take ~8 minutes to post.
- **Hard Cap on Push Updates:** Maximum of **10 push-updates** total across the PR lifecycle. If convergence is not achieved within 10 iterations, the loop halts immediately.

### Loop Logic & Execution Flow

```text
push_update_count = 1
push_commit()

LOOP:
    IF push_update_count > 10:
        ESCALATE_AND_STOP("Hard cap of 10 push-updates reached without convergence.")

    sleep(60s)                         # Wait for workflows and webhooks to trigger
    elapsed = 60
    stable_polls = 0
    last_comment_snapshot = NONE

    WHILE true:
        ci_status = poll_ci_status(pr)         # gh pr view <number> --json statusCheckRollup
        bot_reviews = poll_bot_reviews(pr)     # gh api .../pulls/<number>/reviews
        bot_comments = poll_bot_comments(pr)   # gh api .../pulls/<number>/comments + issues/<number>/comments

        current_comment_snapshot = hash_set(bot_reviews + bot_comments)

        IF current_comment_snapshot == last_comment_snapshot:
            stable_polls += 1
        ELSE:
            stable_polls = 0
            last_comment_snapshot = current_comment_snapshot

        # CI Failure fast-break: If any required CI check has definitively failed, do not wait 8 minutes
        IF ci_has_hard_failure(ci_status):
            BREAK (to fix CI failure immediately)

        # Full stabilization condition: 3 identical polls AND >= 8 minutes elapsed
        IF stable_polls >= 3 AND elapsed >= 480:
            BREAK

        sleep(90s)                             # 1.5-minute poll interval
        elapsed += 90

    # 1. Evaluate CI Exit Criteria
    IF NOT ci_all_green(ci_status):            # statusCheckRollup state != SUCCESS
        handle_ci_failure(ci_status)           # Apply targeted fix or escalate if blocking
        push_update_count += 1
        push_commit()
        GOTO LOOP

    # 2. Evaluate Review Bot Comments
    open_findings = extract_unresolved_bot_findings(last_comment_snapshot)

    IF len(open_findings) == 0 AND ci_all_green(ci_status):
        DONE — Stage 5 is GREEN. PR is fully converged. Do NOT merge.

    # 3. Triage & Address Bot Findings
    blocking_fixes_needed = false
    FOR EACH finding IN open_findings:
        verdict = triage_bot_finding(finding)  # Major/Critical vs Cosmetic Nit
        IF verdict == FIX_REQUIRED:
            apply_code_fix(finding)
            rerun_relevant_stage2_lenses(finding)
            blocking_fixes_needed = true
        ELSE IF verdict == DISMISSED_NIT:
            record_falsifiable_dismissal_rationale(finding)

    IF blocking_fixes_needed:
        push_update_count += 1
        push_commit()                          # Fresh commit resets the review baseline
        GOTO LOOP                              # Reset stabilization window for new commit
    ELSE:
        # All open findings were verified cosmetic nits with written rationales, and CI is green
        DONE — Stage 5 is GREEN. PR is ready for maintainer. Do NOT merge.
```

### Crisp Exit Criteria

A PR satisfies Stage 5 and completes the pipeline **only** when all of the following conditions hold simultaneously on the latest commit:

1. **CI Checks Green:**
   - The PR's `statusCheckRollup` state is strictly `SUCCESS`.
   - All required CI workflows, tests, linters, and typechecks report `conclusion: success`.
   - Zero checks in `FAILURE`, `TIMED_OUT`, `CANCELLED`, or `ACTION_REQUIRED`.
2. **Review Bot Convergence:**
   - The PR has passed the full stabilization window (at least 8 minutes and 3 consecutive stable polls).
   - Zero open/unaddressed major, critical, or app-breaking findings across all configured review bots.
   - Any unaddressed comments are verified cosmetic nits accompanied by explicit, falsifiable rationales.
3. **Iteration Cap Respected:**
   - Total push updates <= 10.
4. **Maintainer Preservation:**
   - The PR remains open. The agent has **not** merged the PR.

### Bot Finding Triage Rules

When evaluating bot findings:
- **Major / Critical / App-Breaking Findings:**
  - Logic bugs, edge case omissions, missing security controls, performance regressions, broken test fixtures, API mismatches.
  - **Action:** Must be fixed in code. Never dismiss. Apply fix, re-verify with relevant Stage 2 lenses, commit, push, and restart stabilization.
- **Superficial Cosmetic Nits:**
  - Stylistic opinions that conflict with repository formatting/lint configs, pedantic docstring rephrasings, or suggestions that contradict project conventions.
  - **Action:** May be dismissed **without** a code change *only* if accompanied by a one-line, falsifiable rationale referencing repo code or conventions (same standard as Stage 2 triage). Reflexive dismissal to escape the loop is forbidden.

### Stop-and-Report Protocols (Escalation Gates)

Stop looping immediately and escalate to the human/orchestrator under any of the following conditions:

1. **Blocking CI Failure:**
   - If CI fails due to infra/runner outages, external service downtime, missing repository secrets, upstream base branch breaks, or a complex failure that cannot be resolved after 2 targeted fix attempts:
   - **Protocol:** Immediately halt. Run `gh pr checks <number>` and fetch relevant run logs via `gh run view <run-id> --log-failed`. Report the exact workflow name, failing job URL, and error log snippet.
2. **Hard Cap Reached (10 Push-Updates):**
   - If 10 push-updates have been pushed and CI remains failing or bot comments remain unresolved:
   - **Protocol:** Stop immediately. Do NOT push an 11th commit. Produce a structured escalation summary containing:
     * PR link and current commit SHA
     * Current CI statusCheckRollup status
     * List of unresolved bot comments and reasons for non-convergence
     * Chronological log of fixes attempted across iterations 1–10
3. **Stall Safeguard (Repeated Bot Finding):**
   - If the identical bot finding (same bot, same file/line or pattern) reopens across 3 consecutive push-fix cycles:
   - **Protocol:** Stop looping. The fix is failing to address the bot's underlying check. Escalate to the human with the bot's verbatim text and all 3 attempted diffs.

### Mandatory Rule: Never Merge

**Never execute `gh pr merge`, never squash, never rebase, and never trigger auto-merge.**
Once Stage 5 reports green:
- Post a final summary to the orchestrator/user noting:
  * PR URL and branch name
  * Final CI checks status (`SUCCESS`)
  * Bot review convergence summary
  * Ready for maintainer review
- Leave the PR in open status for the project maintainer.
- **Delete the pre-fix findings `.md` file in `/home/faizan/better-auth-dossier/`** as its work is done.

---

## Anti-patterns — do not do these

- **Do not let any stage's "no issues found" stand alone.** It must always come with what was checked and at what granularity; an unexplained clean pass is an incomplete report, not a guarantee.
- **Do not let a single lens operate diff-only.** If a lens never looked past the changed hunks, it is disqualified from signing off on Security or Blast-Radius specifically — those categories require the Stage 1 call-graph context by construction.
- **Do not feed adversarial lenses the author's intent narrative.** It defeats the information-asymmetry an attacker or an unfamiliar maintainer would actually have.
- **Do not fix during a Stage 2 review pass.** Finding and fixing are separate steps; mixing them makes the original review unverifiable.
- **Do not gate on line count.** Scope creep (unrelated bundled changes) is the real problem; a large diff that's entirely in-scope is not itself a defect.
- **Do not treat "the change is small" as evidence of low blast radius.** Small edits to high-fan-in/shared code are exactly the case that gets missed this way.
- **Do not dump an entire large diff into one lens pass.** Chunk by logical unit; a single oversized pass is disqualified even if its output looked clean.
- **Do not accept a bare "LGTM" from any lens.** Silence is not a finding and is not a pass — re-run it.
- **Do not skip general OSS security checks (injection, traversal, SSRF, prototype pollution, ReDoS, secret leaks).** These apply to every codebase regardless of whether auth is present.
- **Do not treat Stage 5 as a single wait-then-check.** Bots respond on staggered timelines; a loop that checks once and declares victory will systematically miss slower bots.
- **Do not dismiss a bot comment just to close out the Stage 5 loop faster.** That is the rubber-stamping pattern this pipeline is built to prevent — dismiss only with a real, falsifiable rationale, same bar as Stage 2 triage.
- **Do not exceed the 10 push-update hard cap.** Exceeding 10 iterations indicates systemic divergence; stop and escalate.
- **Do not merge the pull request.** Merging is strictly reserved for the repository maintainer. Never call `gh pr merge` or enable auto-merge.
- **Do not let Stage 3 reuse old lens results.** A fix for one finding can silently reopen another lens's earlier clean verdict; only a fresh, simultaneous all-green pass counts.
- **Do not leave a CONTESTED finding unresolved and call Stage 2 green.** CONTESTED is a staging state, not a verdict — it must be escalated to an independent adjudicating pass that resolves it to PASS or REJECT.
- **Do not loop Stage 3 or Stage 5 indefinitely against a finding that keeps surviving fixes.** Three failed fix attempts on the same issue is a signal to escalate to the orchestrator/human, not a reason to try a fourth time the same way.
- **Do not hardcode a subagent count anywhere in this pipeline.** More reviewers or more passes alone has a documented ceiling; what matters is that each lens has a distinct, well-specified job, not headcount.
- **Do not skip the Security lens without a written justification** that explicitly states the repo has no security-sensitive code paths. Never skip it silently.

---

## Hard gate summary

| Stage | Exit condition |
|---|---|
| 0. Guidelines Discovery | Every guideline source read (CONTRIBUTING, PR template, all workflow triggers, changeset config, labeler, CoC/security, recent merged PRs); `contribution_checklist` written with one row per rule including ABSENT rows and resolved conflicts. |
| 1. Context Assembly | Full package assembled (issue, diff + surrounding code, call-graph slice, tests, conventions, intent narrative, contribution_checklist) and handed to Stage 2. |
| 2. Multi-Lens Audit | Every lens reported finding-or-explicit-clean; every finding triaged, with zero CONTESTED left unescalated; zero open REJECTs; zero hardcoded Security regressions (general OSS & auth); zero scope-creep hunks. |
| 3. Convergence | One single, fresh, complete Stage 2 pass shows all lenses green simultaneously, with the stall safeguard applied if a REJECT repeats 3 cycles running. |
| 4. PR Raise | PR open, every description field populated with real content, commit hygiene verified. |
| 5. Bot-Feedback & CI Loop | CI statusCheckRollup strictly SUCCESS, full stabilization window elapsed with zero open blocking bot comments (all nits triaged with written rationales), hard cap of 10 push updates not exceeded, PR left open for maintainer (never merged), and temporary pre-fix dossier file in `better-auth-dossier` deleted. |

No stage is skipped. No stage begins before the previous one reports green by the condition above — not an approximation of it.
