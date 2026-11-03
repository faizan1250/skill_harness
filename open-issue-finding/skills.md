---
name: open-issue-finding
description: Stage-gated pipeline for autonomously discovering, verifying, and reporting legitimate bugs and security issues in any open-source repository, with a mandatory security-sensitivity branch that routes real vulnerabilities to private disclosure instead of public issues. Use this whenever an orchestrator or agent is tasked with proactively hunting for defects, security vulnerabilities, or code-quality issues in any codebase — or whenever deciding if a finding is real, non-duplicate, and valuable enough to file publicly, or must instead go to private security disclosure (via GitHub Private Vulnerability Reporting, repo SECURITY.md, or security contacts). Every candidate must pass every stage in order; none may be skipped, shortcut, or filed on partial confidence.
---

# Agent Skill — Open Issue Finding

**Sections:** 0. Purpose & Operating Principles · 1. Orchestrator & Subagent Model · 2. Stage Gate Enforcement Rules · 3. Stage 1: Scope Definition & Repository Baseline · 4. Stage 2: Surface Mapping & Architecture Analysis · 5. Stage 3: Vulnerability & Bug Hypothesis Generation · 6. Stage 4: MRE Reproduction & False-Positive Elimination · 7. Stage 5: Severity, Impact & Precedent Assessment · 8. Stage 6: Responsible Disclosure & Issue Filing · 9. Anti-Patterns · 10. Appendix A: Responsible Disclosure & Advisory Protocols · 11. Appendix B: Candidate Funnel Tracking

---

## 0. Purpose & Operating Principles

### Purpose

Govern how an orchestrator or agent autonomously discovers real, high-impact defects and security vulnerabilities in any open-source repository and places them in front of maintainers through the proper channel — public issue or private security advisory — without degrading into low-signal, unverified, AI-generated noise.

Maintainers across open-source ecosystems (e.g., curl, Ghostty, tldraw, Godot) have actively throttled, filtered, or banned automated contributions because unverified volume, repetitive hallucinated patterns, and theoretical nitpicks exhaust maintainer goodwill and destroy trust. Every issue filed by this pipeline spends reputation. Treat every report as an investment of project credibility that must deliver undeniable, reproducible value.

### Non-Negotiable Operating Principles

These principles bind all stages and override local heuristics:

1. **Reproduction Before Filing:** No issue is ever filed without an actually-executed, actually-observed reproduction. A pattern-match, grep hit, compiler warning, or speculative "this looks unsafe" is a *candidate hypothesis*, never a reportable issue.
2. **Strict Grounding:** Every claim must cite exact file paths and line numbers verified fresh in the current session against the latest default branch. Never advance a finding based on model recall, inference, or assumptions about what code "probably" does.
3. **One Issue = One Problem:** Never bundle multiple findings into a single issue or report. Each defect or vulnerability gets an independent pipeline path.
4. **MRE Re-Verification Post-Trimming:** Every candidate must produce a Minimal Reproducible Example (MRE) that is independently re-executed and re-verified *after* code trimming. Trimming extraneous code can silently resolve, mask, or alter the defect.
5. **Zero Style/Taste Nitpicks:** Formatting, naming conventions, stylistic preferences, architectural taste, or vague "could be improved" suggestions are strictly discarded, regardless of model confidence.
6. **Mandatory Security Isolation:** Security-sensitive findings must **never** be filed as public GitHub issues, public comments, or public pull requests. No exceptions, no downgrades, no sanitization for public posting.
7. **Execution Over Self-Grading:** A model re-evaluating its own hypothesis and declaring "this is correct" is not verification. Verification requires concrete, logged execution output against live code.
8. **Senior Researcher Standard:** Hold every finding to the standard of a senior security researcher or staff software engineer who personally reproduced and validated the behavior.
9. **Quality Over Throughput:** Volume is a pure liability; signal is the only metric. A single verified, high-impact finding is a complete success. Discarding twenty unproven candidates is standard procedure.
10. **Repository Governance as First-Class Gate:** Always read `CONTRIBUTING.md`, `SECURITY.md`, issue templates, and steering files before beginning any hunt. Follow all project-specific contribution guidelines and constraints.

