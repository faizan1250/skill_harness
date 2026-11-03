---
name: elite-backend-engineering
description: Enforces production-grade security, performance, and reliability discipline for all backend code — Next.js Server Actions, Route Handlers, and middleware/proxy; FastAPI endpoints; database queries and migrations; auth/authz logic; caching, rate limiting, and background jobs. Trigger this for ANY backend code written, reviewed, or modified for a production system handling real user data and traffic — even a single endpoint, one query, or one action, since every handler is a potential public attack surface and production incident.
---

# Elite Backend Engineering (Next.js + FastAPI)

Standard enforced: this code runs in production for large-brand clients, under real traffic, real attackers, and real financial/reputational stakes. Zero tolerance for "it works on my machine." Every rule below is a directive, not a suggestion.

## The Trust Boundary Rule (foundational)

Every Server Action, Route Handler, and FastAPI endpoint is a public HTTP endpoint reachable directly by `curl`, regardless of what the UI implies. Next.js's own docs state Server Actions are reachable via direct POST; encrypted non-deterministic action IDs and the automatic Origin/Host CSRF check are transport hardening, not authorization. Client-side types, client-side validation, hidden buttons, and "only admins see this route in the UI" provide **zero** security. Next.js 16 renamed Middleware to **Proxy** — treat it as a routing convenience, never the final security boundary. CVE-2025-29927 (CVSS 9.1) let attackers skip all middleware logic via a crafted `x-middleware-subrequest` header, bypassing auth on protected routes entirely; CVE-2026-64643 (patched Next.js 15.5.21 / 16.2.11, July 2026) disclosed internal Server Action IDs to unauthenticated users. Both prove the same lesson: **re-verify inside the handler itself, every time**, regardless of what ran upstream. Every handler independently checks:
(a) **who** is calling — authentication
(b) **whether they may do THIS action on THIS specific object** — object-level authorization, not just route-level
(c) **that the payload is valid** — schema validation

- **Next.js Control-Flow Safety**: Never swallow Next.js internal control-flow exceptions in a generic `try/catch`. If wrapping Server Actions or Route Handlers, re-throw immediately when `isRedirectError(err)` or `isNotFoundError(err)` is true (check `err?.digest?.startsWith('NEXT_REDIRECT')` or `'NEXT_NOT_FOUND'`). Catching these turns intentional redirects and not-found responses into silent 500 failures.

- **Webhook Security**: Verify signatures exclusively against the **raw, unparsed request body** (`await req.text()` / `await req.arrayBuffer()` in Next.js; `await request.body()` in FastAPI) before any JSON parsing. Enforce a strict replay window — reject webhooks where the timestamp header falls outside ±300 seconds of server time. Webhook endpoints must be public (bypass session auth) but strictly guarded by constant-time signature verification.

## Authentication & Authorization

- Auth ≠ authz — "logged in" never implies "allowed to do this."
- Object-level authorization on every resource access: never trust an ID in the request without checking ownership/permission server-side. Broken Object Level Authorization (BOLA/IDOR) is API1 in the OWASP API Security Top 10 and the most common real-world API finding.
- **Multi-Tenant Isolation**: In multi-tenant systems, never query by primary key alone (`WHERE id = :id`). Every query and mutation must be scoped by the authenticated caller's tenant (`WHERE id = :id AND tenant_id = :current_tenant_id`), or enforced globally at the database layer via PostgreSQL Row-Level Security (RLS) with session-scoped tenant claims (`SET LOCAL app.current_tenant_id`). Missing this is the most common multi-tenancy data leak.
- **Constant-Time Verification**: Never compare hashes, API keys, session tokens, or cryptographic signatures with standard equality (`===` or `==`). Always use constant-time comparison: `crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b))` in Node.js and `hmac.compare_digest(a, b)` in Python. Standard equality exits early on the first differing byte, leaking key length and content via nanosecond timing differences to an attacker making repeated requests.
- Session cookies: `httpOnly`, `Secure`, `SameSite=Lax`/`Strict`, `__Host-` prefix where applicable; short `maxAge` (1–2h) for privileged/admin actions, longer (7–30d) for normal web sessions; rotate the session ID on privilege escalation. Use database- or Redis-backed sessions (not just signed cookies) when you need immediate revocation or active-session visibility.
- Never store tokens in `localStorage`/`sessionStorage` — anything JS-readable is XSS-readable.
- JWT payload: identifiers, roles, scopes only — never secrets or PII. Signing prevents tampering, not inspection.
- Return 401 (unauthenticated) vs. 403 (unauthorized) distinctly server-side; don't let the distinction leak exploitable detail to the client beyond the status code.
- FastAPI: `OAuth2PasswordBearer` + scopes for fine-grained RBAC. Deny-by-default; grant least privilege per scope/role, not blanket "logged-in" access.

