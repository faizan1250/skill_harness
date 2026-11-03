---
name: open-issue-triage
description: Systematically triage already-filed, open GitHub issues in any open-source repository and produce a ranked shortlist of the highest-ROI issues worth picking up — where ROI means a real, maintainer-confirmed bug or missing behavior with a fix footprint small enough (SMALL or MEDIUM tier only, LARGE is rejected) that a clean, well-scoped PR has a realistic merge probability. This skill does not discover new bugs and does not implement fixes (route to open-pre-fix for that) — it harvests, scores, filters, and ranks issues that already exist in the tracker, routing any issue with a high-severity, maintainer-unacknowledged signal to a human security review or private disclosure protocol instead of picking it up directly.
---

# Orchestrator Skill — Open Issue Triage

## PURPOSE

Surface the highest-ROI open GitHub issues in any target open-source repository that
are worth picking up and fixing. "Worth picking up" means: real, confirmed
bugs or missing behaviors that maintainers care about, with a fix footprint
small enough that a clean, well-scoped PR has a realistic merge probability.
This skill operates entirely on already-filed, open GitHub issues — it does
not find new bugs.

## SCOPE BOUNDARY

This skill operates exclusively on open issues already filed in the target
GitHub repository. It does not search for undiscovered bugs, does not reproduce
issues from scratch, and does not write fix code. Its single output is a ranked
shortlist of issues safe to hand to a fixer (such as an `open-pre-fix` pipeline).
If no issue is filed for a problem, that problem does not exist for this skill — route
exploratory bug discovery to dedicated testing or human triage.

## CORE DESIGN CONSTRAINT

Merge probability is the optimization target, not severity alone. A P0 bug that
requires an architectural overhaul is lower ROI than a P1 bug fixable in 20-50
lines, because the P0 will stall in review for months while the P1 ships
quickly. Fix footprint is a first-class gate at Stage 3, not a tiebreaker applied
at the end.

### Fix footprint tiers (use these exact labels — no synonyms, no hedging)

| Tier | Definition |
|------|------------|
| SMALL | ≤50 lines changed, 1-3 files, no public API surface change, no database/schema migration |
| MEDIUM | 50-200 lines changed, up to 5 files, possible additive public API change, no breaking changes |
| LARGE | >200 lines, architectural change, breaking change, or new module/subsystem/adapter/plugin |

**LARGE is rejected at Stage 3, unconditionally. Only SMALL and MEDIUM advance
to Stage 4 and beyond.**

## OPERATING PRINCIPLES

1. Never pick up an issue based on title alone. Read the full thread including
   all comments before scoring.
2. Maintainer silence ≠ maintainer approval. An issue with no maintainer
   response is not a confirmed bug, regardless of how clear or emphatic the reporter's
   description is.
3. Fix footprint must be estimated against the real, current source tree — not
   assumed from the issue description. The reporter often does not know how deep the
   fix goes or what abstractions it crosses.
4. A realistic merge probability is not a guarantee. It is an informed
   estimate based on maintainer engagement, scope, and project conventions —
   document the assumptions.
5. Volume is waste. Five well-scored, footprint-assessed issues beats a list
   of twenty vague candidates. The output is a shortlist, not a dump.

---

## REPO ADMISSION RULES — read once, before Stage 1

**An issue can be perfectly genuine and still be the wrong target for an external contributor.** Every
open-source project has its own contribution policies, triage lifecycle, and PR requirements that
decide which changes are acceptable, who may make them, and what artifacts a PR touching them must carry.
Score candidates against the target repository's live rules, not against assumptions.

Discover and read in the target repository:

- `CONTRIBUTING.md`, `.github/CONTRIBUTING.md`, `docs/CONTRIBUTING.md` — requirements for issue
  pre-assignment, RFCs before large changes, branch targets, changeset/changelog expectations, CLA/DCO
  requirements, and test expectations.
- `.github/ISSUE_TEMPLATE/` and `.github/labeler.yml` — label taxonomy and triage workflow conventions
  (e.g., `status: accepted`, `confirmed`, `needs repro`, `help wanted`, `good first issue`).
