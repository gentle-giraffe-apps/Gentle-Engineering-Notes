# Integration Testing Under Realistic Load (Without Hurting the Backend)

This guide covers **how to run integration tests that create/delete accounts and seed “real-world” data at meaningful scale**—while still being safe for the backend and useful for finding **latency, contention, and scalability bugs** that lightweight tests miss.

> Goal: move from “does it work?” to “does it work under realistic pressure?”

---

## Why “lightweight” integration tests miss real problems

Many integration tests validate correctness with minimal data:

- a single user
- a handful of entities
- no concurrency
- happy-path timings

That’s great for correctness—but it often fails to uncover:

- **N+1 queries** and expensive joins that only show up with large tables
- **index bloat / missing indexes**
- **queue backlogs** and async worker starvation
- **lock contention** (row/table locks, distributed locks)
- **cache stampedes** and thundering herds
- **rate limiting / circuit breaking behaviors**
- **slow downstream dependencies** (email, payments, search indexing)
- **p99/p999 latency spikes** that are invisible at small scale

“Induction” (“if it works for 10 rows it will work for 10M”) is often wrong because cost curves are rarely linear.

---

## The core constraint: “Don’t hurt production”

You generally **should not** run heavy-load integration tests against production. Even read-heavy can trigger:

- cache churn
- autoscaling events and cost spikes
- noisy-neighbor effects
- false alarms in monitoring
- data pollution

Instead, design a pipeline that **isolation-protects** production while still representing production reality.

---

# Recommended architecture

## 1) Use a dedicated environment for load-style integration tests

The safest default is a **separate environment**:

- **Staging/Pre-prod** with production-like infrastructure sizing
- **Shadow environment** (sometimes called “perf” or “soak”) used only for load/soak testing
- **Ephemeral environments** for each CI run (more expensive, but very clean)

**Best practices**
- Ensure configuration matches production in all the ways that matter:
  - DB engine/version, indexes, connection pool sizes
  - queue/worker sizing
  - caches and TTLs
  - rate limits
  - search index settings
  - feature flags
- Use **production-like anonymized datasets** (details below).

---

## 2) Seed realistic data safely: three patterns

### Pattern A: “Golden dataset” snapshots (fastest + most stable)
Maintain a **pre-seeded dataset** in your test environment:

- A curated set of accounts and data relationships
- “Small/Medium/Large” variants
- Reset via **DB snapshot restore** or **schema + restore**

**Pros**
- Very fast test setup
- Highly repeatable
- Great for regression tracking

**Cons**
- Requires snapshot tooling
- Must keep dataset current with schema changes

**When to use**
- Daily CI regressions
- p95/p99 latency trend monitoring
- reproducible bug hunts

---

### Pattern B: Deterministic synthetic data generation (flexible + programmable)
Generate realistic data during test setup:

- seeded RNG for determinism
- schema-aware factories
- relationship graphs that match real usage

**Pros**
- Can model new scenarios quickly
- Can scale data size gradually

**Cons**
- Can be slow to generate at high volumes unless optimized
- Needs careful realism (distributions, relationships)

**When to use**
- New feature testing
- Exploring what-if scaling behaviors

---

### Pattern C: Anonymized production-derived datasets (highest realism)
Export production data, anonymize it, and import into test env.

**Pros**
- Highest realism (shape of data, relationship density, edge cases)

**Cons**
- Requires careful privacy/security
- Needs ongoing data governance

**When to use**
- Perf/scaling validation for critical paths
- Query tuning + index verification

---

# Creating and deleting accounts safely

## Prefer “test partitions” over full deletion
Instead of creating accounts and hard-deleting them, use:

- a `test_run_id` field / tag
- a `tenant` / `namespace`
- a dedicated “test organization” root entity
- soft-deletes with TTL-based cleanup

This avoids expensive delete cascades, reduces vacuum/compaction churn, and makes cleanup safer.

**Anti-pattern**
- Running large cascaded deletes in the same DB used by others
- Deleting rows with heavy foreign key graphs frequently

**Better**
- Partition test data by run ID and delete by partition
- Use time-based partitioning and drop partitions (super cheap)

---

## Use TTL cleanup workers (garbage collection)
Have a backend-supported cleanup mechanism:

