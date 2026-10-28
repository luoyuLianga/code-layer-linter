---
name: code-layer-linter
description: Pre-merge layer linter — split a diff into concrete change points (lock, transaction boundary, remote call, validation, cache, MQ send, state transition…) and judge whether each sits in the correct layer/module by industry layering principles and the repo's own conforming precedents. Advisory only — the submitter decides what to fix.
---

# Code Layer Linter

> Orthogonal to design review (should we abstract?), stability review (will it crash?), and code review (is there a bug?).
> This skill answers **one question only — is each change point in the right layer?**

## In One Line

**Judge each change point against industry layering principles (P1–P6), using the repo's conforming precedents as supporting evidence. Advisory only — submitter decides.**
Check scope follows the diff: N change points in, N judgments out.
**Scope: intra-repo layering only.** Cross-service ownership belongs in design review, not here.

## Principles (P1–P6)

| # | Principle | Quick Test |
|---|-----------|------------|
| P1 | **Entry layer translates protocol only** | Controller / Facade / MQ consumer / scheduled-task entry should only: receive & validate params, declare auth, call the layer below, assemble the response. Business logic, transactions, locks, or direct DB/external calls appearing here → wrong layer |
| P2 | **Guarantees live where every entry point must pass** | Business rules, invariants, concurrency control (locks), idempotency, state transitions, and transaction boundaries belong in core/domain services. **Litmus test: if a different entry point (another API, MQ consumer, cron job) invokes the same business logic, does this guarantee still hold? No → wrong layer** |
| P3 | **Technical details hide behind infrastructure** | SQL/Mapper → DAL/repository; external HTTP/SDK → integration client (timeouts, retries, error-code translation, external DTO conversion all live here). Core must not build SQL or use third-party SDK types directly |
| P4 | **Dependencies flow downward only** | Entry → Core → DAL / Integration. Lower layers never import upper-layer types (facade DTOs, controllers). Same-layer services must not form cycles — break with events or extracted components, not by moving layers. **Common/shared modules hold only cross-module utilities, constants, and base DTOs. Business interfaces, business enums, and business-semantic exceptions belong in core, not in common** |
| P5 | **Cross-cutting concerns use the framework's unified mechanism** | Logging, auditing, metrics, auth, exception-to-error-code mapping: if the repo already has an aspect / interceptor / global exception handler for it, use that — don't hand-code a copy in business logic |
| P6 | **Model conversion at boundaries** | External DTO ↔ internal model conversion belongs at the boundary (facade assembler, integration converter). Persistence objects must not leak above core; external DTOs must not sink into core |

**P1–P6 are the standard (industry layering consensus). Repo precedents are supporting evidence, not overrides.**
Use conforming precedents for reference ("similar `X` is also in core") to locate the right class and package.
When a precedent violates P1–P6, still judge the current change by the principles, and note: "legacy precedent `XXX` also violates P_, recommend fixing together."

**Layer-escape red line**: moving logic to a different layer to work around a framework limitation (e.g., `@Transactional` self-invocation in Java, circular dependency in DI containers, ORM proxy boundaries, Python circular imports) must always be flagged ⚠️.
State: "should be in layer X; placed in Z because of Y; the in-layer solution is W" (e.g., explicit transaction API, extract a same-layer component, publish a domain event).

## Execution Steps

### 1. Determine Scope & Layer Map

- Diff scope: `git diff origin/<target-branch>...HEAD` (three-dot — only this branch's own changes).
- Write out the repo's **layer map** by module/package, e.g.:
  `*-service-impl` (facade/entry) → `*-core-service` (core) → `*-dal` (DAL) / `integration` (external calls) / `*-common`.
  If unsure, `ls` the modules and packages first — don't guess.

### 2. Split Into Change Points (The Critical Step)

A change point ≠ a file, ≠ a method. **One change point = one responsibility.** Common types:

> Lock / transaction boundary / idempotency or dedup / state transition / business validation / param format validation / auth / external API call / retry & timeout / cache read-write / MQ send or consume / event publish or listen / scheduled task / model conversion / exception-to-error-code / config or feature-flag read / logging · audit · metrics / new data access (query or write)

For each, write: **what it does (one line) + where it is (`Class#method`, which layer)**.
One method may yield multiple change points — split by responsibility, not by method. Example: lock + transaction + remote call in one method = 3 change points, each judged separately.

Pure copy edits, comments, changes to **existing** constant values, and test code are not change points — summarize in one line as "no placement judgment needed."
New enum classes, exception classes, and utility classes still count (they have placement implications).

### 3. Judge Each Point

For each change point:

1. **Find conforming precedents**: `grep` the repo for similar patterns. Note the precedent location and its layer.
   - Java/Spring: `getLock(`, `TransactionTemplate`, `@Transactional`, `RestTemplate`/`FeignClient`, `@Scheduled`, `@RabbitListener`
   - Go: `sync.Mutex`, `sql.Tx`/`Begin()`, `http.Client`, `cron.New()`, `amqp.Channel`
   - Python: `@transaction.atomic`, `redis.lock()`, `httpx.Client`/`requests.Session`, `@celery.task`, `pika.channel`
   - TypeScript/Node: `prisma.$transaction`, `mutex.acquire()`, `axios`/`fetch`, `bullmq`, `@nestjs/schedule`
2. **Compare against P1–P6**:
   - ✅ **Correct** — basis (principle # or precedent class)
   - ⚠️ **Suggest moving** — from where to where, basis, estimated effort (lines/files)
   - ❓ **Uncertain** — no precedent found and principles don't clearly decide; state both sides

**Noise reduction**:
- Every ⚠️ must include both "move to where" and "basis (principle # / precedent class)." Can't provide both → don't report it.
- Only report placement issues. Naming, style, bugs, and performance are out of scope.
- A change point that is an **as-is reuse** of an existing misplacement (not newly introduced) is labeled "legacy" — not counted against the current change.

### 4. Output (Fixed Format)

```
Layer map: <one line>
No placement judgment needed: <files or note, one line>

| # | Change point (what) | Current location (Class#method / layer) | Verdict | Basis | Suggestion |
|---|---|---|---|---|---|
| 1 | Concurrency lock for first-time setup | UserSetupFacadeImpl#initialize / facade | ⚠️ | P2 (other entry points bypass this lock); precedent: OrderCoreServiceImpl locks in core | Move into SetupCoreServiceImpl; use TransactionTemplate inside lock (precedent: PaymentCoreServiceImpl), ~40 lines / 3 files |

Summary: ✅ N / ⚠️ N / ❓ N
Action: reply with numbers to fix; no reply = skip, does not block merge. ❓ items need your call.
```

### 5. After Fixes

Re-run steps 2–4 only for fixed items. Don't re-check everything. Skipped ⚠️ items are not re-reported.

## When to Run

- **Before merge (required)**: as a step in your pre-merge checklist. Advisory only — does not block the merge verdict.
- **Before writing code (recommended)**: review the plan against P1–P6 before implementation — cheaper than fixing at merge time.

## Example (Illustrating the P2 Litmus Test)

A user-setup feature adds a distributed lock in the **entry/facade** layer to prevent concurrent first-time initialization. The lock was placed there to work around a framework limitation with declarative transactions in the core layer.

**P2 litmus test**: another entry point (a background job or a different API) also triggers first-time setup but doesn't go through this facade method → the concurrency guarantee is lost → ⚠️.

**Correct fix**: place the lock in the core layer using an explicit transaction API (e.g., `TransactionTemplate` in Java, `BEGIN/COMMIT` in Go, `transaction.atomic()` in Python) so the transaction commits before the lock is released.
