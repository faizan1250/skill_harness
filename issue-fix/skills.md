---
name: fix-existing-code-issues
description: Use before making ANY change to fix a bug, resolve an issue, address unexpected behavior, or investigate a regression in an existing/production codebase — as opposed to building something new. Language- and stack-agnostic (backend, frontend, data pipeline, concurrent/distributed systems). Trigger on any report of something broken, wrong, crashing, regressed, intermittent, or "not working as expected," even for changes that look small or obvious. Enforces full root-cause clarity, blast-radius analysis, and real verification before a single line of fix code is written.
---

# Fix Existing Codebase Issues

## 1. Identity & Trigger
This is production code for demanding clients. A wrong or shallow fix costs more than the original bug — in reopened incidents, eroded trust, and compounding debt. Speed is never the priority; correctness and durability are. Activates on any bug, defect, crash, regression, or unexpected-behavior report against existing code.

## 2. The Core Law: Full Clarity Before Any Change
Reach full clarity on WHAT is happening, WHY it is happening, and WHERE the fault actually originates before writing one line of fix code.

**Iron law: no fix without a root cause established first.** Investigation and modification are separate phases and never overlap — no exploratory patches, no "let me try this and see." If you catch yourself editing code before you can state the root cause in one sentence backed by evidence, stop, back out the edit, and return to investigation.

## 3. Phase 1 — Reproduce & Capture Ground Truth
- Capture the exact observed behavior yourself: full error text, stack trace, logs, or a precise statement of actual vs. expected output. Never theorize from a paraphrased bug report.
- Reproduce deterministically. If it doesn't reproduce on the first try, don't call it "flaky" yet — first raise the reproduction rate: pin time, seed randomness, isolate the filesystem, freeze network calls, and re-run the trigger repeatedly (10–100x). A bug that reproduces half the time is debuggable; one that reproduces on 1 in 100 runs usually needs a better trigger, not a guess.
- If genuinely intermittent, identify the triggering condition (load, ordering, specific input shape, concurrent access) rather than giving up on a repro.
- Record full environment/state context: data present, action sequence, configuration, versions.
- Exit condition: you can trigger the failure on demand, or you have a quantified reproduction rate and a working theory of the trigger.

## 4. Phase 2 — Root Cause Investigation (not symptom-matching)
- Trace backward through the actual call/data path from where the failure surfaced to where the bad state, bad value, or wrong branch first originated. The place an error is *observed* (crash site) and the place it was *caused* (fault origin) are frequently different — treat them as separate until proven identical.
- Form multiple candidate hypotheses before committing to one. Rule each out with evidence — logs, targeted instrumentation, an isolated test — never with assumption.
- Read the surrounding code and its contracts, not just the failing line, before concluding anything.
- For unfamiliar or legacy code: map the relevant call graph and data flow — who calls this, what do they assume, what shape of data flows in and out — before forming a theory.
- Narrow fast with divide-and-conquer: binary-search the code path (is the bad value present at the midpoint of the flow, or not yet?), or bisect the change history when the bug is a regression against a previously-good state.
- Exit condition: you can state "X happens because Y does Z under condition W," backed by concrete evidence — not a guess. Do not proceed to Phase 3 without this sentence.

## 5. Phase 3 — Blast Radius Analysis (mandatory before any edit)
- Enumerate every direct caller and transitive consumer of the code you're about to change, not just the function containing the bug.
- Enumerate shared state, side effects, and contracts other parts of the system rely on — including any that unknowingly depend on the *buggy* behavior being exactly as it is today. Removing a bug can itself be a breaking change to such a caller.
- Ask explicitly: what else reads or writes this data, calls this function, subscribes to this event, or assumes this ordering/timing?
- Distinguish a fix localized to one function from one that ripples across modules or services, and plan for the ripple before editing — don't discover it after something downstream breaks.
- Include disabled, dormant, or "dead" code paths in your inventory. Don't assume a path is unreachable from its name or a comment — an unrelated change (a repurposed flag, an altered branch condition) can silently reactivate it later. Production incidents have come from exactly this: leftover legacy code, believed inert, reactivated by an otherwise unrelated deploy.
- If the root cause is architectural rather than local, say so explicitly now — before you start editing, not mid-edit.
- Exit condition: a concrete, file/function-level list of everything touched by or dependent on the change, with each item's current actual behavior noted.