---

## 1. Orchestrator & Subagent Model

**Subagent allocation per stage is dynamic and must never be hardcoded.** The orchestrator determines the number, specialty, and coordination model of subagents based on codebase scale, candidate volume, and technical domain.

### Dynamic Sizing Heuristics

- **Stages 1 & 2 (Scope & Surface Mapping):** Parallelize by architectural domain, subsystem, or package boundary (e.g., core engine, network/transport layer, storage/persistence layer, CLI/API handlers).
- **Stage 3 (Hypothesis Generation):** Parallelize by discovery technique or attack vector (e.g., pattern-grep sweep, state/concurrency analysis, input-boundary fuzzing, dependency audit).
- **Stages 4 & 5 (MRE Reproduction & Impact Assessment):** Parallelize strictly **per candidate** (one subagent owns one candidate end-to-end). Keeping reproductions in isolated subagent contexts prevents bias, context contamination, and cross-candidate confusion.
- **Stage 6 (Disclosure / Filing):** Execute per verified candidate. Separate subagents for drafting public issues versus formatting private security advisories.

The orchestrator enforces the stage exit gates. No candidate advances until its exit criteria are fully satisfied.

---

## 2. Pipeline Flow & Stage Gate Enforcement Rules

```
Stage 1: Scope Definition
   ↓
Stage 2: Surface Mapping
   ↓
Stage 3: Vulnerability / Bug Hypothesis Generation
   ↓
Stage 4: MRE Reproduction & False-Positive Elimination
   ↓
Stage 5: Severity, Impact & Precedent Assessment
   ↓
Stage 6: Responsible Disclosure & Reporting
   ├─ [Security-Sensitive] ──→ Private Disclosure (GitHub PVR / SECURITY.md) ──→ STOP
   └─ [Non-Security Defect] ─→ Public GitHub Issue Filing (via Repo Template)
```

### Stage Gate Enforcement Rules

- **Strict Sequential Gating:** No stage may be skipped. No candidate advances to Stage $N+1$ until Stage $N$ is 100% satisfied for that candidate.
- **Binary Pass/Fail:** "Mostly green" does not exist. If any checklist item is unmet or ambiguous, the candidate is discarded (or routed privately at Stage 6).
- **Per-Candidate Progression:** Candidates move independently. In a batch of 15 candidates, each progresses at its own speed; never advance a batch because the majority cleared a gate.
- **Candidate Funneling:** Discarding candidates is the standard and expected outcome at Stages 3, 4, and 5. The pipeline is an aggressive filter designed to eliminate low-signal noise.

---

## 3. STAGE 1 — Scope Definition & Repository Baseline

**Objective:** Ingest repository governance, constraints, contribution requirements, and disclosure channels before inspecting a single line of application logic.

### Required Exploration Checklist

Execute against the target repository's current default branch (`main` / `master`):

1. **Governance & Contribution Policies:**
   - Read `CONTRIBUTING.md`, `.github/CONTRIBUTING.md`, or `docs/CONTRIBUTING.md`.
   - Note required issue templates, mandatory labels, reproduction standards, and explicit scope exclusions (e.g., policies forbidding automated bug submissions, benchmark disputes, or minor documentation PRs).
2. **Security & Vulnerability Disclosure Policies:**
   - Read `SECURITY.md`, `.github/SECURITY.md`, or security documentation.
   - Record declared private disclosure channels:
     - GitHub Private Vulnerability Reporting (PVR) enabled on the repo.
     - Dedicated security contact email or advisory portal (e.g., HackerOne, Bugcrowd, security@domain).
     - Security policy version and supported versions eligible for security fixes.
3. **Agent & Steering Guidelines:**
   - Read any `AGENTS.md`, `CLAUDE.md`, `.cursorrules`, or repository developer documentation that defines local coding standards, invariants, or architectural constraints.
4. **Issue Templates & Metadata:**
   - Inspect `.github/ISSUE_TEMPLATE/` or `.github/issue_template.md`. Note mandatory fields, structured forms, and required reproduction platforms.
   - Inspect `.github/labeler.yml` or repo labels to identify correct issue categorization (e.g., `bug`, `triage`, `area/*`).