- `.github/workflows/*.yml` `on:` blocks — which gates run on fork/contributor PRs (linters, commit
  linters, typecheck, test suites). Note which workflows are `pull_request_target` (run with repo secrets,
  may prove nothing about fork code) versus `pull_request` (gated, and may require maintainer approval
  before executing for first-time contributors).
- `gh pr list --state merged --limit 30` (or `gh pr list --repo <owner>/<repo> --state merged --limit 30`)
  — empirical evidence of what maintainers actually merge (PR titles, size, commit conventions,
  discussion patterns).

This is **triage-scoped**: you are not producing a PR yet, so you do not need a full
contribution checklist. You need three questions answered:

1. **Is this even admissible under policy?** (e.g. Does the repository require maintainer pre-assignment
   or an approved RFC before submitting code? Is the subsystem in feature freeze or restricted to core maintainers?)
2. **Is the change type in scope for an external contributor PR at all?** A focused bug fix is almost always
   admissible; an architectural refactor, public API overhaul, or brand-new subsystem usually is not.
3. **What will a PR for this have to carry?** (regression test, changeset/changelog entry, linked issue syntax,
   labels) — so the triage ranking reflects real delivery cost, not just bug severity.

**Do not invent a policy that isn't written in the repo.** If a guideline source is absent, state that and
move on with standard open-source conventions. An issue whose remedy turns out to be disallowed by repository
policy should be dropped or flagged, not ranked.

---

## STAGE 1 — Issue Harvesting

**Goal:** produce a complete, de-noised candidate list. Incomplete pagination
at this stage silently corrupts every downstream score.

1. Fetch ALL open issues, fully paginated — never trust a single page of
   results:
   ```bash
   # In local clone of target repo:
   gh issue list --state open --limit 1000 \
     --json number,title,labels,createdAt,updatedAt,comments,url

   # Or targeting a remote repo explicitly:
   gh issue list --repo <owner>/<repo> --state open --limit 1000 \
     --json number,title,labels,createdAt,updatedAt,comments,url
   ```
   Cross-check the returned count against the repo's open-issue count shown in
   the GitHub UI / repo metadata. If they don't match, paginate manually via:
   ```bash
   gh api --paginate "repos/<owner>/<repo>/issues?state=open&per_page=100"
   ```
   *(Note: when paginating `/issues` via `gh api`, filter out PR objects where `.pull_request != null`).*

2. For each issue, pull the comment list and inspect each comment's
   `author_association`. Comments with `author_association` of `MEMBER`,
   `OWNER`, or `COLLABORATOR` count as maintainer signal. `CONTRIBUTOR`,
   `FIRST_TIME_CONTRIBUTOR`, and `NONE` do not.

   **Why `author_association` and not a hand-maintained handle list:**
   `author_association` is computed dynamically by GitHub per-comment at request time,
   so it stays correct as the maintainer roster changes. A hand-maintained
   list of maintainer usernames goes stale silently the moment someone joins
   or leaves the organization — do not substitute one.

3. Compute `last_maintainer_comment_date` per issue from maintainer comments.

4. Apply exclusion filters — drop the issue entirely if any of these hold:
   - No maintainer comment (`MEMBER`/`OWNER`/`COLLABORATOR`) in the last 6
     months (or repository-appropriate staleness window).
   - Labeled with negative triage labels: `wontfix`, `invalid`, `question`,
     `duplicate`, `works-as-intended`, or `as-designed`.
   - Is a feature request or discussion without maintainer bug confirmation
     (e.g., labeled `enhancement`, `feature`, `RFC`, `discussion` with no
     maintainer comment acknowledging it reflects broken behavior rather than
     a desired enhancement).

5. Output a raw candidate table: issue number, title, label set, days open,
   `last_maintainer_comment_date`, URL.

