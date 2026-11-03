# Skill — Personal-Proj

**Scope:** Owns everything related to working on **Saif's personal projects**. This skill is loaded by the orchestrator **only after** the user (Saif) confirms the target project is one of his personal repos, not an open-source contribution.

**This skill is independent.** It contains the full personal pipeline, habits, and PR convergence loop. It does not require reading `agent-orchestrator/skills.md` to function — it only cross-links to the orchestrator core for inventory references.

**Hard rule:** The 5-Phase OSS pipeline, all `open-*` orchestrator-only skills, and the heavy 6-lens audit are **strictly forbidden** in personal projects. This skill exists precisely to enforce that boundary.

**Related skills:**
- Parent / Core: `C:/Users/bari2/Desktop/SKILLS/agent-orchestrator/skills.md` — role, tool lockdown, skill inventory
- Sibling: `C:/Users/bari2/Desktop/SKILLS/OS-Contribution/skills.md` — OSS pipeline (mutually exclusive)
- General skills referenced by dispatched fixers (read from inventory only — never read here):
  - `C:/Users/bari2/Desktop/SKILLS/frontend/skills.md`
  - `C:/Users/bari2/Desktop/SKILLS/backend/skills.md`
  - `C:/Users/bari2/Desktop/SKILLS/ui-ux/skills.md`
  - `C:/Users/bari2/Desktop/SKILLS/issue-fix/skills.md`
  - `C:/Users/bari2/Desktop/SKILLS/review/skills.md`

---

## Core Personal Habits (Mandatory)

1. **Always Raise a GitHub Issue on Discovery:** Whenever you encounter an issue/bug in a personal project — whether during reading files, fixing, auditing, or any other task — **immediately raise a GitHub issue** (`gh issue create`). Never fix out-of-scope issues silently or leave them undocumented.
2. **Short, Bite-Sized Issues Only (No Mega Issues):** Never create or tackle monolithic, massive issues. Break complex problems down into small, isolated, bite-sized GitHub issues. Fix them **one at a time** to eliminate cognitive mess and maintain repo cleanliness.
3. **Maximize High Commit Count (Boost GitHub Profile Contribution Count):** Saif merges personal PRs strictly using **"Create a merge commit"** (`gh pr merge --merge`), which preserves and credits every single commit toward total GitHub profile contributions. Therefore, **prioritize a high number of commits in personal project PRs**. Never make one giant squashed commit — subagents must commit frequently, logically, and progressively across steps (e.g. tests, types, implementation steps, refactor, doc updates) to maximize total contribution count.

---

## Personal Project Pipeline Workflow

### 1. Issue Sourcing
Saif provides the specific GitHub issue to work on.

### 2. Context Gathering (2 Subagents)
- Dispatch **2 parallel scout subagents** to achieve 1001% context on the issue, root cause, and dependencies.
- The fixer subagent must have complete certainty with **zero assumptions**.

### 3. Fix & PR Creation (Fixer Subagent)
- After the 2 context scouts report back, dispatch **1 fixer subagent** equipped with the full context packet and the matching general skill (`backend`, `frontend`, `ui-ux`, or `issue-fix`).
- Fixer implements the fix with **frequent, multiple commits (prioritizing high commit count to boost profile contributions)** rather than batching everything into one commit.
- **Self-PR:** At the end of its session, the fixer subagent **raises the PR itself** (`gh pr create`).

### 4. CI & Review Bot Convergence Loop (Orchestrator Owned)
- Orchestrator **never abandons the task after PR creation**.
- Review bots and CI take **3–12 minutes** to run.
- Run a strict **1.5-minute polling loop**: poll status → if bots flag defects or CI fails, dispatch a fixer to fix, commit, and push → resume polling until CI is 100% green and review bots have **0 outstanding issues**.
- Per the orchestrator core's lockdown rule: when the polling subagent claims CI and review bots are fully green, **redispatch a confirmation subagent** to independently verify the claim. If the verification contradicts the claim, keep the fix loop going. Do not accept a single unverified "all green" report at face value.

---

## Fixer Skill Selection (Personal Projects)

| The work is | Skill |
| --- | --- |
| Brand new frontend feature | `frontend` |
| Brand new backend, API, or server-side feature | `backend` |
| Existing frontend — UI/UX, visual, layout, motion, interaction | `ui-ux` |
| Existing frontend non-UI/UX work, existing backend work, any bug or issue fix | `issue-fix` |

`review` may be assigned to a separate subagent for diff/code review, in parallel, but is not mandatory.

---

## Dispatch Template (Personal Projects)

Every dispatch prompt carries all of these:

1. The task, scoped tightly enough for one subagent to finish alone.
2. Skill instruction: If and only if the subagent's task requires reading or writing code files, provide the exact absolute path of the **one** general skill to read. If the task is non-code work (e.g. pure Git operations, PR polling, web search, normal talk), explicitly state that no skill view is needed.
3. The repo path.
4. The full context packet — from the 2 scout subagents, never send a fixer in blind.
5. The scope boundary: which files this subagent owns, and which are off-limits.
6. Stop and report instead of guessing on any ambiguity.
7. For a PR subagent, additionally: the 1.5 minute interval, the fix/tolerate triage split, and the explicit instruction never to merge.

Every dispatch also carries the shared-workspace rules:

- Work only on your assigned branch. Never switch, checkout, rebase, merge, reset, revert, or clean another agent's work.
- Touch only the files this task owns. Anything unfamiliar in the tree belongs to another agent.
- **Never `git stash`.** Not to save work, not to unblock yourself, not ever.
- Dirty tree you did not create → leave it and report it to the user. Do not repair it.
- Before committing, confirm staged files are only yours. Never sweep up paths you did not touch.

When you parallelise, the file-ownership split is what keeps agents off each other. Parallel subagents that share a file will collide — split the work, never the intent.

If two branches are ever going to be live at once, they belong in separate worktrees or clones, not one shared checkout.

---

## Before Every Dispatch (Personal)

- Does the subagent's task require reading or writing code files? If yes, assigned exactly one general skill with full absolute path. If no (Git ops, PR polling, web search, normal talk), confirmed NO skill view is assigned.
- Task scoped so one subagent can finish it?
- No two parallel subagents own the same file?
- Fixers carry the full context packet from the 2 scout reports?
- High commit count directive given to the fixer (bite-sized, logical progression)?
- Self-PR directive given to the fixer (no separate PR subagent needed)?
- Polling loop planned with confirmation re-dispatch on green claims?

---

## Hard Nevers (Personal)

- Never use the 5-Phase OSS pipeline or read `open-*` orchestrator-only skills when working on personal projects.
- Never abandon a personal project PR after creation — the orchestrator must poll CI and review bots (1.5 min loop) until CI is green and review bots have 0 issues.
- Never accept a single unverified "all green" claim from the polling subagent — always redispatch a confirmation subagent.
- Never fix discovered issues out-of-scope without raising a short, bite-sized GitHub issue first (`gh issue create`).
- Never tackle or create massive, monolithic issues in personal projects — always break them into bite-sized, single-concern issues.
- Never batch a personal project PR into one giant squashed commit — preserve and increment commit count for profile contribution tally.
- Never merge a personal project PR with anything other than `gh pr merge --merge` ("Create a merge commit") to preserve every commit.
- Never implement anything yourself.
- Never let a subagent `git stash`, switch branches, or touch files outside its ownership boundary.