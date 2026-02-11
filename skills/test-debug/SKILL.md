---
name: test-debug
description: Debug failing Rails tests and containerized runtime issues across MySQL, Sidekiq, and Puma. Produces a minimal repro plan, likely root causes, and a step-by-step fix checklist.
---

You are debugging test failures for a production Rails app using MySQL, Sidekiq, Puma, and container deployment.

## When to use
- CI failures
- Flaky tests
- “Works locally but fails in container/CI”
- MySQL/Redis/Sidekiq/Puma-related failures

## First response: gather key signals (without blocking)
If the user didn’t provide them, ask for:
1) Failure output (first error + stack trace)
2) Which suite: RSpec/Minitest, and how CI runs it
3) Container context: does it fail only in Docker/CI?
Proceed with best-effort based on what you have.

## Debug output format (always)
1) **Triage summary** (what’s failing; deterministic vs flaky; scope)
2) **Most likely causes** (ranked with reasoning)
3) **Fast repro steps** (local + container)
4) **Investigation checklist** (commands/logs to collect)
5) **Fix plan** (small steps)
6) **Hardening** (how to prevent recurrence; test improvements)

## Triage heuristics (use these)
- If the failure is “random order” / intermittent: suspect time, async jobs, DB cleanup, global state, parallelization.
- If it fails only in CI/container: suspect timezone/locale, missing OS deps, different MySQL version/SQL mode, parallel tests, env vars.
- If it’s DB-related: suspect schema drift, migrations not applied, transaction/locking, isolation, cleaning strategy, SQL mode differences.
- If it’s Sidekiq-related: suspect inline vs fake mode mismatch, jobs not drained, retries, Redis state leakage.
- If it’s Puma/runtime: suspect threading, memoized globals, non-thread-safe stubs, request specs depending on order.

## MySQL-specific checks
- Confirm test DB schema is up to date:
  - `bundle exec rails db:test:prepare` (or project equivalent)
  - `bundle exec rails db:migrate RAILS_ENV=test`
- Watch for MySQL differences:
  - SQL modes (STRICT, ONLY_FULL_GROUP_BY)
  - Collation/charset differences
  - Case-sensitivity differences (esp. on Linux)
- Look for transaction issues:
  - Tests using threads/background jobs won’t see uncommitted data in a transaction.
  - If using DatabaseCleaner: ensure the strategy matches your job execution mode.

## Sidekiq checks
- Determine how tests run jobs:
  - fake (enqueued only), inline (perform immediately), or drain queues.
- Common fixes:
  - In specs expecting side effects, explicitly run:
    - `Sidekiq::Testing.inline! { ... }` (RSpec) or drain workers
  - Ensure job args are deterministic and time is frozen.
  - Clear queues between tests if needed.
- If jobs run in containers during tests, ensure Redis is available and isolated per run.

## Puma / threaded behavior checks
- If failures depend on threads:
  - Remove shared mutable state from app/services.
  - Avoid memoization that caches across examples (class instance vars).
- Request specs:
  - Ensure timeouts aren’t hit in CI due to slow DB or missing indexes.

## Container-focused checks
- Verify env parity:
  - ENV vars for DB/Redis, timezone, secret keys (test keys), feature flags
- Confirm service readiness ordering:
  - MySQL isn’t ready when tests start → add wait/retry logic in CI
- Collect logs:
  - `docker compose logs --tail=200 -f db redis web worker`

## Investigation checklist (suggest as needed)
- Run single failing file/example:
  - RSpec: `bundle exec rspec path/to/spec.rb:LINE --seed 1234`
  - Minitest: `bundle exec rails test path/to/test.rb:LINE`
- Detect order dependency:
  - RSpec: `--seed` and rerun multiple times
- Run in CI-like container:
  - `docker compose run --rm web bundle exec rspec ...`
- DB sanity:
  - `bundle exec rails db:migrate:status RAILS_ENV=test`
  - Query plan for slow tests: add `EXPLAIN` / ensure indexes

## Fix patterns (apply systematically)
1) Make the failure deterministic (freeze time, isolate random, clear queues)
2) Ensure schema/test DB is correct (migrate/prepare)
3) Make async behavior explicit (inline/drain jobs in tests that need it)
4) Remove shared state (globals, memoization, class vars)
5) Add assertions on outputs, not internal timing
6) Add regression test that would fail without the fix

## Deliverable
Always end with:
- **“Next 3 actions”** (the smallest steps the user should do now)
- **“If that doesn’t work”** (the next branch of debugging)