## Input Validation & Schema Enforcement

- Validate at every trust boundary, not just the outermost one: Zod at every Server Action/Route Handler entry, Pydantic v2 at every FastAPI endpoint.
- No manual `if`/`else` validation chains — schema-first, always.
- Reject unknown fields by default: Zod `.strict()`, Pydantic `ConfigDict(extra="forbid")`. Prevents mass-assignment-style vulnerabilities.
- A passing schema is **not** authorization — a well-formed object can still point at a row the caller doesn't own. Validation and object-level authz are separate, both mandatory checks.
- Validate types, ranges, and business-rule constraints — not just presence.
- **Strict Output DTOs / Response Filtering (OWASP API3)**: Never return raw ORM entities or database models directly to the client. Define explicit response schemas for every endpoint. FastAPI: enforce `response_model=...` with explicit fields or return dedicated Pydantic schemas. Next.js: explicitly map and select returned fields before returning from Route Handlers or Server Actions — never spread database records into client-facing JSON. Returning raw entities leaks password hashes, internal IDs, Stripe customer IDs, and tenant metadata.

## Database & Query Discipline (PostgreSQL)

- Index every foreign key, filter column, and sort column — no exceptions without a documented reason.
- Composite index column order: equality columns first (highest-cardinality first among equals), range columns next, `ORDER BY` columns last. Reversing this order is one of the most common, most expensive mistakes in production.
- Verify with `EXPLAIN (ANALYZE, BUFFERS)` before shipping any non-trivial query — trust actual rows/time, not planner estimates. A `Seq Scan` on a large table is a red flag.
- Eliminate N+1 via `selectinload()` for one-to-many/collections and `joinedload()` for many-to-one — never loop-and-query. A single query firing thousands of times at sub-millisecond cost still adds up to real seconds of DB time.
- Never `SELECT *` in production code paths.
- Keyset/seek pagination for any list that can grow large: `WHERE (created_at, id) < (:last_created, :last_id) ORDER BY created_at DESC, id DESC LIMIT :n`. Deep `OFFSET` degrades linearly with depth and skips/duplicates rows under concurrent writes — never use it beyond small, bounded lists.
- Wrap multi-step mutations in an explicit transaction. **Transactions alone do not stop race conditions**: PostgreSQL `READ COMMITTED` (the default isolation level) allows lost updates and check-then-act races. For balance deductions, inventory counts, or quota increments, mandate one of: (1) atomic SQL expressions (`UPDATE accounts SET balance = balance - :amt WHERE id = :id AND balance >= :amt`), (2) pessimistic locking (`SELECT ... FOR UPDATE` inside the transaction), or (3) optimistic concurrency control (version-stamped rows: `WHERE version = :expected_version`).
- **Explicit Connection Management — distinguish stateful vs. serverless runtimes**:
  - **FastAPI / stateful containers**: Explicit pool sizing (`pool_size`, `max_overflow`, `pool_pre_ping=True`, `pool_recycle` set below the database's idle-connection timeout, e.g., 180s under a ~10min managed-Postgres idle cutoff). Never rely on driver defaults for production load.
  - **Next.js / serverless (Vercel, Lambda)**: Never run standard multi-connection pools — each concurrent cold-start instantiates its own pool, exhausting `max_connections` instantly (e.g., 50 lambdas × 20 connections = 1,000 DB connections). Mandate an external transaction-mode proxy (PgBouncer, AWS RDS Proxy, Supabase Supavisor) with `max_connections=1` at the app level, or use HTTP/WebSocket edge drivers (Neon, PlanetScale, Turso).
- `expire_on_commit=False` on async sessionmakers — avoids implicit lazy-load-after-commit errors under asyncio, since there's no implicit re-fetch across an `await`.
- **Strict `AsyncSession` lifecycle**: Wrap `yield session` in FastAPI dependencies with `try...except...finally`. On any unhandled exception call `await session.rollback()`; always call `await session.close()` in `finally`. Never let an un-rolled-back session return to the pool — it can hold open transactions or carry dirty state for the next caller.
- **Zero-Downtime Migration Discipline (Expand/Contract)**: Migrations must always be backward-compatible with currently running application code, since deployments are rolling and migrations run before new code is fully live:
  - Never rename or drop a column in a single release. Step 1 (Expand): add new column, dual-write to both old and new. Step 2: backfill existing rows. Step 3 (Contract): switch reads to new column. Step 4: drop old column in a subsequent deployment.
  - Adding `NOT NULL` on an existing large table must use `ALTER TABLE ADD CONSTRAINT ... NOT VALID` followed by `VALIDATE CONSTRAINT` to avoid a long table-exclusive lock that blocks all reads and writes during migration.

## Async Correctness (FastAPI)

- Never mix blocking calls inside `async def` — `time.sleep`, `requests.get`, sync DB drivers, sync file I/O freeze the **entire single-threaded event loop**, stalling every concurrent request, not just the caller.
- Use async-native equivalents always: `asyncio.sleep`, `httpx.AsyncClient`, `asyncpg`/`AsyncSession`, `aiofiles`.
- If a dependency only has a sync driver, declare the route as plain `def` (FastAPI runs it in the AnyIO worker threadpool automatically). **Critical constraint**: the default AnyIO threadpool cap is 40 workers. Slow blocking I/O on `def` routes will saturate this pool and starve the entire server including health checks. Tune `anyio.to_thread.current_default_thread_limiter().total_tokens` for heavy sync workloads, or offload prolonged sync tasks to an isolated worker process.
- CPU-bound work never belongs in the request path — offload to a worker/background job.
- `BackgroundTasks` run *after* the response is sent but still on the *same* event loop — a blocking call inside one still freezes concurrent requests. Use a real task queue (Celery/ARQ/Dramatiq) for anything non-trivial, long-running, or requiring true parallelism.
- Use `Depends()` for DB sessions, auth, and service injection — never global state or module-level singletons.
- **Request Cancellation Propagation**: Bind long-running operations to client disconnect signals to avoid executing orphaned work after the client has gone. Next.js: pass `req.signal` from Route Handlers into downstream `fetch()` calls and DB query drivers. FastAPI: check `await request.is_disconnected()` before initiating expensive multi-step sub-tasks.

## Idempotency & Reliability

- Assume at-least-once delivery everywhere: retries, webhooks, and queue consumers can all invoke a handler twice. Every mutation handler must be idempotent.
- Require an idempotency key on all financial or non-repeatable mutations:
  - **Route Handlers & FastAPI**: read from the standard `Idempotency-Key` HTTP header.
  - **Next.js Server Actions**: pass `idempotencyKey: string` directly within the validated input payload schema — Server Actions are RPC calls and you cannot append arbitrary HTTP headers to them without tearing apart the action dispatcher.
  - Store the key alongside a SHA-256 hash of the canonicalized payload via a **database unique constraint written in the same transaction as the mutation**. A `SETNX`-then-process flow has a TOCTOU race that lets two concurrent retries both believe they claimed the key.
- If the same key arrives with a different body hash, reject with 409 — never silently return a cached response for a different payload.
- **Handle In-Flight Idempotency States**: An idempotency record must carry an explicit lifecycle (`PROCESSING` → `COMPLETED` / `FAILED`). Insert with status `PROCESSING` before starting any external side effect. If a concurrent duplicate arrives while status is `PROCESSING`, return `409 Conflict` (or `425 Too Early`) with a `Retry-After` header. Once completed, update to `COMPLETED` and persist the response body so subsequent identical requests return the cached payload immediately without re-executing.
- Background jobs: small, single-responsibility, idempotent by design.
- Retries: exponential backoff with full jitter — `delay = random(0, min(base * 2^attempt, cap))` — bounded attempts, landing in a dead-letter queue on exhaustion. Never unbounded retry loops.
- Circuit breakers around flaky downstream dependencies instead of hammering them mid-outage.

## Rate Limiting & Caching

- Rate-limit by identity (user/API key) primarily, IP as a secondary layer. Unrestricted Resource Consumption is API4 in the OWASP API Security Top 10.
- Atomic operations only — Redis `INCR`+`EXPIRE`, or a Lua script for token-bucket/sliding-window. A naive read-then-write has a race window under concurrent load; a Lua script executes atomically server-side in one round trip, no `MULTI`/`EXEC` retry loop needed.
- Sliding-window log (Redis sorted set: prune + count + add in one Lua call) for boundary-accuracy-sensitive endpoints (auth, payments); token bucket where controlled bursts are acceptable.
- Cache-aside for reads with an explicit TTL on every key — never write a cache key without an expiry.
- Invalidate cache explicitly on writes; don't rely on TTL alone for correctness-sensitive data.
- Apply stricter limits to expensive/mutating endpoints than to cheap idempotent GETs.

## SSRF & Outbound Requests

- Any endpoint fetching a user-supplied URL (webhooks, avatar import, link preview, "import from URL") is an SSRF vector — API7 in the OWASP API Security Top 10.
- Validate outbound targets strictly: resolve DNS, normalize addresses (including IPv6-mapped IPv4 `::ffff:x.x.x.x` and integer-encoded IP representations), and verify the resolved IP is not within RFC 1918, RFC 3927 (link-local / cloud metadata `169.254.169.254`), or loopback ranges. **Pin the resolved IP**: execute the outbound HTTP request directly to the validated IP address with the original domain passed via the `Host` header and TLS SNI — this eliminates DNS rebinding (TOCTOU between resolution and connection caused by low-TTL records flipping to an internal address after validation passes).
- Disable redirect-following on outbound HTTP clients, or re-validate the target on every hop — a validated URL can 302 to an internal address.
- Route outbound fetches through a single egress proxy/allowlist point for auditability where the app makes many external calls.

## Secrets & Data Protection

- Secrets live in a secret manager (AWS Secrets Manager, Vault, GCP Secret Manager) — never in `.env` files committed to any repo, never hardcoded. Commit only `.env.example` with placeholders.
- If a secret is ever committed, rotate it immediately — scrubbing git history does not un-leak an already-exposed value.
- Sensitive fields (tokens, PII where required) encrypted at rest at the field level, with keys stored separately from the application/database.
- Show high-value secrets (API keys) to the user exactly once at creation — never re-displayable, only regenerable.
- Never log secrets, tokens, passwords, connection strings, or full PII — log identifiers/hashes instead. A single debug log line containing a connection string, scraped by a log aggregator, is one of the most common real-world leak vectors.
- Generic error messages to clients; full error detail only in server-side logs.

## Observability

- Structured JSON logging with a correlation/trace ID (`trace_id`/`span_id`, OpenTelemetry conventions) propagated across every request and background job.
- Log at service boundaries — entry, exit, error — with enough context to reconstruct what happened without redeploying.
- Distributed tracing for any request touching more than one service or downstream call.
- Alert on what actually predicts incidents: error rate, p95/p99 latency, queue depth, dead-letter-queue growth — not just uptime. A growing DLQ is a leading failure indicator, not a lagging one.

## Error Handling Contract

- Every endpoint handles: validation failure, auth failure, not-found, downstream failure, and success — no bare happy-path-only handlers.
- Never leak stack traces, internal paths, or ORM/SQL details to the client.
- Distinguish expected errors (4xx — handled gracefully, logged at info/warn) from unexpected ones (5xx — logged loudly, alerted).
- **Server Action Error Contract**: Never throw generic exceptions from Server Actions for domain or validation errors — Next.js production builds mask all thrown errors into opaque digest strings (e.g., `digest: "12345678"`), giving the client zero actionable feedback. Instead, return a typed discriminated union: `type ActionResult<T> = { success: true; data: T } | { success: false; error: string; code: string; fieldErrors?: Record<string, string[]> }`. Reserve thrown exceptions exclusively for fatal unhandled crashes caught by root error boundaries.

## Pre-Ship Checklist

- [ ] **Object-Level & Tenant Auth**: Verified ownership AND tenant boundary (`tenant_id` / RLS) inside the handler — zero blind trust of IDs or upstream proxies
- [ ] **Schema In & Out**: Inputs strictly validated (`.strict()` / `extra="forbid"`); response models/DTOs explicitly defined to prevent API3 data leakage
- [ ] **Control Flow Safe**: No generic `try/catch` swallowing Next.js `redirect()` or `notFound()` — re-throw on `NEXT_REDIRECT` / `NEXT_NOT_FOUND` digest
- [ ] **Server Action Contract**: Expected client errors returned as typed `{ success: false, error, code }` objects, not thrown exceptions
- [ ] **Connection Architecture**: Long-lived pools for FastAPI; connection proxies (PgBouncer/RDS Proxy) or HTTP drivers for serverless Next.js
- [ ] **Async & Threading**: No blocking calls in `async def`; AnyIO threadpool limits tuned if using sync `def`; `AsyncSession` closed with rollback on exception in `finally`
- [ ] **Constant-Time Crypto**: Signatures, tokens, and keys compared with `timingSafeEqual()` / `compare_digest()` — never standard equality
- [ ] **Webhook Security**: Signatures verified against raw unparsed body; timestamp within ±300s enforced
- [ ] **Race-Free Concurrency**: Balance/quota/inventory mutations protected by atomic SQL, `SELECT FOR UPDATE`, or optimistic version locks — not just transaction wrapping
- [ ] **Idempotency Lifecycle**: Financial/mutation endpoints guarded by payload hash + unique constraint; `PROCESSING` → `COMPLETED` / `FAILED` states handled; in-flight duplicates return 409 + `Retry-After`
- [ ] **Query Plan Verified**: Indexes verified with `EXPLAIN (ANALYZE, BUFFERS)`; zero N+1 (`selectinload` / `joinedload`)
- [ ] **Zero-Downtime Schema**: Migrations follow expand/contract — no breaking column renames/drops or bare `NOT NULL` additions on large tables in a single release
- [ ] **Hardened SSRF**: Outbound URLs resolved, normalized (including IPv6-mapped IPv4), validated against private/metadata IP blocks; IP pinned post-resolution to prevent DNS rebinding
- [ ] **Request Cancellation**: Long-running ops bound to `req.signal` (Next.js) or `request.is_disconnected()` (FastAPI) to avoid orphaned execution
- [ ] **Secrets Never Logged or Hardcoded**: No tokens, connection strings, or PII in logs
- [ ] **Observability**: Trace ID propagated through structured logs on every request and background job