5. **Automation & CI Workflow Footprint:**
   - Inspect `.github/workflows/*.yml` `on:` triggers for issue events. Identify automated bots (stale bot, auto-assigner, labeler, triage actions) to understand how newly opened issues are handled.
6. **Scope Boundaries & Exclusions:**
   - Explicitly identify out-of-scope files: vendored third-party code, generated code (`dist/`, `build/`, auto-generated clients/protos), sample applications, test fixtures, and deprecated packages.

### Stage 1 Exit Gate — ALL must be true to proceed

- [ ] Target repository is checked out on the latest commit of the default branch.
- [ ] `CONTRIBUTING.md` and repo contribution constraints read and recorded.
- [ ] `SECURITY.md` read; private disclosure channel and contact details explicitly logged.
- [ ] Repository issue templates and required fields documented.
- [ ] Out-of-scope paths (vendored, generated, sample code) identified and recorded in the hunt dossier.

---

## 4. STAGE 2 — Surface Mapping & Architecture Analysis

**Objective:** Systematically decompose the codebase into architectural components, trust boundaries, state models, and external integrations to direct hypothesis generation toward high-leverage areas.

### Surface Mapping Dimensions

1. **Component & Module Decomposition:**
   - Map core runtime engines, public API entry points, protocol parsers, network handlers, storage adapters, CLI interfaces, and shared utility layers.
2. **Trust Boundaries & Data Flow:**
   - Identify where untrusted external data enters:
     - HTTP/WebSocket request bodies, query strings, headers, cookies.
     - CLI flags, environment variables, configuration files.
     - Inter-process communication (IPC), message queues, RPC inputs.
     - File system readers, archive extractors, serialization decoders.
   - Trace untrusted inputs to critical execution sinks:
     - SQL/NoSQL query execution.
     - Operating system process execution (`exec`, `spawn`, `subprocess`).
     - File system path resolution and read/write operations.
     - Memory allocation, buffer manipulation, string slicing.
     - HTML/DOM generation, template rendering, reflection calls.
3. **State, Concurrency & Lifecycle Surfaces:**
   - Identify shared mutable state: global variables, in-memory caches, connection pools, singleton managers.
   - Trace concurrency mechanisms: async/await promise chains, goroutines, threads, locks, mutexes, event emitters, background workers.
4. **Security & Invariant Boundaries:**
   - Locate authentication, authorization, token issuance, credential handling, session validation, cryptographic primitives, and tenant isolation layers.
   - Formulate the critical invariants that must hold true across all execution paths (e.g., "every tenant query must filter by tenant_id", "tokens must expire strictly within TTL", "memory buffers must enforce bounds checks").
5. **Runtime & Compatibility Matrix:**
   - Document target runtimes, supported engine versions (e.g., Node LTS vs current, Python 3.9–3.12, Go 1.21+, browser runtimes), and OS-specific behavior branches.

### Stage 2 Exit Gate — ALL must be true to proceed

- [ ] Architecture and component decomposition documented.
- [ ] Trust boundaries and untrusted-input-to-sink pathways mapped.
- [ ] Shared state, concurrency primitives, and background lifecycles mapped.
- [ ] Core system invariants explicitly defined.
- [ ] High-risk attack surfaces ranked and prioritized for hypothesis generation.

---

## 5. STAGE 3 — Vulnerability & Bug Hypothesis Generation

**Objective:** Cast an expansive, multi-vector discovery net across mapped surfaces. Generate raw, unverified candidate hypotheses with exact file and line citations.

### Universal Discovery Vectors

