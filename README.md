<div align="center">

# AGENTIC-SKILLS

**Install once. From then on the agent picks its own role and its own skill.**

It checks the tools it has, resolves which of the two roles it is, reads that role's skill, and starts work. You never name a file, never pass a path, never remind it.

</div>

---

## Two roles, one rule: exactly one skill per phase

| | |
|---|---|
| **Agent-A · Orchestrator** | Has a dispatch tool. Scopes work, partitions files, dispatches subagents, gates on a zero-finding audit. Never writes code itself. |
| **Agent-B · 1:1** | No dispatch tool. Owns one task end to end — implement, test, PR, poll to green. No delegation. |

Role resolves from the tools the agent actually has. An explicit instruction overrides it.

## Start here

| File | What it's for |
|---|---|
| [`setup.md`](setup.md) | **One-paste install.** Copy the prompt inside, paste it into your agent's chat — it clones the repo and wires everything up. |
| [`system-prompt.md`](system-prompt.md) | The pasteable system prompt. Generalized, de-identified, no personal data. Drop it into your harness's global instruction file. |

`system-prompt.md` is the payload. `setup.md` is the automated installer for it — you need the payload, and either install route delivers it.

## Every turn, unprompted

The system prompt makes the agent resolve its own role before it does any work, from the tools it actually holds — a dispatch tool means Agent-A, no dispatch tool means Agent-B. It reads that role skill, then matches the task against the inventory. Agent-B reads **one** general skill itself, or none when nothing applies, and that read is mandatory before its first edit. Agent-A never reads a general skill — it hands the exact path to a subagent, and a subagent gets **no** skill view at all when its task is non-code work.

| Turn | Who decides | What happens |
|---|---|---|
| 1 | Agent | Reads `/home/faizan/SKILLS/agent-orchestrator/skills.md` or `/home/faizan/SKILLS/agent-1to1/skills.md` |
| 2 | Agent | Agent-B reads one `/home/faizan/SKILLS/<skill>/skills.md` — or none. Agent-A assigns that path to a subagent |
| 3 | You | Nothing. No command, no file path, no reminder |

Routing is automatic. All skills live at `/home/faizan/SKILLS`, by explicit path, never by a loader.

## Domain skills

| Work | Skill |
|---|---|
| New frontend feature | `frontend/` |
| New backend or API | `backend/` |
| Existing UI being improved | `ui-ux/` |
| Bug or issue fix | `issue-fix/` |
| Any diff under audit | `review/` |

`frontend` and `backend` are for **new features only**. One skill, read in full, before the first edit — never rationalize a skip.

## Orchestrator skills (`open-*`)

The canonical, universal `open-*` suite consists of 5 orchestrator-only skills tracked directly in this repository. They are reserved exclusively for Agent-A to govern repository-wide issue discovery, triage, investigation, audit, and delivery pipelines across any codebase:

| Skill | Focus | Pipeline & Operation |
|---|---|---|
| `open-issue-finding` | Phase 1: Proactive Discovery | Proactive bug & vulnerability discovery, verified MREs, coordinated responsible disclosure |
| `open-issue-triage` | Phase 1: Issue Ingestion | Open GitHub issue harvesting, signal scoring, footprint classification, merge-ROI optimization |
| `open-pre-fix` | Phase 2: Pre-Fix Investigation | Pre-fix investigation, MRE reproduction, root cause analysis, blast radius mapping, and surgical fix planning |
| `open-post-fix` | Phase 4: Post-Fix Audit | Multi-lens post-fix audit (6 parallel lenses initial pass, focused re-audits) |
| `open-pre-pr` | Phase 5: Verification & Gates | Pre-PR verification, 6-stage hard gates, PR dispatch, CI & review-bot convergence loop |

### The 5-Phase Lifecycle & Operation