- Any entity created with `expires_at` or `ttl_seconds`
- A scheduled cleanup job removes expired test data
- Rate limited + chunked deletes (safe, backpressure-aware)

This lets tests **create realistically** without fear of long-term DB growth.

---

# How to simulate real-world load without “hammering” the backend

## 1) Separate “correctness tests” from “load/soak tests”

**Correctness integration tests**
- small data
- run on every PR
- strict pass/fail

**Load-style integration tests**
- larger datasets, concurrency
- run on schedule (nightly) or per-release
- trend-based evaluation (latency budget regressions)

This prevents turning PR validation into a backend DOS.

---

## 2) Add a backend “test mode” that is safer but still realistic

If your backend supports it, add a controlled test profile:

- uses dedicated resources (DB/schema/cluster)
- can bypass external side effects:
  - no emails/SMS
  - no billing
  - no irreversible third-party calls
- still executes core logic (auth, persistence, business rules)

**Important:** don’t make test mode “too fake”—you still want real query shapes and queue behavior.

---

## 3) Use rate limiting and concurrency budgets from the test harness

Your test runner should obey a budget:

- max requests/sec
- max concurrency
- max accounts created per minute
- max rows created per run
- max runtime

And it should adapt based on backend signals:

- HTTP 429/503
- queue depth
- p95 latency spikes

**Technique: adaptive load**
- Start at low concurrency
- Gradually ramp up (step function)
- Stop ramp if error rates or latency exceed thresholds

This is safer than blast-and-pray.

---

## 4) Prefer “shape realism” over “raw volume”

You can often find load-related bugs without billions of rows by ensuring you have:

- realistic relationship depth (e.g., 1 account → 10 projects → 500 items each)
- realistic distributions (many small accounts, few giant ones)
- realistic access patterns (read hot sets, write bursts)

**Example**
- 90% of accounts: 1–5 projects
- 9% of accounts: 6–50 projects
- 1% of accounts: 200–1000 projects (the “whales”)

Simulating “whales” is often where the bugs hide.

---

# Test scenarios that specifically catch load problems

## Scenario 1: “Cold start” account hydrate
Measure how long it takes to load an account’s dashboard when:

- caches are cold
- related tables are large
- the user has “whale” scale

**What to measure**
- p50/p95/p99 response time
- DB query count
- bytes returned
- cache hit ratio

---

## Scenario 2: Concurrency on shared resources
Simulate multiple clients updating the same “hot” entity:

- shared project, shared playlist, shared cart
- concurrent writes and reads

This catches:
- transaction isolation issues
- race conditions
- lock contention
- retry storms

---

## Scenario 3: Burst writes + async processing
Send bursts that enqueue background work:

- notifications
- search indexing
- analytics pipelines
- media processing jobs

Validate:
- queue depth stabilizes
- workers keep up
- “eventual consistency” completes within SLA

---

## Scenario 4: Long-tail pagination / deep history
Test endpoints that paginate deep into history with large datasets.

Common bugs:
- OFFSET pagination slowdowns
- missing composite indexes
- unbounded sorts

Prefer keyset pagination where possible.

---

# Designing a safe, efficient “scaffolding” system

## 1) Provide backend-supported “seed endpoints” (if feasible)
Instead of thousands of normal API calls, provide an internal-only endpoint like:

- `POST /internal/test/seedAccount`
- `POST /internal/test/seedWhaleAccount`
- `POST /internal/test/cleanup?runId=...`

These endpoints can:
- insert in batches
- enforce budgets
- tag data with run IDs
- avoid expensive side effects
- record seed metadata for debugging

**Security**
- available only in non-prod
- requires service auth + IP allowlists
- audited

---

## 2) Use bulk insert / batch APIs
If internal seed endpoints are not possible, provide batch endpoints:

- `POST /items/batch`
- `POST /events/batch`
- bulk upsert patterns

This reduces request overhead and is closer to real throughput.

---

## 3) Keep test data factories schema-aware
When you generate synthetic data, ensure:

- valid foreign keys
- valid state machines (draft → published → archived)
- realistic timestamp distributions
- realistic text lengths and blobs (don’t use “aaa” everywhere)