1. **Pattern-Grep Sweeps for High-Risk Constructs:**
   - **Timing Side-Channels:** String/buffer equality checks (`==`, `===`, `memcmp`, standard equals) comparing secrets, tokens, API keys, password hashes, signatures, or HMACs. Must use constant-time primitives (e.g., `crypto.timingSafeEqual`, `hmac.compare_digest`, `subtle.constantTimeCompare`).
   - **Insecure Randomness / Weak PRNG:** Non-cryptographic random generators (`Math.random()`, `random.random()`, `rand()`) used to create session tokens, secrets, nonces, salt, IVs, or unique identifiers where collision resistance is critical.
   - **Strippable Assertions:** Invariant checks or authorization gates implemented using `assert()`, debug macros, or compiler-optimized statements that can be stripped in production builds.
   - **Injection Flaws:** User-controlled values reaching SQL queries (raw string concatenation), shell executions (`child_process.exec`, `os.system`), template engines, dynamic regex compilers (`new RegExp(...)` without escaping), or dynamic code evaluators (`eval`, `Function`, `vm`).
   - **Path Traversal & Arbitrary File Access:** File path construction concatenating unvalidated user inputs with root directories without canonical path verification (`path.resolve`, `filepath.Clean`, checking path prefix boundary).
   - **Unsafe Deserialization:** Parsing untrusted payloads via unconstrained YAML, pickle, Java serialization, or dynamic object hydration.
   - **Cache Keying & State Isolation:** Caching mechanisms keyed solely on user-provided values without scoping to authenticated session or tenant identity.
   - **Error & Timing Asymmetries:** Divergent response codes, error details, or measurable timing differences between "invalid user" and "invalid password", enabling user enumeration.
2. **Protocol, Auth & State Machine Sweep:**
   - Missing session rotation upon privilege escalation or login.
   - Token lifecycle gaps: missing revocation upon logout, refresh tokens without rotation or reuse detection, lack of expiration validation.
   - Insecure defaults: cookies missing `HttpOnly`, `Secure`, or `SameSite` flags; permissive CORS headers (`Access-Control-Allow-Origin: *` with credentials); unvalidated redirect targets.
   - State transition flaws: actions permissible out-of-order, skipped intermediate validation steps, or replayable requests.
3. **Differential Testing & Specification Conformance:**
   - Where the codebase implements multiple adapters, drivers, or parsers claiming identical contracts, execute identical inputs across them to detect divergent behavior or silent error swallowing.
   - Compare actual behavior against published RFCs, formal specs, or documented library promises.
4. **Trace-the-Boundary Fuzzing:**
   - Manually trace boundary inputs through public APIs and handlers: empty strings, `null`, `undefined`, negative numbers, integer overflow boundaries ($2^{31}-1$, $2^{64}-1$), oversized payloads, malformed Unicode/surrogates, expired timestamps, and malformed JSON/headers.
5. **Concurrency, Race Conditions & Resource Lifecycles:**
   - Time-of-Check to Time-of-Use (TOCTOU) bugs in file operations or database checks.
   - Unsynchronized mutable state accessed across threads/coroutines; unawaited promises causing unhandled rejections or out-of-order execution.
   - Resource leaks: unclosed database handles, leaked sockets, uncollected event listeners, unbounded memory caches growing without eviction policies.
6. **Documentation-to-Code Mismatch:**
   - Public documentation promises specific defaults, error codes, validation checks, or guarantees that the underlying code completely fails to enforce.
7. **Dependency & Supply Chain Analysis:**
   - Audit lockfiles and manifests (`package.json`, `go.mod`, `requirements.txt`, `Cargo.lock`) for known vulnerabilities (CVEs) where the vulnerable method/code path is actually imported and executed in the target repository.

### Candidate Output Format (Mandatory Fields)

Every generated hypothesis must record:
- **File path & line number(s):** verified directly from code read during this session.
- **Discovery vector:** the exact technique that surfaced the candidate.
- **Hypothesis statement:** concise, concrete statement of the failure condition.
- **Initial classification:** Bug (Logic / Concurrency / Resource) or Security Vulnerability.
- **Status:** `RAW_CANDIDATE` (unverified).

### Stage 3 Exit Gate — ALL must be true to proceed

- [ ] All mapped surfaces evaluated across discovery vectors.
- [ ] Every candidate possesses a verified `file:line` citation read in the current session.
- [ ] No candidate has been assumed or extrapolated from memory.
- [ ] Candidates lacking concrete file/line citations are discarded before Stage 4.

---

## 6. STAGE 4 — MRE Reproduction & False-Positive Elimination