### Stage 1 Exit Gate
- [ ] **Repo admission rules read once, before scoring** (see above)
- [ ] Full pagination verified against repo-reported open issue count
- [ ] Every candidate has a computed `last_maintainer_comment_date` (or null)
- [ ] All three exclusion filters applied and documented per dropped issue
- [ ] Raw candidate table produced with all six fields populated

---

## STAGE 2 — Signal Scoring

**Goal:** score every surviving candidate on four independent axes, 1-5 each,
with a written reason per score. No axis may be scored without a cited piece
of evidence from the thread.

### Axis 1 — Maintainer engagement signal
- 5: Maintainer explicitly confirmed it's a bug AND applied an accepting/confirming label (e.g. `bug`, `confirmed`, `accepted`, `help wanted`, `status: triaged`)
- 4: Maintainer confirmed it's a bug in comments, no confirming label applied
- 3: Maintainer replied substantively but non-committally ("looking into it", "reproduced on my end", asking technical clarifying questions)
- 2: Maintainer reacted (emoji reaction) or assigned milestone/project only, no substantive comment
- 1: No maintainer response at all

### Axis 2 — Reproduction clarity
- 5: Minimal, runnable repro (standalone script, sandbox/playground link, minimal reproduction repo, or copy-pasteable test case)
- 4: Clear reproduction steps, not minimal but readily followable and verifiable
- 3: Repro steps present but missing environment, version, config, or dependency detail
- 2: Description of behavior only, no steps or reproduction code
- 1: Vague description or isolated stack trace only, cannot reconstruct the failure

### Axis 3 — Reporter credibility signal
- 5: 3+ independent reporters or confirmed duplicate issues pointing here
- 4: 2 independent reporters confirming the same failure
- 3: 1 reporter, corroborated by a maintainer or recognized active contributor
- 2: 1 reporter, no corroboration, but description is technically coherent and plausible
- 1: 1 reporter, report contradicts documented/expected behavior or stems from user configuration error

### Axis 4 — Fix urgency & impact
- 5: Critical defect: affects core correctness, data integrity/corruption, security vulnerability, or crash in standard execution path
- 4: High impact: affects functional correctness in an edge case or non-default configuration
- 3: Moderate impact: UX or DX degradation (confusing error message, bad default, missing validation)
- 2: Minor inconvenience, clean workaround exists and is documented
- 1: Cosmetic (typo, log message wording, documentation formatting)

### Scoring gate

- Minimum passing total to advance: **≥12/20**.
- **Hard veto, overrides the total score:** if Axis 4 = 5 AND Axis 1 = 1 (no
  maintainer acknowledgment), do not advance this issue through Stage 3.
  This is a potential undisclosed critical vulnerability or data-loss defect.
  Route it to a human security review / private reporting channel instead of
  triaging it as a normal public pickup candidate.

### Handoff protocol for vetoed issues

A hard-veto is not a silent drop. For every issue that triggers the Axis 4=5 /
Axis 1=1 veto, emit a handoff record before continuing:

```
ISSUE: <number> — <url>
VETO REASON: Axis 4 = 5 (core correctness/security/data integrity),
             Axis 1 = 1 (no maintainer response of any kind)
EVIDENCE: <one sentence citing the specific behavior described and why it
           reads as critical/security/data-integrity-affecting>
STATUS: FLAGGED FOR HUMAN REVIEW — DO NOT AUTO-ESCALATE
SUGGESTED NEXT STEP: A human maintainer or security triage engineer reviews
                      whether to handle this privately (e.g. GitHub Security
                      Advisory or private security contact) before public contribution.
```

**Do not auto-trigger public fix pipelines for vetoed issues.** This skill
treats unacknowledged critical vulnerabilities strictly as a flag for human review,
not an automated pickup task. Do not proceed past this stage for the vetoed issue,
and do not include it anywhere in the Stage 5 shortlist.

### Stage 2 Exit Gate
- [ ] All four axes scored for every candidate, each with a cited reason and thread quote/evidence
- [ ] Total computed and compared against the ≥12/20 threshold
- [ ] Axis 4=5 / Axis 1=1 hard veto checked for every candidate before anything advances
- [ ] Every veto produced a handoff record in the format above
- [ ] Vetoed issues routed to human review, not silently dropped or included in shortlist

