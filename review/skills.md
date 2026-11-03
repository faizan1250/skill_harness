---
name: review-implemented-changes
description: Activates immediately after a feature or any non-trivial change has just been implemented — by this agent, another agent, or a human — and before it is considered done, committed, or shipped. Use this whenever you or someone else says a change is "finished," "ready," or "done," before running final tests, before committing, before opening a PR, or before telling the user a task is complete. Language- and framework-agnostic staff-engineer-grade review discipline: treats the just-written diff as a stranger's risky pull request headed to production for a major client, not as work to rubber-stamp. Do not skip this because the change "looks right" — that feeling is exactly what this skill exists to distrust.
---

# Review Implemented Changes — Staff Audit Protocol

## 1. Audit Stance & Pacing: Zero Time Constraints

- **Calm, Steady, and Comprehensive Audit**: Reviewing code is an adversarial deep-dive audit, not a checklist speedrun. Rushing, skimming, or rubber-stamping defeats the entire purpose of this skill.
- **Time Is Never a Constraint**: When this skill is active, you are explicitly instructed to take all the time you need. Never optimize for turn speed or conciseness at the expense of depth. Exhaustively inspect every line, trace full call-chains, verify state transitions, and analyze potential edge-case failures.
- **Adversarial Stance**: Treat the diff as if it were submitted by a stranger who may have misunderstood the business domain, hallucinated APIs, bypassed security gates, or weakened test suites to achieve a green build. The diff is guilty until proven innocent.
- **Familiarity Breeds Blindness**: If you authored the change or watched it being built, actively discard your memory of intent. Inspect only what is written on disk. "It runs and looks plausible" is not a passing bar.

---

## 2. The Core Law: Verify Against Intent, Never Against Output

1. **Re-Read the Original Requirement First**: Before reading the diff, read the original issue, user prompt, design doc, or specification from start to finish. If you cannot articulate the original acceptance criteria and constraints without looking at the code, you are not ready to review.
2. **Directionality of Verification**: Walk from Requirement → Diff. Never walk from Diff → Requirement (rationalizing why whatever code was written makes sense in retrospect).
3. **Plausibility Trap**: AI-generated code excels at looking syntactically elegant, well-commented, and structurally idiomatic while implementing completely incorrect logic. Demand empirical proof of correctness against the original requirement.

---

## 3. Requirement Fidelity & Scope Boundaries

- **Exhaustive Constraint Matching**: Verify every stated requirement, constraint, edge case, and acceptance criterion. Did the author satisfy all criteria, or only the easiest/most obvious happy path?
- **Silent Scope Reduction**: Flag any requirement that was silently omitted, narrowed, simplified, or deferred without explicit user authorization.
- **Unrequested Additions (Scope Creep)**: Flag speculative generalizations, unneeded helper utilities, "while I was here" refactors, or unrequested features. Code not requested is liability added.
- **Implicit Assumptions**: Any assumption made by the author to resolve ambiguity must be documented and validated against the task specification.

---

## 4. Blast Radius, Callers & Contract Invariants

- **Full File Audit**: Every touched file must have a direct causal justification tied to the task. Flag drive-by formatting or unrelated edits.
- **Caller Tree Verification**: Enumerate every caller of modified functions, endpoints, interfaces, data models, or types. Confirm that caller expectations, parameter order, return values, nullability, and exception semantics remain intact.
- **Breaking Contract Changes**: A signature or type change is only as safe as its least defensive consumer. Check both internal callers and external consumers/APIs.
- **Shared State & Side Effects**: Identify any altered globals, singletons, caches, database rows, session stores, or lifecycle hooks. Ensure concurrent interactions cannot corrupt shared state.
- **Diff Size Warning**: If the diff touches excessive lines or disparate concerns, state explicitly that it should be audited in distinct logical batches.

---

## 5. Deep Correctness, Invariants & Edge Cases

- **Trace Execution Paths**: Trace the primary happy path end-to-end. Then explicitly trace at least 3 adverse scenarios:
  1. Empty, null, undefined, or zero-length inputs.
  2. Boundary conditions, off-by-one indices, numeric overflow/underflow, and extreme input sizes.
  3. Concurrent execution, partial failures, network timeouts, and aborted operations.
- **Branch Reachability**: Verify every conditional branch (`if`, `switch`, ternary, match) is logically reachable, intentional, and handles the inverse condition correctly.
- **State Invariants**: Ensure system state remains valid before, during, and after operations. If an operation fails mid-way, verify that partial mutations are either rolled back or safely isolated.