- **Phase 1 — Issue Finding & Sourcing (`open-issue-finding` / `open-issue-triage`)**: Hunt for new defects with verified MREs, or triage existing open GitHub issues to isolate high-ROI candidates. Real vulnerabilities route through private disclosure channels.
- **Phase 2 — Pre-Fix Investigation & Planning (`open-pre-fix`)**: Rigorous reproduction, root cause isolation, invariant verification, full blast-radius mapping, and architectural fix planning with disjoint file partitioning for 1 to max 3 fixers.
- **Phase 3 — Fix (Domain Skills)**: Fixers execute in parallel with strict disjoint file boundaries. Note that `open-issue-fix` does not exist; fixers are subagents that read domain skills (`issue-fix`, `ui-ux`, `frontend`, `backend`) armed with the Phase 2 Context Packet.
- **Phase 4 — Post-Fix Multi-Lens Audit (`open-post-fix`)**: Adversarial audit gate. First pass runs 6 parallel lenses (Security, Blast Radius, Performance, Concurrency/State, Idiomatic Style, Test Coverage). Gate requires 0 must-fix and 0 should-fix. Re-audits are focused and restricted strictly to modified files.
- **Phase 5 — PR Raise & Poll (`open-pre-pr`)**: Enforces pre-PR hard gates, builds standardized PR dispatches, and monitors CI and review bots (1.5 min interval, max 10 push updates, never merge).

> [!IMPORTANT]
> The `open-*` suite consists of private, orchestrator-only skills. They are loaded exclusively by Agent-A via explicit path (`/home/faizan/SKILLS/open-*/skills.md`). Agent-B is strictly forbidden from reading them, and Agent-A never delegates them to leaf subagents.

## Use it

**Fastest path.** Open [`setup.md`](setup.md), copy the whole prompt block, paste it into your agent's chat. Done — the agent detects your harness, clones this repo, ensures paths resolve to `/home/faizan/SKILLS`, and appends the system prompt to your global instruction file. It then reports the clone path and the file it wrote; run the "Verify it worked" checks in `setup.md` to confirm.

**Manual path**, if you'd rather do it yourself:

```bash
git clone https://github.com/kaiizer777/SKILLS.git ~/Desktop/SKILLS
```

Use the same destination `setup.md` names. All skills live at `/home/faizan/SKILLS` — use this fully expanded path, never the `~` form: `~/Desktop/SKILLS` works as a shell argument and fails inside a file path. If `~/Desktop` does not exist, clone into a directory of your choosing and substitute that path instead.

Then paste [`system-prompt.md`](system-prompt.md) into your harness's global instruction file:

| Harness | Global instruction file |
|---|---|
| Gemini CLI | `~/.gemini/GEMINI.md` |
| opencode | `~/.config/opencode/AGENTS.md` |
| Claude Code | `~/.claude/CLAUDE.md` |
| Cursor | `~/.cursor/rules/agentic-skills.mdc` (global) or paste into **Settings → Rules → User Rules** |
| Anything else | Whatever your harness calls its global instructions. `AGENTS.md` and `CLAUDE.md` are the two names most tools read — check its docs, don't guess the path. |

**The two role skills**, loaded **by explicit path**, not auto-discovered:

| Skill | Role |
|---|---|
| `agent-orchestrator/skills.md` | Agent-A, when the agent has a dispatch tool |
| `agent-1to1/skills.md` | Agent-B, when it does not |

Nothing to wire up by hand — `system-prompt.md` already names both by path and resolves the role from the tools the agent has. The paths it refers to are `/home/faizan/SKILLS/agent-orchestrator/skills.md` and `/home/faizan/SKILLS/agent-1to1/skills.md`.

Most skill loaders want `SKILL.md`. Do not rename these files — the system prompt and both role skills reference `skills.md` by exact path, and each role skill states that only files named `skills.md` are loadable. For auto-discovery, add a `SKILL.md` that points at the existing `skills.md` instead of renaming it.

## `system-prompt.md`

A single system prompt: role resolution, branch and commit rules, PR handling, verification discipline. All skill paths resolve directly to `/home/faizan/SKILLS`.

## `setup.md`

A copy-paste setup prompt. Paste it into your agent's chat and it detects your harness, clones this repo, ensures paths resolve to `/home/faizan/SKILLS`, appends the system prompt to the right instruction file, and reports back what it did. Verify the result yourself with the checks in `setup.md`.

## Conventions

- Branch prefixes `feat/` `fix/` `chore/` `refactor/` — never commit to `main`
- One clean commit per task
- **PRs are never auto-merged** — the owner merges
- Never `git stash` in a shared workspace
- Out-of-scope findings become separate issues, not inline fixes

## Not in version control

`env/` (live secrets) — excluded via `.gitignore` by design. The `open-*` orchestrator skills are tracked directly in the repository.

> Secret ever committed? **Rotate it.** Scrubbing history doesn't un-leak a value.

## License

MIT — © 2026 AGENTIC-SKILLS contributors.