**Objective:** Transform theoretical hypotheses into verified, executed reality or ruthlessly eliminate them.

### Required Verification Procedure (Per Candidate)

1. **Construct a Minimal Reproducible Example (MRE):**
   - Isolate the hypothesis into a minimal, standalone reproduction: a runnable script, CLI command, or automated test case within the repository's test framework.
   - Strip all non-essential context, dependencies, and configuration.
   - **Post-Trimming Re-Test Rule:** Execute the MRE *after* trimming. If trimming causes the defect to disappear, the stripped code was load-bearing or mitigating the issue. Re-investigate or discard.
2. **Execute and Observe:**
   - Execute the MRE directly against a fresh checkout of the latest default branch.
   - Record exact, unedited stdout, stderr, process return codes, stack traces, and database/network logs.
   - *Speculative assertion ("this code should fail because...") is an automatic disqualifier. Actual execution is mandatory.*
3. **Verify Determinism & Rate:**
   - Execute the reproduction repeatedly (minimum 3 runs).
   - Classify as 100% deterministic or intermittent. If intermittent, document the observed failure rate (e.g., 4/10 executions under concurrent load). Intermittent findings must document exact concurrency or timing preconditions.
4. **Trace End-to-End Execution Flow:**
   - Trace execution from the untrusted entry point to the failing line in the current codebase. Confirm the flagged code is actually reached under realistic runtime conditions, not orphaned or dead code.
5. **Inspect Call-Chain Mitigations:**
   - Trace upstream callers and downstream helpers. Verify whether an upstream validator, type checker, framework middleware, or normalization function already sanitizes the input or neutralizes the flaw.
6. **Examine Intent, Docs & Test History:**
   - Review surrounding code comments, git history (`git log -S`, `git blame`), open discussions, and existing unit tests.
   - Confirm the observed behavior is not documented, intentional, or an accepted design trade-off. Flagging deliberate architectural decisions as bugs is the primary failure mode of automated audits.
7. **Confirm Against Latest Upstream Default:**
   - Ensure reproduction succeeds against the latest upstream default branch, not a stale checkout, pre-release branch, or old release tag.

### Stage 4 Exit Gate — ALL must be true to proceed

- [ ] Actually-executed MRE exists, with exact console/test output recorded.
- [ ] MRE was re-tested post-trimming and verified to trigger the exact failure.
- [ ] Determinism verified and reproduction rate explicitly documented.
- [ ] Execution path confirmed reachable under realistic runtime conditions.
- [ ] Call-chain verified to have no upstream or downstream mitigations.
- [ ] Verified that behavior is not documented, intentional, or tested design.
- [ ] Verified against the latest upstream default branch.

*Any unchecked box results in immediate candidate elimination. Record as `discarded@stage4 (<reason>)`.*

---

## 7. STAGE 5 — Severity, Impact & Precedent Assessment

**Objective:** Confirm the verified finding is non-duplicate, assess real-world severity, and write a concrete impact justification that clears the maintainer signal bar.

### 1. Duplicate & Precedent Check

Execute a comprehensive search across repository history:

1. **Open and Closed Issues:**
   - Search GitHub issues using broad and narrow keywords, error strings, and affected function/symbol names.
   - **Critical:** Explicitly clear GitHub's default `is:open` filter to search both open and closed issues (`is:issue`). Closed issues frequently hold the maintainer's prior ruling on the matter.
2. **GitHub Discussions & Community Channels:**
   - Search Discussions, forums, and Q&A threads for prior triage, duplicate questions, or maintainer guidance on the behavior.
3. **Changelogs, Releases & Commit History:**
   - Check `CHANGELOG.md`, GitHub Releases, and recent commit logs to verify the issue has not already been addressed in an unreleased commit or latest patch.
4. **Disposition Rules:**
   - **Exact or near-duplicate open issue exists:** Discard candidate from filing pipeline. (If the reproduction adds novel, critical diagnostic evidence, evaluate whether to add a focused comment to the existing issue).
   - **Already fixed in latest default branch:** Discard candidate.
   - **Closed as wontfix / by-design / intended behavior:** Discard candidate immediately. The maintainer has already ruled.
   - **No prior precedent found:** Candidate proceeds.