---

## STAGE 3 — Estimated Fix Footprint Assessment

**Goal:** convert each surviving candidate's footprint from a guess into a
sourced estimate against the actual codebase.

1. Identify the source files implicated by the issue — from file paths or stack
   traces already cited in the thread, or by tracing the described behavior through
   the codebase (`grep`/`rg` for the relevant function, class, error message, or
   subsystem across the repo source tree, e.g. `src/`, `lib/`, or `packages/*/src/`).
2. Read those files in the local checkout. Do not estimate from the issue description alone.
3. Determine:
   - Number of files that would need to change
   - Whether the fix touches public API surface (exported types, function
     signatures, public CLI options, configuration contracts)
   - Whether it requires a database/storage schema migration or breaking change
   - Whether a comparable fix or adjacent test exists elsewhere in the codebase as
     a reference pattern (cite it if so — it significantly raises merge probability)
4. Assign SMALL / MEDIUM / LARGE per the table in CORE DESIGN CONSTRAINT.
   **Reject LARGE outright — do not carry it forward under any framing.**
5. Document the estimate with the specific file paths and line ranges read, not a vibe call.

### Stage 3 Exit Gate
- [ ] Every surviving candidate has implicated source files identified and
      actually read (paths listed)
- [ ] Footprint tier assigned using the exact SMALL/MEDIUM/LARGE definitions
- [ ] Every LARGE-tier candidate is dropped here, no exceptions
- [ ] Public-API-surface and schema/migration questions explicitly answered
      (yes/no), not left implicit

---

## STAGE 4 — Duplicate PR & Contributor Claim Check

**Goal:** avoid competing with in-flight, claimed, or already-rejected work.

1. Search open PRs referencing the issue number or describing the same fix:
   ```bash
   gh pr list --repo <owner>/<repo> --state open --search "<issue_number> in:body"
   gh pr list --repo <owner>/<repo> --state open --search "<keyword from issue title>"
   ```
2. If an open PR already addresses it:
   - **Skip** this candidate — do not compete with an in-flight fix.
   - Exception: the PR has been open and unreviewed (no maintainer comment,
     no review, no commit/CI activity) for **>30 days with no activity**.
     In that case the candidate may proceed, but note the stale PR in the
     final output so the fixer coordinates rather than duplicates blindly.
3. **Contributor-claim check — inspect comment thread, not just open PRs.** An issue gets claimed long
   before any PR exists. Scan the comments already pulled in Stage 1 for any contributor
   association (`CONTRIBUTOR`, `COLLABORATOR`, `MEMBER`, or community member) who stated an intent
   to work on it: "I'd like to work on this", "I'll take this up", "assigning myself", a linked WIP branch,
   or a linked draft PR.
   - If someone has claimed it, **do not race them.** Drop the candidate and name the claimant in
     the final output so the fixer can coordinate in-thread first. Racing another contributor is
     the fastest way to burn goodwill with maintainers and the community.
   - Proceed past a claim only on the same terms as the 30-day PR exception above: no linked
     branch, no draft PR, and no follow-up comment in >30 days. When you do, state so explicitly in the
     output so the coordination cost is visible rather than hidden.
   - A `CONTRIBUTOR` claim is a **coordination blocker, not a scoring input.** It must never be
     scored as maintainer approval under Axis 1, and it must not raise or lower the priority score.
4. Search closed/merged PRs for prior rejected or merged attempts at the same fix
   (`--state closed` returns both merged and closed PRs):
   ```bash
   gh pr list --repo <owner>/<repo> --state closed --search "<issue_number> in:body"
   gh pr list --repo <owner>/<repo> --state closed --search "<keyword from issue title>"
   ```
