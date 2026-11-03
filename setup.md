<div align="center">

# Setup

**One copy-paste block installs everything. No package manager, no build step.**

The repo is Markdown only — skills plus a system prompt. Installation = one `git clone` plus appending `system-prompt.md` to your agent's global instruction file.

Payload: [`system-prompt.md`](system-prompt.md) · Repo overview: [README.md](README.md)

</div>

---

## Pick your harness

Your harness reads global instructions from a fixed file. The install block below finds that file for you; this table is what it matches against.

| Harness | Global instruction file |
|---|---|
| Gemini CLI | `~/.gemini/GEMINI.md` |
| opencode | `~/.config/opencode/AGENTS.md` |
| Claude Code | `~/.claude/CLAUDE.md` |
| Cursor | `~/.cursor/rules/agentic-skills.mdc` (also pasteable into Settings → Rules → User Rules) |
| Other | ask your agent to locate its global instruction file, or use the manual steps below |

If a file doesn't exist yet, create it — same path, empty to start.

## Manual setup

1. `git clone https://github.com/kaiizer777/SKILLS.git ~/Desktop/SKILLS`
2. If `~/Desktop` doesn't exist on this machine (headless Linux, bare SSH session, some container setups), clone into a different parent such as `~/src` instead: `git clone https://github.com/kaiizer777/SKILLS.git ~/src/SKILLS`. Write down the path you actually used — step 3 below needs it.
3. All skill paths in `system-prompt.md` and role skill files resolve directly to `C:/Users/bari2/Desktop/SKILLS`. Ensure your harness configuration points to `C:/Users/bari2/Desktop/SKILLS`.
4. Append the resulting text to your harness's global instruction file from the table above. If equivalent content is already there, don't duplicate it — merge.

Nothing else is required. No install script, no lockfile, no build.

## Copy-paste install prompt

Paste this entire block into your AI coding agent's chat. It's written *to the agent*, not to you.

```text
Set up the AGENTIC-SKILLS repo for me. Do all of the following, in order, and report as you go.

**Path resolution:** All skill paths resolve directly to C:/Users/bari2/Desktop/SKILLS.

1. DETECT THE HARNESS.
   Work out which AI coding tool you are running inside, and find that harness's global instruction file. Known mappings:
     - Gemini CLI   -> ~/.gemini/GEMINI.md
     - opencode     -> ~/.config/opencode/AGENTS.md
     - Claude Code  -> ~/.claude/CLAUDE.md
     - Cursor       -> ~/.cursor/rules/agentic-skills.mdc
     - other        -> the equivalent per-user instruction/memory file for that tool
   If you cannot determine the harness with confidence, STOP and ask me which one it is. Do not guess and do not write to more than one candidate path.

2. CLONE THE REPO.
   Run: git clone https://github.com/kaiizer777/SKILLS.git ~/Desktop/SKILLS
   The folder name SKILLS matches the repo name on purpose — once cloned, the fully expanded absolute path to that folder is C:/Users/bari2/Desktop/SKILLS. Check whether ~/Desktop exists before running the command; if it does not (headless Linux, bare SSH session, some container setups), STOP and ask me for a destination instead. Do not create a ~/Desktop folder and do not silently substitute another path.
   If ~/Desktop/SKILLS already exists and already is this repo, skip cloning and reuse it. If it exists and is something else, ask me for a different name.
   Do not run any other git command that changes state — no checkout, branch, commit, add, stash, or push. Clone only.

3. READ system-prompt.md FROM THE CLONE.
   Read the whole file at <clone path>/system-prompt.md. Confirm all skill paths resolve directly to C:/Users/bari2/Desktop/SKILLS. All skills live at C:/Users/bari2/Desktop/SKILLS, including in role skill files (agent-1to1/skills.md and agent-orchestrator/skills.md). Never quote an unresolved path back to me.

4. MERGE IT INTO THE GLOBAL INSTRUCTION FILE.
   - Read the target file first, if it exists.
   - Check whether an equivalent block of the system prompt is ALREADY there (match on distinctive content, not just the first line). If it is, do not append a second copy — leave it as is and tell me it was already installed.
   - Otherwise append the contents of system-prompt.md exactly as read in step 3 — that is, with paths pointing directly to C:/Users/bari2/Desktop/SKILLS. Verbatim means word-for-word on the resolved text.
   - Preserve everything already in that file. Never delete, reorder, or rewrite existing content.
   - Never overwrite the whole file with anything that isn't already there.
   - Create the file, and any missing parent directory, if it does not exist.

5. STOP AND ASK BEFORE ANY DESTRUCTIVE STEP.
   Before deleting, truncating, or replacing any existing file or content, ask me first and wait for a yes. If you are unsure whether a step is destructive, treat it as destructive.

6. REPORT BACK, briefly:
   - the clone path you used
   - the instruction file you appended to (or that was already current)
   - the exact next step: start a NEW session so the prompt is live — instruction files load at session start. (On Gemini CLI you can instead run /memory refresh in the current session.)

Note: the open-* orchestrator skill folders (open-issue-finding, open-issue-triage, open-pre-fix, open-post-fix, open-pre-pr) are tracked directly in the repository and reserved exclusively for Agent-A.
```

## Verify it worked

Start a new session first — instruction files load at session start, so a change mid-session may not be live (on Gemini CLI, `/memory refresh` reloads in place). Then, in the new session:

1. **Ask who it is:** *"What is your role right now, and why?"* — it should name a role (orchestrator or 1:1) and justify it from the tools it actually has.
2. **Ask about skill routing:** *"I'm about to add a new React form to an existing app — which skill file do you read first, and why that one?"* — it should name an absolute path under `C:/Users/bari2/Desktop/SKILLS`, and explain new-feature vs. existing-code routing rather than picking arbitrarily. A tilde form like ~/Desktop/SKILLS does not expand in a file path. An agent may also print one as shorthand when the installed paths are already expanded, so do not judge the install from its reply alone — open the global instruction file and inspect the skill-path entries themselves. Replace only the entries that actually contain a tilde path with the fully expanded path, then start a new session.
3. **Ask about boundaries:** *"Name one thing you must not do without asking me."* — it should reference a concrete rule (no `git stash`, never commit to `main`).
4. **Ask for the file map:** *"What's in this repo?"* — it should list the five domain skills from the inventory table (frontend, backend, ui-ux, issue-fix, review) plus the two role skills. **If the agent resolved itself as the orchestrator role, skip this check** — listing the repo tree is forbidden to it, and a refusal here is correct behavior, not a failed install.

If a check comes back vague, re-read your instruction file: the block probably didn't land, or landed outside the file the harness actually loads.

If the agent quotes a SKILL PATH that does not resolve to `C:/Users/bari2/Desktop/SKILLS`, inspect the global instruction file and ensure all paths point to `C:/Users/bari2/Desktop/SKILLS`, then start a new session.

## Uninstall

Delete the appended block from your harness's global instruction file, keeping everything else. The clone is standalone — `rm -rf ~/Desktop/SKILLS` (or whatever path you ended up using) removes it, and nothing outside that directory needs to be reverted. Only delete that folder if it is the clone; if you were asked to install into a different location, use that one.