### 2. Concrete Impact Justification (The Maintainer Value Bar)

Write an explicit, rigorous justification explaining why this finding justifies maintainer attention:

- **Specific Defect Category:** Concrete technical classification (e.g., Unhandled Exception / Crash, Data Corruption, Concurrency Race Condition, Resource Leak / OOM, Protocol / RFC Violation, ReDoS, Authentication Bypass, Injection).
- **Real-World Blast Radius:** Who is affected, under what conditions, and with what consequences? State the actual observed failure from Stage 4 (e.g., "Under concurrent HTTP keep-alive requests, file descriptors are leaked at a rate of 1 per connection drop, leading to process exhaustion after ~1024 dropped connections").
- **Exclusion of Subjective Preferences:** Explicitly confirm the finding is not a stylistic preference, naming suggestion, documentation typo, minor refactor, or generic advice ("needs more tests").
- **Severity Rating:** Assign objective severity based on impact and exploitability:
  - **Critical:** Remote code execution, unauthenticated auth bypass, silent data corruption, persistent system-wide denial of service.
  - **High:** Authenticated privilege escalation, sensitive data/credential disclosure, memory corruption, crash reachable via standard input.
  - **Medium:** Logic errors under edge conditions, resource leaks under high load, spec non-compliance with interoperability impact.
  - **Low:** Edge-case errors with minimal operational impact and simple workarounds.

### Stage 5 Exit Gate — ALL must be true to proceed

- [ ] Open and closed issues, discussions, and commit history searched and documented.
- [ ] Confirmed novel (no existing duplicate or prior "wontfix" ruling).
- [ ] Written impact justification completed, grounded in Stage 4 execution data.
- [ ] Confirmed non-stylistic and non-cosmetic.
- [ ] Objective severity level classified with justification.

---

## 8. STAGE 6 — Responsible Disclosure & Issue Filing

**Objective:** Direct the verified, high-impact finding to the appropriate communication channel. Security vulnerabilities must **never** be disclosed publicly.

### Security-Sensitivity Branch (CRITICAL & NON-NEGOTIABLE)

Every surviving candidate is evaluated against the security classification criteria below. Classification is explicit and mandatory.

#### Security Classification Criteria

A finding is classified as **Security-Sensitive** if it involves ANY of the following:
- Authentication or authorization bypass.
- Privilege escalation (horizontal or vertical).
- Remote code execution (RCE) or arbitrary command execution.
- SQL, NoSQL, template, LDAP, or shell injection.
- Secret, credential, token, API key, or private key leakage.
- Path traversal, arbitrary file read, or arbitrary file write.
- Cryptographic weakness, broken encryption, or weak PRNG in security contexts.
- Timing side-channels with measurable exploitability.
- Cross-Site Scripting (XSS), Cross-Site Request Forgery (CSRF), or Server-Side Request Forgery (SSRF).
- Memory corruption, buffer overflow, or use-after-free.
- Denial of Service (DoS / ReDoS) with low cost to the attacker targeting shared services.
- Sensitive personal data (PII) exposure.

---

### Path A: Candidate is Security-Sensitive → Private Disclosure

1. **Absolute Public Embargo:**
   - **Do not create a public GitHub issue, public PR, public discussion, or public comment.**
   - Do not commit reproduction code to public branches, forks, or gists.
2. **Select Private Disclosure Channel (in priority order):**
   - **Channel 1 — GitHub Private Vulnerability Reporting (PVR):** If enabled on the repository (Security tab → "Report a vulnerability"), use GitHub's native advisory submission mechanism.
   - **Channel 2 — Repository Security Policy:** Follow the reporting mechanism declared in `SECURITY.md` (e.g., email to security contact, PGP key, Bugcrowd/HackerOne program).
   - **Channel 3 — Maintainer Security Contact:** Contact the designated maintainer(s) via secure email listed in repository metadata or package manifests.