## 6. Phase 4 — Choose the Correct Fix, Not the Convenient One
Reject on sight:
- Fixes that only suppress the symptom at the crash site (e.g., a null/bounds check wrapped around a value that should never have been null/out-of-bounds).
- Fixes that special-case the one failing input instead of correcting the general logic that produced it.
- Fixes that patch where the error surfaced without correcting where it originated.
- Exception handling or default values added purely to stop an error from surfacing, without understanding why the invalid state arose — this often just relocates the bug, since downstream code that relied on that error propagating now fails silently instead.

Then:
- Among genuinely correct options, choose for correctness and long-term maintainability, not diff size or speed.
- If the true fix is larger than a minimal patch — e.g., it requires correcting a flawed abstraction the bug lives inside — do that fix, and tell the user explicitly that you're going beyond a minimal patch and why, rather than silently under-scoping.
- Close off the *class* of bug where feasible, not just this one instance.
- Never widen scope beyond what root cause + blast radius actually justify — every touched file must trace back to one of those two analyses. "More correct" is not license for an unrelated rewrite.
- Treat these edit types as regression-prone by nature — they fix one case while risking another — and double-check them specifically: changed loop bounds/indices/boundary conditions; changed return or error contracts on a shared function; added or removed synchronization/locking (this changes execution interleaving and can trade one race condition for another); added exception handling that swallows something a caller expected to observe.

## 7. Phase 5 — Verify, Don't Assume
- Re-run the original Phase 1 reproduction and confirm correct behavior now.
- Re-check every item on the Phase 3 blast-radius list, not just the original failure site.
- Reason through or run adjacent tests and code paths sharing the touched logic, watching specifically for the regression types listed in Phase 4.
- After 2 failed fix attempts on the same bug, stop iterating on variations — return to Phase 2 and re-diagnose from scratch. A third tweak on a failed theory is guessing, not fixing.
- A vanished symptom is not proof of a fixed root cause without blast-radius and adjacent-path verification — it's evidence you stopped looking. Never declare a fix complete on "the code looks correct" alone, and never report a check as passed without having actually run it.
- If full verification isn't possible in your environment (no test runner, can't execute the app, no access to the failing environment), state exactly what was verified and what remains unverified. Do not imply confidence you don't have.

## 8. Anti-Patterns to Actively Refuse
- Patching the first line in a stack trace without checking whether it's the fault origin or just where the failure became visible.
- Adding a try/catch, null check, or default purely to stop an error surfacing, without understanding why the bad state occurred.
- Making an edit, observing the immediate symptom vanish, and stopping — without blast-radius or regression verification.
- Narrowing a fix to "make the failing test pass" instead of fixing the behavior the test exists to represent.
- Treating "I can't easily reproduce it" as license to guess instead of first raising the reproduction rate (Phase 1).
- Maintaining two or more parallel implementations of the same logic because patching a copy was faster than fixing the shared source.
- Reporting a fix, a test run, or a check as complete or passing without having actually executed it — self-assessment is not verification.

## 9. When the Bug Reveals a Deeper Problem
If investigation shows the bug is a symptom of a broader architectural or design issue — not correctly fixable with a local change — say so explicitly. Lay out the tradeoff: local patch now, with its specific limitations and what it leaves unfixed, versus the proper structural fix, with its larger scope and effort. Let the user decide. Do not silently pick the shallow option to dodge that conversation — and don't silently commit to the large rewrite either.

## 10. Final Report Format
Every fix is reported with:
1. **Root cause** — one sentence, plus the concrete evidence supporting it.
2. **Blast radius** — what was checked and touched, and why each item was in scope.
3. **The fix** — what changed, and why this option won over the alternatives considered in Phase 4.
4. **Verification** — what was actually confirmed and how, and what remains unverified, if anything.

## Applying This Across Stacks
"Call graph," "blast radius," "reproduction," and "regression" map onto whatever tooling the language/ecosystem at hand provides — a debugger and test runner in one stack, a REPL and property-based fuzzer in another. Use that stack's own equivalent to execute each phase; the phases and stop-conditions themselves don't change.