A “factory” system should allow:
- **scale knobs** (items per project, projects per account)
- **distribution knobs** (many small + few large)
- **feature knobs** (enable attachments, comments, tags)

---

# Backend protections you should ask for (or implement)

## Resource isolation
- Separate DB cluster, or at least separate schema + resource quotas
- Separate cache namespace
- Separate queue topics

## Circuit breakers and backpressure
- When the backend is struggling, tests should fail fast (or ramp down), not amplify load.

## Observability first
Tests are useless if you can’t see why they’re slow.

You want:
- request tracing (trace IDs propagated)
- DB query timing, lock waits
- queue depth, worker utilization
- cache hit rate, eviction rate
- p95/p99 latency charts by endpoint

---

# How to validate “real-world load conditions” in a test

## Key metrics (at minimum)
Track per scenario:
- success rate
- error types (4xx vs 5xx)
- p50/p95/p99 latency
- throughput (RPS)
- DB time per request
- queue time to completion (for async flows)

## Regression gates
Instead of strict single-value assertions, use budgets like:
- p95 latency must be within X% of last baseline
- error rate must be < Y%
- DB queries per request must not exceed N

Store baselines per release tag to compare across versions.

---

# CI/CD strategy

## Suggested test layers
1. **PR Gate:** small integration tests, deterministic, fast
2. **Nightly:** medium dataset, moderate concurrency
3. **Pre-release:** large dataset + ramp + soak
4. **Post-deploy:** canary “smoke + latency budget” checks

## Keep “known releases” runnable
Tag and retain previous releases so you can run:
- the same scenarios against old versions
- compare performance and correctness
- bisect regressions

This is especially valuable when you suspect a backend change introduced a regression.

---

# Practical recipe: a safe “heavy-load integration test” run

1. Restore or select a known dataset (Small/Med/Large) **OR** seed synthetic data tagged with `test_run_id`.
2. Warm up critical caches (optional and measured).
3. Ramp concurrency: 1 → 5 → 20 → 50 (example) while observing 429/503/p95.
4. Run scenario suite:
   - whale dashboard hydrate
   - deep pagination query
   - burst writes + async completion
   - concurrent updates to shared resources
5. Record metrics + traces per scenario.
6. Cleanup:
   - prefer partition drop or TTL cleanup
   - otherwise chunked deletes by `test_run_id`.
7. Produce a report:
   - deltas vs baseline
   - top slow endpoints
   - top slow DB queries
   - queue backlog behavior

---

# Common pitfalls

- **Too much load in PR CI** → creates flaky tests and angry backend teams
- **Unrealistic data** → green tests, real users still suffer
- **No observability** → you detect slowness but can’t diagnose it
- **Hard deletes with cascades** → test suite slowly destroys DB performance
- **Ignoring p99** → p50 looks fine, users still complain
- **No isolation** → tests compete with other staging users and become meaningless

---

# Checklist

- [ ] Separate environment or isolated resources for load testing
- [ ] Data seeded with realistic distributions and relationships
- [ ] Test harness enforces rate and concurrency budgets
- [ ] Scenarios target latency tail, contention, and async processing
- [ ] Baselines per release + regression budgets
- [ ] Cleanup via TTL or partitions, not massive cascade deletes
- [ ] Deep observability: traces, DB metrics, queue metrics

---

## Appendices

### A. “Scale knobs” you can standardize in config
- accounts: 10 / 100 / 1,000
- projects per account: distribution (p50=3, p95=20, p99=500)
- items per project: distribution (p50=50, p95=5,000)
- attachment rate: 0% / 5% / 25%
- comment depth: 0 / 2 / 20
- concurrency: 1 / 5 / 20 / 50
- ramp schedule: step size + hold time
- budget thresholds: p95, error rate, DB time

### B. Suggested naming
- “Correctness Integration Suite”
- “Performance Integration Suite”
- “Ramped Load Scenario Pack”
- “Soak + Tail Latency Pack”

---

## 🤖 Tooling Note

Portions of drafting and editorial refinement in this repository were accelerated using large language models (including ChatGPT, Claude, and Gemini) under direct human design, validation, and final approval. All technical decisions, code, and architectural conclusions are authored and verified by the repository maintainer.