---

## 6. Error Handling & Failure Recovery

- **Zero Silent Failures**: Flag any empty `catch` block, `except: pass`, swallowed promises, or blind error discards (`catch-and-ignore`).
- **Dangerous Fallbacks**: Flag fallback returns (e.g., returning `false`, `null`, or empty array on any failure) on security-critical or data-mutating paths. Swallowed authentication/authorization errors can fail open catastrophically.
- **Observability & Error Context**: Ensure caught errors are logged with meaningful diagnostics (stack trace, input metadata, failure stage) and rethrown or returned as structured, typed domain errors.
- **Caller Actionability**: Ensure downstream callers can unambiguously distinguish between different failure modes (e.g., Not Found vs. Unauthorized vs. Upstream Timeout).

---

## 7. Security, Authorization & Input Hygiene

- **Defense-in-Depth Auth**: Any new endpoint, action, mutation, or CLI command must enforce both **Authentication** (identity) and **Object-Level Authorization** (ownership/tenant permissions). Route-level gates alone are insufficient. Verify that tenant/user `A` cannot access tenant/user `B`'s resources via IDOR (Insecure Direct Object Reference).
- **Server-Side Trust Boundary**: Never trust client-provided IDs, roles, pricing, or sanitized state. Validate all inputs at the backend boundary using schemas (e.g., Zod, Pydantic, Joi) or strict validators.
- **Injection Vectors**: Inspect every dynamic query, shell invocation, HTML template interpolation, regular expression, and file path manipulation for SQLi, Command Injection, XSS, ReDoS, or Path Traversal.
- **Credential Hygiene**: Ensure no secrets, tokens, API keys, or sensitive customer PII are hardcoded, logged, committed to source control, or leaked in error responses.

---

## 8. Dependency & Import Integrity

- **Hallucination Check**: Confirm every imported package, module, and external API actually exists in the official package registry (npm, PyPI, Crates.io, Go modules) and behaves as assumed. AI-hallucinated packages are a real supply-chain attack vector.
- **Version Pinning & Vulnerabilities**: Ensure dependencies use pinned or strictly bounded versions. Check that new packages do not introduce known CVEs or unmaintained bloat.
- **Reinventing the Wheel**: Flag external dependencies imported for tasks already solvable by native language features or libraries already present in the project.

---

## 9. Test Suite Integrity & Regression Proof

- **Assert Observable Behavior**: Tests must assert external contracts and side effects, not internal implementation mechanics that can pass even when output is wrong.
- **No Weakened Assertions**: Strictly audit test diffs. Treat modified assertions, widened numeric tolerances, deleted assertions, or newly added skip annotations (`@pytest.mark.skip`, `it.skip`, `xit`, `t.Skip()`, `#[ignore]`, or any language-equivalent) as critical defects. Agents frequently weaken test assertions to force a passing test run.
- **Negative & Failure Testing**: Tests must explicitly cover error paths, bad inputs, validation rejections, and timeout scenarios — not just green happy paths.
- **Falsifiability Check**: Mentally invert the logic in the implementation. If the test would still pass with the bug reintroduced, the test is tautological and useless.

---

## 10. Performance, Concurrency & Resource Lifecycles

- **N+1 Queries & Loops**: Identify database queries, network requests, or disk I/O executed inside iterative loops over dynamic collections.
- **Unbounded Operations**: Flag database queries without limits/pagination, unbounded memory buffering, or unbounded concurrency spawns (`Promise.all` over arbitrary input arrays).
- **Resource Leaks**: Ensure every connection, file descriptor, lock, timer, stream, or subscription is reliably closed/released in `finally` blocks, defer statements, or context managers across both success and failure exits.
- **Race Conditions**: Identify check-then-act patterns, non-atomic database operations, or memory access lacking synchronizing primitives.

---

## 11. Architecture, Consistency & Maintainability

- **Repo Idioms & Patterns**: Confirm new code blends seamlessly with existing project architecture, directory conventions, typing standards, and error-handling idioms rather than introducing an alien style.
- **DRY vs. Decoupling**: Check whether new helpers duplicate existing project utilities. Conversely, flag tight coupling introduced between previously decoupled modules.
- **Dead Code & Leftover Artifacts**: Ensure no commented-out code, temporary debug logging, scratch files, or unused imports remain.

---