3. **Format Private Advisory Report:**
   - Adhere strictly to the format defined in [Appendix A](#appendix-a--responsible-disclosure--advisory-protocols).
   - Include affected versions, CVSS vector/severity, summary, step-by-step reproduction, verified MRE, impact analysis, and suggested remediation (if evident).
4. **Close Pipeline for Candidate:**
   - Log candidate status as `routed-to-private-disclosure`.
   - **Halt all further actions for this candidate.** The candidate does not enter public issue filing, does not get "sanitized" for a public issue, and is never discussed publicly while awaiting disclosure.
5. **Ambiguity Rule:**
   - If there is any doubt whether a finding has security implications, **always default to Private Disclosure**. A private disclosure that maintainers reclassify as a non-security bug costs nothing; a public issue disclosing an unpatched zero-day causes immediate harm to users.

---

### Path B: Candidate is Non-Security Defect → Public Issue Filing

1. **Follow Repository Issue Templates:**
   - Use the template identified in Stage 1 (`.github/ISSUE_TEMPLATE/`). Populate all mandatory sections without altering template syntax or headers.
2. **Draft Issue Report Structure:**
   - **Title:** Crisp, descriptive, action-and-consequence format (e.g., `Buffer pool leak in HTTP keep-alive handler under high concurrency` — avoid generic titles like `Bug in parser`). **NEVER use Conventional Commit prefixes (e.g., `fix:`, `fix(scope):`, `feat:`, `chore:`) in issue titles. BAD: `fix(email-verification): change-email link mints session...` GOOD: `Change-email verification link mints new session when autoSignInAfterVerification is disabled`. Validation before filing: if title matches `^(fix|feat|chore|docs|refactor|test|style|perf|build|ci)(\(.*\))?!?:` - REJECT and rewrite without prefix.** Conventional Commits are reserved strictly for git commits and PR titles. GitHub issue titles must always be clean, plain-text descriptions of the observable bug or behavior.
   - **Environment:** Tested commit hash, runtime version, operating system, and build configuration.
   - **Expected Behavior:** Clear, concise statement of correct behavior.
   - **Actual Behavior:** Explicit statement of the defect observed.
   - **Steps to Reproduce:** Sequential, unambiguous steps to trigger the issue.
   - **Minimal Reproducible Example (MRE):** Markdown-formatted code block containing the verified, self-contained MRE from Stage 4.
   - **Execution Output / Stack Trace:** Actual logs, stderr, or failure trace recorded during Stage 4 execution.
   - **Impact Assessment:** Grounded in Stage 5 analysis.
   - **Suggested Fix (Optional):** Presented modestly and without presumption only if obvious from Stage 4 root-cause analysis.
3. **Labeling & Routing:**
   - Apply matching labels from `.github/labeler.yml` or repo label set if authorized by the issue template.
4. **One Issue = One Problem:**
   - Ensure the report covers only this single verified defect.
5. **Submit & Log:**
   - Submit issue via GitHub API / `gh issue create`.
   - Record issue URL against candidate in the run log.

### Stage 6 Exit Gate — ALL must be true to proceed

- [ ] Security classification explicitly logged (`SECURITY_SENSITIVE: YES/NO`).
- [ ] If YES: submitted via private disclosure (PVR / `SECURITY.md`), public pipeline halted, logged as `routed-to-private-disclosure`.
- [ ] If NO: issue drafted conforming to repo template, verified MRE included, submitted publicly, and issue URL recorded.
- [ ] Issue title verified clean - does NOT start with `fix:`, `feat:`, or any `type(scope):` prefix.

---

## 9. Anti-Patterns — DO NOT

- **DO NOT** file an issue without an actually-executed, verified MRE. Theoretical deductions are not bugs.
- **DO NOT** self-grade or accept a secondary model pass as verification. Execution against real code is required.
- **DO NOT** cite files, functions, lines, or behaviors not read fresh in the current session. Hallucinated code references destroy maintainer credibility.
- **DO NOT** disclose security-sensitive findings publicly under any circumstances.
- **DO NOT** downgrade a security vulnerability into a "sanitized" or "vague" public issue.
- **DO NOT** flag intentional design, documented limitations, or accepted trade-offs as bugs. Check commit history and documentation first.
- **DO NOT** misattribute upstream framework/runtime bugs to the target repository.
- **DO NOT** prefix issue titles with Conventional Commit tags (e.g., `fix:`, `fix(scope):`, `feat:`, `chore:`). Conventional Commits belong strictly on commit messages and pull request titles, never on GitHub issue trackers.
- **DO NOT** file style, formatting, naming, linting, or subjective refactoring recommendations.
- **DO NOT** submit generic, low-effort suggestions ("add more unit tests", "improve error handling").
- **DO NOT** bundle multiple unrelated findings into a single issue.
- **DO NOT** test against stale clones, unreleased branches, or old tags. Always verify against latest default `HEAD`.
- **DO NOT** maximize issue volume. Volume is a liability; high signal is the only success criterion.
- **DO NOT** bypass repository contribution guidelines, issue templates, or code-of-conduct policies.

---

<a id="appendix-a--responsible-disclosure--advisory-protocols"></a>
## 10. Appendix A — Responsible Disclosure & Advisory Protocols

When routing a security-sensitive finding to private channels (GitHub Private Vulnerability Reporting, email specified in `SECURITY.md`, or maintainer security contacts), structure the advisory as follows:

```markdown
# [Vulnerability Type] in [Component/Module]: [Concise Summary]

## Metadata
- **Affected Repository:** [owner/repo]
- **Affected Component:** [module/package/file]
- **Vulnerability Category:** [CWE classification, e.g., CWE-208, CWE-89, CWE-79]
- **Estimated Severity:** [Critical / High / Medium / Low] (CVSS v3.1 if applicable)
- **Vulnerable Versions:** [Commit SHA or release version range]

## Summary
[Concise executive summary describing the vulnerability and its core mechanism.]

## Threat Scenario & Impact
[Concrete description of how an attacker could exploit this defect, what access/privilege is required, and what confidentiality, integrity, or availability impact results.]

## Steps to Reproduce & Minimal Reproducible Example
1. [Step 1]
2. [Step 2]
3. [Run reproduction script below]

```[language]
// Minimal, self-contained reproduction script (MRE)
...
```

## Observed Output
```
[Actual execution output / terminal log demonstrating the vulnerability]
```

## Proposed Remediation (Optional)
[Objective, unpresumptuous suggestion for fixing the vulnerability, citing specific lines if applicable.]

## Disclosure Timeline & Embargo
We adhere to responsible disclosure practices and request acknowledgment of this report within [e.g., 7 days]. We request a coordinated disclosure window (standard 90 days) prior to any public disclosure to allow for patch development and release.
```

---

## 11. Appendix B — Candidate Funnel Tracking

Every execution of this pipeline maintains a transparent candidate log tracking every finding from inception to final disposition. This ensures auditability across pipeline runs:

### Disposition Statuses

- `discarded@stage1 (out-of-scope)`
- `discarded@stage3 (no concrete file:line citation)`
- `discarded@stage4 (cannot reproduce in isolation)`
- `discarded@stage4 (unreachable code path)`
- `discarded@stage4 (mitigated upstream)`
- `discarded@stage4 (intended / documented behavior)`
- `discarded@stage5 (duplicate of #<issue_number>)`
- `discarded@stage5 (closed as wontfix / by-design)`
- `discarded@stage5 (insufficient impact / style nitpick)`
- `routed-to-private-disclosure (CWE-<id> via <channel>)`
- `filed-public-issue (#<issue_number>)`

### Run Summary Table Format

| Candidate ID | Target File:Line | Discovery Vector | Stage Exited | Final Disposition |
|---|---|---|---|---|
| C-01 | `src/auth/token.ts:142` | Timing Analysis | Stage 6 | `routed-to-private-disclosure (CWE-208 via PVR)` |
| C-02 | `src/http/parser.ts:88` | Boundary Fuzzing | Stage 4 | `discarded@stage4 (mitigated upstream by router)` |
| C-03 | `src/cache/store.ts:210` | Concurrency Sweep | Stage 6 | `filed-public-issue (#1042)` |
| C-04 | `src/utils/format.ts:35` | Pattern Grep | Stage 5 | `discarded@stage5 (insufficient impact / style nitpick)` |