5. **Authorship split — your own prior PR is a different case from someone else's.** Resolve who you
   are first, then classify every PR found in steps 1 and 4 by author:
   ```bash
   CURRENT_USER=$(gh api user --jq .login)
   gh pr list --repo <owner>/<repo> --state all --author "$CURRENT_USER" \
     --search "<issue_number> in:body" --json number,title,state,mergedAt,closedAt
   ```
   Never hardcode a login — resolve it dynamically, so this stays correct across accounts and machines.
   Then:
   - **Yours, MERGED** → the issue is **already fixed. Drop the candidate outright.** Do not rank it.
     Selecting it wastes maintainer review time on a solved problem.
   - **Yours, OPEN** → already handled by step 2 (skip). Nothing new to add.
   - **Yours, CLOSED without merge** → **do not auto-retry.** You already spent that goodwill once, and
     re-filing the same approach reads as ignoring the feedback that closed it. Proceed only when at
     least one of these is true:
     - a maintainer (`MEMBER` / `OWNER` / `COLLABORATOR`) explicitly invited or encouraged a retry in
       the issue or PR thread, or
     - the new attempt is materially different in approach, not a reworded version of the same one.
     Either way, **name the prior PR number and why it was closed in the final output.** Silently
     re-attempting your own rejected PR damages your standing with upstream maintainers.
   - **Someone else's, CLOSED** → fall through to step 6 and apply the rejection-reason test.