## 12. Response & Audit Report Format

Produce a structured audit report covering every section below:

### 12.1 Severity Classification (5 Tiers)

Every finding must be strictly categorized into one of the following 5 tiers:

| Tier | Severity | Definition & Impact | Action Required |
| :--- | :--- | :--- | :--- |
| **1** | **`CRITICAL`** | Vulnerabilities (RCE, SQLi, Auth bypass, IDOR), data loss/corruption, silent test weakening, system crashes, or fundamental logic inversion breaking core requirements. | **Must fix immediately.** Blocks commit, merge, or deployment. |
| **2** | **`HIGH`** | Major functional bugs, unhandled real-world edge cases causing failure, broken API contracts, missing server-side validation, or severe performance degradation under typical scale. | **Must fix.** Must be resolved before shipping to production. |
| **3** | **`MEDIUM`** | Uncaught exceptions in non-primary flows, swallowed errors without context, missing timeout/resource cleanup, race conditions in plausible edge scenarios, or incomplete test coverage for realistic paths. | **Should fix.** Needs remediation or documented rationale. |
| **4** | **`LOW`** | Sub-optimal performance/query efficiency, minor logic or typing inconsistencies, inadequate diagnostic logging, or deviations from codebase naming conventions that increase maintenance friction. | **Nice to have.** Fix during normal workflow or file as tech debt. |
| **5** | **`IMPROVEMENT`** | Architectural cleanup, abstraction opportunities, code deduplication, idiomatic style enhancements, or clarity improvements with zero runtime risk. | **Optional.** Polish for long-term codebase health. |

---

### 12.2 Output Template

Your audit report should use the following structure as a guide. Adapt depth and verbosity to the scope of the change, but never omit sections or collapse findings — every section must be explicitly addressed:

```markdown
# 🛡️ Comprehensive Code Audit Report

## 1. Executive Summary & Verdict
- **Verdict**: [🚨 REJECT / ⚠️ CONDITIONAL PASS / ✅ PASS]
- **High-Level Summary**: [2-4 sentences explaining the overall quality, risks, and health of the changes]
- **Findings Tally**:
  - `CRITICAL`: X
  - `HIGH`: Y
  - `MEDIUM`: Z
  - `LOW`: W
  - `IMPROVEMENT`: V

---

## 2. Detailed Findings

### [SEVERITY_TAG] Finding Title (e.g., [CRITICAL] Unchecked IDOR in Billing Webhook)
- **File & Line**: `path/to/file.ext:L123-L135`
- **Category**: [Security | Correctness | Error Handling | Test Integrity | Performance | Architecture]
- **The Problem**: [Clear, unambiguous explanation of the flaw or vulnerability]
- **Failure Scenario / Exploit**: [Step-by-step trigger condition showing how this fails or is exploited in practice]
- **Remediation**:

        - problematic code
        + corrected code

*(Repeat for every finding across all severity tiers, ordered highest to lowest severity)*

---

## 3. Section-by-Section Verification Matrix

*(Sections 1–2 are stance and process — not domain-verifiable. The matrix covers audit domains only.)*

| Section | Audit Domain | Status | Notes |
| :--- | :--- | :--- | :--- |
| 3 | Requirement Fidelity | [PASS / FAIL / WARN] | Brief evaluation |
| 4 | Scope & Blast Radius | [PASS / FAIL / WARN] | Caller verification notes |
| 5 | Deep Correctness & Logic | [PASS / FAIL / WARN] | Edge cases checked |
| 6 | Error Handling & Recovery | [PASS / FAIL / WARN] | Silent failures / catches |
| 7 | Security & Authorization | [PASS / FAIL / WARN] | AuthN/AuthZ & injection checks |
| 8 | Dependency Integrity | [PASS / FAIL / WARN] | Hallucination & CVE checks |
| 9 | Test Suite Integrity | [PASS / FAIL / WARN] | Verification of assertions |
| 10 | Performance & Resources | [PASS / FAIL / WARN] | Leak & N+1 audit |
| 11 | Design & Architecture | [PASS / FAIL / WARN] | Codebase harmony |

---

## 4. Explicit Unverified List
*Document honestly everything that was NOT checked or cannot be verified from the diff alone:*
- [e.g., Did not run load/stress testing under high concurrent volume]
- [e.g., Did not verify database migrations against production dataset volumes]
- [e.g., Third-party webhook signatures were mocked rather than validated against live provider sandbox]
```