6. If a closed PR attempted this and was closed without merging, read the rejection reason:
   - If the reason is still valid (e.g. maintainer states "we intentionally don't want this
     behavior changed") — **disqualify the candidate.**
   - If the reason no longer applies or was procedural (e.g. "abandoned by author", "needs tests",
     "missing docs", "wrong approach but right problem") — the candidate may proceed. State explicitly
     how the new attempt resolves the prior closure reason.

### Stage 4 Exit Gate
- [ ] Open PR search performed for every candidate (issue-number search AND keyword search)
- [ ] Contributor-claim check performed on every candidate's comment thread; every live claim resolved
      as skip-or-flagged-stale, and no contributor claim was scored as maintainer approval
- [ ] Closed PR search performed for every candidate
- [ ] Authorship resolved via `gh api user --jq .login` and every prior PR classified by author;
      own merged PR → candidate dropped entirely, own closed PR → not auto-retried without maintainer
      invitation or a materially different approach, with its number reported in output
- [ ] Every duplicate-in-flight PR resolved per the skip / 30-day-exception rule
- [ ] Every prior rejection reason read and explicitly classified as still-valid (disqualify) or
      resolved (proceed)

---

## STAGE 5 — Final Ranked Output

**Goal:** a short, defensible, fixer-ready shortlist. Max 5 issues.

### Empty-funnel exit path

If zero candidates survive to this stage — whether Stage 1 filtered
everything, Stage 2 scored everything below 12/20, Stage 3 classified
everything LARGE, or Stage 4 disqualified everything — **do not loop, retry,
or lower any threshold to manufacture output.** Emit a null-result report
instead:

```
RESULT: no qualifying issues found
FUNNEL: Stage 1 candidates: <n> → Stage 2 passed: <n> → Stage 3 passed (SMALL/MEDIUM): <n> → Stage 4 passed: <n>
DROPPED AT: <stage where the count hit zero, and the specific reason>
```

A null result reported honestly is a correct output. A padded or
threshold-lowered shortlist is not.

### Ranking formula
```
rank_score = (Axis4 × 2) + (Axis1 + Axis2 + Axis3 + Axis4) − footprint_penalty
footprint_penalty: SMALL = 0, MEDIUM = 1
```
Sort descending. Take the top 5 (fewer if fewer candidates survived — do not
pad the list).

**Note on the penalty sign:** the penalty must make MEDIUM score *lower* than
an otherwise-identical SMALL candidate — that is the entire point of
penalizing larger footprint under the CORE DESIGN CONSTRAINT. With the minus
sign in the formula above, that requires `MEDIUM = 1, SMALL = 0` (subtracting
1 for MEDIUM, 0 for SMALL). Do not use negative penalty values here — a
negative value subtracted becomes an addition and would rank MEDIUM *above*
SMALL, which inverts the intended preference.

**Worked example:** two candidates with identical signal scores
(Axis1=4, Axis2=4, Axis3=3, Axis4=4, total=15):
- Candidate A, SMALL: `(4×2) + 15 − 0 = 23`
- Candidate B, MEDIUM: `(4×2) + 15 − 1 = 22`

Candidate A ranks above Candidate B on footprint alone, as intended.

### Per-issue output format
For each shortlisted issue, output exactly:

1. **Issue number + URL + title**
2. **One-paragraph summary** of what's broken and why it matters
3. **Footprint tier** (SMALL/MEDIUM) + specific files to touch
4. **Why merge probability is realistic** — cite the maintainer signal
   (Axis 1 evidence) and the scope (Stage 3 evidence) together; this is not
   optional narrative, it is the load-bearing justification for why this
   issue and not another
5. **Suggested first step** for the fixer — a concrete starting action
   (e.g. "reproduce using the repro in comment #3, then trace
   `src/core/handler.ts` around line ~80")

### Stage 5 Exit Gate
- [ ] Max 5 issues in final output, no padding to reach 5
- [ ] Every issue carries all five required output fields
- [ ] Ranking formula applied and shown (not just a final order with no math)
- [ ] Any Stage 4 stale-PR or stale-claim exceptions are flagged inline in the summary

---

## COMMAND REFERENCE

Quick lookup for the commands used across stages. All assume `gh` is
authenticated against a token with read access to the target repository.

| Purpose | Command |
|---------|---------|
| List all open issues, full pagination (local repo) | `gh issue list --state open --limit 1000 --json number,title,labels,createdAt,updatedAt,comments,url` |
| List all open issues, full pagination (remote repo) | `gh issue list --repo <owner>/<repo> --state open --limit 1000 --json number,title,labels,createdAt,updatedAt,comments,url` |
| Manual paginated fallback | `gh api --paginate "repos/<owner>/<repo>/issues?state=open&per_page=100"` |
| Pull comments + author association for one issue | `gh api "repos/<owner>/<repo>/issues/<number>/comments" --jq '.[] \| {user: .user.login, assoc: .author_association, created_at: .created_at}'` |
| Search open PRs by issue number | `gh pr list --repo <owner>/<repo> --state open --search "<number> in:body"` |
| Search open PRs by keyword | `gh pr list --repo <owner>/<repo> --state open --search "<keyword>"` |
| Search closed PRs by issue number | `gh pr list --repo <owner>/<repo> --state closed --search "<number> in:body"` |
| Search own PRs on target repo | `gh pr list --repo <owner>/<repo> --state all --author "$(gh api user --jq .login)" --search "<number> in:body"` |
| Inspect recent merged PR patterns | `gh pr list --repo <owner>/<repo> --state merged --limit 30` |
| Grep source for an implicated symbol | `rg -n "<symbol or function name>" <source_path>` |

---

## ANTI-PATTERNS

- **DO NOT** score an issue higher because it sounds important or has a dramatic title. Score against
  the axes only.
- **DO NOT** skip Stage 4. A redundant or competing PR is wasted effort regardless of how
  good the fix is.
- **DO NOT** estimate footprint from the issue description alone. Read the
  source code.
- **DO NOT** include LARGE footprint issues in the output under any framing
  ("it's only large because X" is not a valid exception).
- **DO NOT** treat an issue as high-priority solely because it has many
  thumbs-up reactions. Reactions are user sentiment, not maintainer priority
  signal.
- **DO NOT** advance an issue with Axis 4 = 5 and no maintainer acknowledgment
  to Stage 3. Route it correctly to human review / private security disclosure instead.
- **DO NOT** race an active contributor who claimed an issue in the comments.
- **DO NOT** pad the Stage 5 shortlist to hit 5 entries if fewer candidates
  actually survived. A shortlist of 2 real candidates beats 5 with filler.
- **DO NOT** carry a disqualified closed-PR candidate forward by reframing
  the rejection reason as resolved without stating specifically how.
- **DO NOT** auto-retry your own previously closed PR without maintainer invitation or
  a materially different approach.
