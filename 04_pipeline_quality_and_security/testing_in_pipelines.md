# Testing in Pipelines — Interview Questions

**Subject:** CI/CD
**Topic:** Test Pyramid in CI, Flaky Test Detection, Test Parallelisation, Test Selection, Deflaking Strategies
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is the test pyramid, and how does it apply to CI pipelines?

**Answer:**

The **test pyramid** (Mike Cohn, 2009) is a heuristic for the relative quantity of tests at each level: a wide base of fast, hermetic unit tests; a middle layer of integration tests; a narrow top of slow end-to-end tests.

```
        ╱E2E╲          (5-10%)   slow, brittle, business-critical paths
       ╱─────╲
      ╱  Int  ╲        (15-25%)  service contract, DB, queue
     ╱─────────╲
    ╱   Unit    ╲      (70-80%)  fast, hermetic, focused
   ───────────────
```

**Why the shape matters in CI:**

- **Cost per test.** A unit test runs in milliseconds; an E2E test in minutes. A pipeline of 10,000 unit tests + 100 E2E tests is faster and cheaper than 1,000 of each.
- **Failure diagnosis.** A failing unit test points at one function. A failing E2E test points at "something between the browser and the database." Diagnosis cost grows with scope.
- **Flakiness scales with scope.** Unit tests are deterministic; E2E tests have networks, browsers, timing. Inverting the pyramid means fighting flakes constantly.

**Anti-patterns:**

| Shape | Name | Problem |
|-------|------|---------|
| Inverted pyramid | "Ice cream cone" | Slow, flaky, expensive CI |
| All middle | "Hourglass" | Missing fast feedback or end-to-end coverage |
| All E2E | "Pillar of fire" | Pipeline takes hours; merges blocked by flakes |

**Pipeline implication:** put unit tests in the PR-blocking path, integration in PR or post-merge, E2E in post-merge nightly. Fast feedback for the change author; thorough validation for `main`.

```yaml
jobs:
  unit:        # PR-blocking, < 2 min
    runs-on: ubuntu-latest
    steps: [..., { run: pytest tests/unit }]

  integration: # PR-blocking, < 8 min
    needs: unit
    runs-on: ubuntu-latest
    services: { postgres: { image: postgres:16 } }
    steps: [..., { run: pytest tests/integration }]

  e2e:         # post-merge or nightly only
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps: [..., { run: pytest tests/e2e }]
```

### Q2. What is a flaky test, and what causes flakiness?

**Answer:**

A **flaky test** is one that produces inconsistent pass/fail results without any code change — passing one run, failing the next, with no edit to the code or the test.

**Common causes:**

| Cause | Example |
|-------|---------|
| **Timing / race conditions** | Test polls a result with no synchronisation; passes when fast, fails when slow |
| **Order dependence** | Test A leaves global state that test B depends on; reorder, fail |
| **Shared resources** | Two tests use the same temp file path or DB row |
| **External dependencies** | Test hits a third-party API; that API is sometimes slow or down |
| **Time-of-day sensitivity** | "Yesterday's date" off-by-one at midnight UTC |
| **Random data** | Property-based test occasionally finds a real bug; not flaky but feels flaky |
| **Environment differences** | Passes on dev's M1 Mac, fails on Linux CI runner |
| **Resource contention** | Parallel tests starve each other for CPU/memory/file handles |
| **Test pollution** | A previous test mutates a singleton or env var |

**Why it matters:**

Even a 1% flake rate becomes catastrophic at scale. With 100 tests per pipeline at 1% individual flake, a pipeline has ~63% chance of at least one flake (`1 - 0.99^100`). Developers re-run pipelines, lose trust, eventually merge despite failures. The team's CI signal becomes noise.

**The cultural cost is the real cost:**

- "Just rerun it" becomes the default response to red builds
- Real failures get masked by assumed flakes
- The team stops believing CI's verdict
- Outages happen on changes that "passed CI" — but CI was meaningless

**Detection approach:**

- Track per-test pass rate across runs
- Anything below 99% is suspect
- Auto-quarantine tests below 95% and ticket for fix

### Q3. What is test parallelisation, and what are the two main forms?

**Answer:**

**Test parallelisation** runs many tests concurrently to reduce wall-clock time. Two complementary forms:

**1. Within-process parallelism — multiple workers, one runner.**

```bash
pytest -n auto              # pytest-xdist
go test -p 4 ./...          # go's built-in
mocha --parallel --jobs 4   # mocha
```

Lower overhead (one VM startup), but tests must be hermetic — no shared global state, no port conflicts, no shared temp dirs.

**2. Across-runner parallelism — sharding across runners.**

```yaml
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4, 5, 6, 7, 8]
    steps:
      - run: pytest --shard=${{ matrix.shard }}/8
```

Each runner gets 1/N of the test files. Independent VMs, full isolation, unbounded scaling. Higher fixed cost (8 runners spin up, each does setup).

**Combining both:**

Most large pipelines use both. 8 shards, each running `pytest -n 4`. 32-way parallelism on a 4,000-test suite drops a 30-minute wall-clock to ~1 minute (in theory; setup overhead and skew make it more like 2-3 minutes).

**Sharding strategies (best to worst):**

1. **By historical duration** — record per-test runtime, bin-pack into balanced shards. CircleCI, Buildkite, Knapsack Pro do this automatically.
2. **By count** — equal number of tests per shard. Simple, but a 10-second test and a 30-millisecond test count equally → straggler skew.
3. **By hash of test name** — deterministic but oblivious to duration.
4. **Round-robin alphabetical** — worst, predictable skew.

**Diminishing returns:**

Wall-clock time has a floor: the longest single test. Sharding 100 tests across 100 shards still takes as long as the slowest test plus fixed setup. Past that, adding shards costs money for zero speedup.

### Q4. How do you isolate tests from each other to prevent test pollution?

**Answer:**

**Test pollution** is when one test's side effects affect another. Isolation prevents it.

**Levels of isolation (cheapest to strictest):**

**1. Per-test fixtures with cleanup:**

```python
import pytest

@pytest.fixture
def db():
    conn = create_connection()
    conn.execute("BEGIN")
    yield conn
    conn.execute("ROLLBACK")
    conn.close()

def test_user_create(db):
    db.execute("INSERT INTO users ...")
```

Each test sees a clean DB state; rollback at teardown undoes any inserts.

**2. Per-test schema or namespace:**

```python
@pytest.fixture
def db_schema():
    schema = f"test_{uuid.uuid4().hex}"
    conn.execute(f"CREATE SCHEMA {schema}")
    yield schema
    conn.execute(f"DROP SCHEMA {schema} CASCADE")
```

Each test has its own schema, no cross-test interference even on parallel runs.

**3. Per-test container:**

```python
@pytest.fixture(scope="function")
def postgres():
    with PostgresContainer("postgres:16") as pg:
        yield pg
```

Heaviest, strongest isolation. Use sparingly (slow startup).

**4. Per-test temp dirs:**

```python
def test_writes_file(tmp_path):
    (tmp_path / "out.txt").write_text("hello")
```

`tmp_path` is per-test. Don't `os.chdir` into it without restoring the cwd — that's a global pollution.

**Common pollution sources to audit:**

- Module-level state (caches, singletons, registries)
- Environment variables (set in test 1, read in test 2)
- File system writes outside `tmp_path`
- Global random seed (set in conftest, but each test should reseed)
- Time mocking that escapes the test (`freezegun` not properly reset)
- Open ports that don't get released (use `:0` to let the OS pick)

**Detection technique — randomise test order:**

```bash
pytest --random-order      # pytest-randomly
go test -shuffle on
```

Runs tests in random order each time. Order-dependent tests fail probabilistically, exposing pollution.

### Q5. What is test selection, and when should you use it?

**Answer:**

**Test selection** runs only the tests likely to be affected by a change, instead of the full suite. The goal is fast PR feedback without losing coverage signal.

**Selection strategies:**

**1. Path-based (cheap, approximate):**

```yaml
- uses: dorny/paths-filter@v3
  id: changes
  with:
    filters: |
      api: [services/api/**]
      web: [services/web/**]

- if: steps.changes.outputs.api == 'true'
  run: pytest services/api/tests
```

If only the `api` directory changed, only `api` tests run. Doesn't catch cross-module dependencies.

**2. Build-graph-based (precise, requires tooling):**

Bazel and Pants build a dependency graph from BUILD files. They know "this test depends on these modules" and skip tests whose deps are unchanged.

```bash
bazel test $(bazel query 'rdeps(//..., set('"$(git diff --name-only main)"'))' --output=label)
```

**3. Test impact analysis (TIA — runtime trace):**

Tools like Microsoft's TIA, Launchable, or Coverage-based selection record per-test code coverage, then on a change run only tests whose covered code changed.

```bash
# Pseudocode
changed_files = git diff --name-only main
relevant_tests = [t for t in all_tests if t.coverage & changed_files]
pytest <relevant_tests>
```

**4. Predictive ML-based (Launchable, Buildkite Test Engine):**

Trains a model on historical test outcomes vs change features (file paths, commit metadata). Runs the top-N most likely to fail first, surfaces failures faster.

**When to use selection:**

| Situation | Use selection? |
|-----------|---------------|
| PR feedback in a monorepo with 50,000 tests | Yes |
| Small project (200 tests, runs in 30 sec) | No — overhead exceeds savings |
| `main` branch validation | No — run everything |
| Release candidate validation | No — run everything plus extras |

**Caveat:** never use selection on `main`. PRs may use selection for speed; merging to `main` should run the full suite to catch the corner cases selection missed.

### Q6. What metrics should you track for a CI pipeline's test health?

**Answer:**

**Quantitative metrics:**

| Metric | Target | Why |
|--------|--------|-----|
| **PR pipeline duration (p50)** | < 10 min | Developer attention budget |
| **PR pipeline duration (p95)** | < 20 min | Tail latency hurts more than mean |
| **Pipeline success rate (excl. real failures)** | > 99% | Anything less = developers lose trust |
| **Mean time to detect (MTTD)** | < 5 min | How fast does CI catch a breaking commit |
| **Mean time to recovery (MTTR) of `main`** | < 30 min | How fast can the team unblock |
| **Test flake rate** | < 1% per test, < 5% per pipeline | Threshold for noise to swamp signal |
| **Per-test runtime distribution** | No test > 30 sec | Slow tests dominate wall-clock |
| **Coverage** | > 80% line, watch trend | Trend matters more than absolute |
| **Test count growth** | Tracks LOC growth | If LOC grows but tests don't, debt is accumulating |

**Qualitative metrics:**

- "I trust CI's verdict" (developer survey, quarterly)
- Time spent debugging flaky tests vs writing features
- Frequency of "rerun all jobs" clicks (high = signal lost)

**Tooling to capture:**

```yaml
# Upload JUnit XML for analysis
- run: pytest --junit-xml=junit.xml
- uses: dorny/test-reporter@v1
  with:
    name: pytest
    path: junit.xml
    reporter: java-junit
```

```yaml
# Or push to a test analytics platform
- uses: codecov/test-results-action@v1
  with: { token: ${{ secrets.CODECOV_TOKEN }} }
```

Platforms like Buildkite Test Engine, CircleCI Insights, Datadog CI Visibility, and Trunk Flaky Tests aggregate JUnit XML across runs and compute these metrics over time.

**Interview insight:** the question "how do you know if your CI is healthy?" often distinguishes engineers who've owned a pipeline from those who've only used one. Mentioning specific metrics with thresholds (not just "we measure flakes") signals operational maturity.

---

## Intermediate

### Q7. How do you detect a flaky test programmatically, and what should you do when one is identified?

**Answer:**

**Detection algorithm:**

For each test, compute the pass rate over the last N runs on `main` (where the code is presumed correct):

```python
def is_flaky(test_id: str, lookback: int = 100) -> bool:
    runs = fetch_recent_runs(test_id, branch="main", limit=lookback)
    if len(runs) < 30:
        return False  # not enough data
    pass_rate = sum(r.passed for r in runs) / len(runs)
    return 0.05 < (1 - pass_rate) < 1.0  # fails sometimes, not always
```

**Higher-fidelity signal — re-run on failure:**

```yaml
- run: pytest --tb=short
- if: failure()
  run: pytest --last-failed --tb=short    # re-run just the failures
- if: failure() && success_on_retry
  run: |
    echo "::warning::Test was flaky — failed then passed on retry"
    # Push to flake tracker
```

If the test fails then passes, that's a flake by definition.

**Response playbook (in order):**

**1. Auto-quarantine (within minutes).**

Mark the test as `@pytest.mark.flaky` so it runs but doesn't block PRs:

```python
@pytest.mark.flaky(reruns=3)
def test_unstable():
    ...
```

Or move to a `quarantine/` directory excluded from required CI.

**2. Open a ticket automatically.**

```yaml
- if: steps.flake_check.outputs.is_flaky == 'true'
  run: |
    gh issue create --title "Flaky test: ${{ steps.flake_check.outputs.test_name }}" \
      --label flaky-test --body "Failure rate: ${{ steps.flake_check.outputs.rate }}%"
```

**3. Assign an owner.**

CODEOWNERS-style mapping — the team owning the test's directory gets the ticket.

**4. SLA for resolution.**

- Quarantined test must be fixed or deleted within 30 days
- After 30 days, the test is automatically deleted (the owner can revive it from Git)

**5. Don't accept "I reran it and it passed" as resolution.**

Flakes have causes. Either fix the cause or delete the test. "Sometimes it passes" is not green.

**Interview insight:** the bar for senior engineers is recognising that flaky tests are a *system* problem. Individual test fixes are necessary but not sufficient — without quarantine + SLA + auto-deletion, flakes accumulate faster than they're fixed.

### Q8. How do you deflake a test that has a race condition?

**Answer:**

Race conditions are the most common flake cause. Standard pattern: replace timing assumptions with synchronisation.

**Anti-pattern (the flake):**

```python
def test_async_completes():
    start_async_task()
    time.sleep(0.5)               # hope 500 ms is enough
    assert task_completed()
```

Sometimes 500 ms isn't enough (CI runner under load). Sometimes the task completes in 50 ms but `time.sleep` wastes 450 ms anyway.

**Fix 1 — Wait for the condition explicitly:**

```python
def test_async_completes():
    start_async_task()
    wait_until(task_completed, timeout=10)   # poll with backoff

def wait_until(predicate, timeout: float, interval: float = 0.05):
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        if predicate():
            return
        time.sleep(interval)
    raise TimeoutError(f"{predicate.__name__} did not become true within {timeout}s")
```

Fast when fast, tolerant when slow.

**Fix 2 — Synchronise via the system under test:**

```python
def test_message_published():
    consumer = subscribe_to(topic="events")
    publish(event)
    msg = consumer.receive(timeout=5)         # blocks until received or timeout
    assert msg.payload == expected
```

The receive operation provides synchronisation. No sleep needed.

**Fix 3 — Mock the clock:**

If timing is intrinsic (e.g., a 24-hour scheduler), don't use real time:

```python
from freezegun import freeze_time

@freeze_time("2025-04-15 00:00:00")
def test_daily_rollover(scheduler):
    scheduler.start()
    with freeze_time("2025-04-16 00:00:01"):
        scheduler.tick()
    assert scheduler.last_run == datetime(2025, 4, 16)
```

**Fix 4 — Inject the clock as a dependency:**

```python
class Scheduler:
    def __init__(self, clock=time.monotonic):
        self.clock = clock

# Test injects a fake clock
class FakeClock:
    def __init__(self): self.t = 0.0
    def __call__(self): return self.t
    def advance(self, dt): self.t += dt

clock = FakeClock()
scheduler = Scheduler(clock=clock)
clock.advance(60)
```

**General principles:**

- **Never `sleep` for "long enough."** Always poll for the condition.
- **Prefer synchronous APIs at the test boundary.** Make the test wait on a real signal, not a guessed delay.
- **Inject clocks and randomness.** Tests should control the non-determinism, not be victim to it.
- **Increase, then increase, then increase the timeout.** If a test still flakes at 30 s timeout, the bug is real.

### Q9. How do you parallelise tests in a way that scales linearly across runners, and what limits that scaling?

**Answer:**

**Linear scaling target:** N runners → ~N× speedup.

**Setup for linear scaling:**

```yaml
test:
  strategy:
    fail-fast: false
    matrix:
      shard: [1, 2, 3, 4, 5, 6, 7, 8]
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-python@v5
      with: { python-version: "3.12", cache: pip }
    - run: pip install -r requirements.txt
    - run: pytest --splits 8 --group ${{ matrix.shard }} --splitting-algorithm least_duration
```

`pytest-split` uses historical durations (stored in a JSON file) to balance shards.

**What limits scaling:**

**1. Fixed setup cost per shard.**

Each shard does: VM boot (30 s), checkout (10 s), deps install (60 s), test setup (10 s) = ~110 s before any test runs. With 8 shards on a 16-min suite, you get 16/8 + 1.8 = 3.8 min per shard, not 2 min. Past 16 shards the setup dominates.

**Mitigation:** pre-baked runner images, layer caching, smaller deps install (Q4 of `artifact_management`).

**2. Skew (the longest single test or shard).**

If one test takes 5 minutes and the rest take seconds, no parallelism helps — the slowest shard dictates wall-clock.

**Mitigation:** identify slow tests (`pytest --durations=20`), refactor or move to a separate "slow" job.

**3. Shared resources contention.**

If all shards hit the same database, the DB becomes the bottleneck and adding shards adds load without speedup.

**Mitigation:** per-shard databases, in-process databases (SQLite for unit tests), or stub external services.

**4. Cost vs speed.**

8 shards cost 8× the runner-minutes for a ~7× speedup. The marginal cost grows; the marginal benefit shrinks.

**Mitigation:** size parallelism for cost-effectiveness, not maximum speed.

**Sharding algorithm comparison:**

```python
# Synthetic test durations
durations = [10, 8, 7, 6, 5, 5, 4, 3, 2, 2, 1, 1]   # total = 54s

# Round-robin (count-based)
shard_a = [10, 7, 5, 4, 2, 1]   # 29s
shard_b = [8, 6, 5, 3, 2, 1]    # 25s   skew = 4s

# Bin-packing (duration-based)
shard_a = [10, 6, 5, 4, 2]       # 27s
shard_b = [8, 7, 5, 3, 2, 1, 1]  # 27s   skew = 0s
```

**Empirical scaling result:**

Most pipelines achieve ~70-80% of linear speedup at 8-shards, dropping to ~50% at 16 shards. Beyond that, you're paying for marginal returns.

### Q10. How does test impact analysis (TIA) work, and what are its trade-offs vs running everything?

**Answer:**

**TIA** maps each test to the production code it exercises (the test's "footprint"). On a change, only tests whose footprint includes the changed code run.

**Algorithm:**

```python
# Build phase (once, on main)
test_to_files = {}
for test in all_tests:
    with coverage_recording(test) as cov:
        run_test(test)
    test_to_files[test.id] = cov.covered_files()

# Selection phase (per PR)
changed = git_diff_files("main", "HEAD")
relevant = [t for t, files in test_to_files.items() if files & changed]
pytest <relevant>
```

**Real implementations:**

- **Microsoft TIA** — for .NET, integrates with Azure DevOps
- **Launchable** — language-agnostic, ML-augmented
- **Bazel / Pants** — derive footprint from explicit BUILD-file deps
- **Coverage-based custom** — `coverage.py` + custom selector

**What it gives you:**

| Scenario | Without TIA | With TIA |
|----------|-------------|----------|
| Touch one util function used by 5 tests | Run all 5,000 tests | Run 5 tests |
| Touch a core abstraction used by 4,500 tests | Run all 5,000 tests | Run 4,500 tests |
| Touch a config file no test reads | Run all 5,000 tests | Run 0 tests (uh oh) |

**Trade-offs:**

**Pros:**

- Massive speedup for typical changes
- Faster PR feedback → faster merge throughput

**Cons:**

- **Coverage-based TIA misses tests that should fail.** If a test's coverage data is stale, or a behavioural change doesn't change line coverage, the test isn't selected.
- **Indirect effects.** A change to a config file or a build setting may affect tests that don't "cover" that file.
- **Footprint maintenance.** Coverage data must be re-recorded as the codebase evolves; otherwise selection drifts toward stale footprints.
- **Risk of false confidence.** Devs see "0 tests ran, all passed" and merge.

**Mitigation rules:**

1. **Always run the full suite on `main`.** TIA on PRs, full on merge. Catches selection misses before they reach production.
2. **Run on a regression baseline.** Daily nightly full run; if it fails on a commit that passed PR, the gap is documented.
3. **Conservative selection.** Run TIA-selected + a random 10% sample. Random sample finds gaps over time.
4. **Skip TIA for cross-cutting changes.** Changes to test infra, build config, lockfiles → run everything.

**When NOT to use TIA:**

- Test suite < 5 minutes — overhead of TIA selection > savings
- Critical-path systems where coverage gaps are catastrophic
- Heavily integration / E2E weighted suites — coverage doesn't model interactions

**Interview insight:** TIA is a reliability vs speed trade-off. Senior engineers articulate the trade-off and design guardrails (full run on `main`, daily baseline) rather than presenting TIA as pure win.

### Q11. What is mutation testing, and where does it fit in CI?

**Answer:**

**Mutation testing** evaluates *test suite quality* by automatically modifying ("mutating") production code and checking whether tests catch the change. A surviving mutant means the test suite is blind to that change — a coverage gap that line coverage doesn't reveal.

**How it works:**

```python
# Original
def is_adult(age):
    return age >= 18

# Mutants applied automatically:
def is_adult(age): return age > 18         # mutant 1: >= → >
def is_adult(age): return age >= 17        # mutant 2: 18 → 17
def is_adult(age): return age >= 19        # mutant 3: 18 → 19
def is_adult(age): return not (age >= 18)  # mutant 4: negate result
def is_adult(age): return True             # mutant 5: replace body
```

For each mutant, run the test suite. If tests fail → mutant is "killed." If tests pass → mutant "survives" — a real coverage gap.

**Mutation score = killed / total mutants.** A score of 100% means every mutation is caught; in practice 70-85% is realistic for well-tested code.

**Tooling:**

- **Python:** `mutmut`, `cosmic-ray`
- **JavaScript:** `Stryker`
- **Java:** `PIT`
- **Go:** `go-mutesting`
- **C#:** `Stryker.NET`

**Where it fits in CI:**

**Don't put mutation testing in the PR-blocking path.** It is slow — running the entire test suite N times for N mutants means a 5-min suite becomes 5 hours. It also has noise (some mutants are equivalent and unkillable).

**Instead:**

```yaml
# .github/workflows/mutation.yml
name: Mutation testing
on:
  schedule:
    - cron: "0 2 * * 0"      # weekly, Sunday 02:00
  workflow_dispatch:

jobs:
  mutmut:
    runs-on: ubuntu-latest
    timeout-minutes: 360
    steps:
      - uses: actions/checkout@v4
      - run: pip install mutmut pytest
      - run: mutmut run --paths-to-mutate src/
      - run: mutmut html
      - uses: actions/upload-artifact@v4
        with: { name: mutation-report, path: html/ }
```

The weekly report becomes a backlog source — "test our `pricing.py`, mutation score is 42%."

**Interview insight:** mutation testing is a "quality of quality" tool. Mention it when the interviewer asks how you'd measure test effectiveness beyond coverage. Few companies run it; demonstrating awareness shows engineering maturity.

### Q12. How would you reduce the runtime of an end-to-end test suite that takes 90 minutes?

**Answer:**

E2E suites are the worst offenders for slow pipelines. Strategies, in order of impact:

**1. Move tests down the pyramid (highest impact).**

Audit every E2E test. For each, ask: "could this be an integration test or a unit test?" Most can. A 90-min E2E suite often hides 60 min of work that belongs in faster layers.

```python
# E2E (slow): browser → server → DB → assert
def test_user_signup_via_browser():
    browser.click("Sign up")
    browser.fill_form(...)
    browser.submit()
    assert browser.see("Welcome")

# Equivalent integration test (10x faster):
def test_user_signup_api():
    response = client.post("/users", json={...})
    assert response.status_code == 201
    assert User.query.filter_by(email=...).first() is not None
```

E2E should be reserved for: critical user journeys, browser-specific bugs, payment flows.

**2. Parallelise (Q9).**

Shard E2E across N runners. Each shard runs a subset of tests against its own application instance + DB.

```yaml
e2e:
  strategy:
    matrix:
      shard: [1, 2, 3, 4, 5, 6, 7, 8]
  steps:
    - run: docker compose up -d
    - run: playwright test --shard=${{ matrix.shard }}/8
```

90 min / 8 shards ≈ 11 min, plus per-shard setup overhead.

**3. Run only on `main` and nightly (not every PR).**

E2E catches integration regressions, not first-time bugs. Lower-layer tests catch bugs at code-write time. Run E2E on `main` post-merge; alert the team if `main` breaks.

**4. Selective E2E on PRs based on change scope.**

If a PR only touches CSS, skip E2E. If it touches checkout flow, run the checkout E2E suite only.

```yaml
e2e-checkout:
  if: contains(github.event.pull_request.changed_files, 'src/checkout/')
  steps: [..., { run: playwright test tests/e2e/checkout }]
```

**5. Reduce setup time.**

E2E suites often spend 5 min spinning up a fresh environment. Pre-bake docker images, use `tmpfs` for the DB, share long-lived test environments where safe.

**6. Kill the flakes (Q7-Q8).**

A 5% E2E flake rate triples the effective wall-clock (one rerun per 20 tests). Quarantine flaky E2E aggressively.

**7. Consider visual / contract tests instead.**

For UI: Percy / Chromatic visual tests are seconds, not minutes, and catch visual regressions E2E often misses.
For APIs: Pact contract tests verify each side independently — fast, no full integration needed.

**Realistic outcome:**

| Action | Time saved |
|--------|-----------|
| Move 50 trivially-replicable E2E tests down to integration | 30 min |
| Parallelise remaining across 8 shards | 60 min → 8 min |
| Move from PR to post-merge | (PR feedback) → fast |

A typical 90-min E2E ends up as: ~10 min on PR (smoke only) + ~12 min on `main` + nightly full run. The team gets fast feedback and thorough validation, separately.

---

## Advanced

### Q13. Design a flake quarantine system for a large monorepo with 50,000 tests across 80 teams.

**Answer:**

The system must auto-detect flakes, quarantine them with minimal human latency, route to owners, enforce SLAs, and resist accumulation.

**Architecture:**

```
[CI runs on main] --JUnit XML--> [Test Result DB]
                                        |
                                        v
                            [Daily flake analyser job]
                                        |
                                        v
                            [Quarantine config (Git)] -- consumed by --> [PRs/CI]
                                        |
                                        v
                              [Issue tracker (per team)]
                                        |
                                        v
                              [SLA monitor + escalation]
```

**Components:**

**1. Test result ingestion.**

Every CI run on `main` uploads JUnit XML to a long-term store (S3 + Athena, or a SaaS like Buildkite Test Engine).

```yaml
- run: pytest --junit-xml=junit.xml
- uses: actions/upload-artifact@v4
  with: { name: junit-${{ github.run_id }}, path: junit.xml }
- run: aws s3 cp junit.xml s3://test-results/${{ github.repository }}/${{ github.run_id }}.xml
```

Schema captures: test ID, outcome, duration, run ID, commit SHA, branch.

**2. Flake detection (daily job).**

```python
# scheduled daily
from collections import defaultdict

def detect_flakes(runs_window_days=7, min_runs=20, threshold=0.95):
    by_test = defaultdict(list)
    for run in fetch_main_runs(days=runs_window_days):
        for test in run.tests:
            by_test[test.id].append(test.outcome)

    flakes = []
    for test_id, outcomes in by_test.items():
        if len(outcomes) < min_runs:
            continue
        pass_rate = sum(o == "pass" for o in outcomes) / len(outcomes)
        if 0.05 < (1 - pass_rate) and pass_rate < threshold:
            flakes.append((test_id, pass_rate))
    return flakes
```

**3. Auto-quarantine via PR.**

When a flake is detected, open a PR adding the test to a quarantine list:

```yaml
# quarantine.yml — read by pytest plugin
quarantined:
  - id: "tests/api/test_users.py::test_signup_email_normalisation"
    detected_at: "2025-04-15"
    pass_rate: 0.87
    owner: "@my-org/identity-team"
    issue: "https://github.com/my-org/repo/issues/4321"
    expires_at: "2025-05-15"
```

The pytest plugin marks these tests as `xfail(strict=False)` — they run but don't block.

```python
# conftest.py
import yaml
QUARANTINED = {q["id"]: q for q in yaml.safe_load(open("quarantine.yml"))["quarantined"]}

def pytest_collection_modifyitems(items):
    for item in items:
        if item.nodeid in QUARANTINED:
            item.add_marker(pytest.mark.xfail(strict=False, reason="quarantined"))
```

**4. Routing via CODEOWNERS.**

The quarantine PR is auto-assigned to the owning team based on the test path:

```
# .github/CODEOWNERS
/services/identity/   @my-org/identity-team
/services/payments/   @my-org/payments-team
```

The detection job opens the issue with the owning team as assignee.

**5. SLA enforcement.**

A weekly job:

- Closes auto-quarantine entries past `expires_at` by deleting the test from the quarantine list — *and from the test file*
- Files an escalation issue to the team's manager if a quarantine has been open > 30 days
- Posts weekly metrics to the engineering Slack: "12 quarantined, 3 expired this week, 8 new"

**6. Dashboard.**

- Top-10 flakes by failure rate
- Open quarantine entries by team and age
- Trend: quarantined count over time

**Cultural reinforcement:**

- Teams own their tests. Owning teams' on-call rota includes "deflake duty."
- Pipeline never reruns automatically. A flake fails the build; the next push retries naturally.
- "Mark as flaky" is a deliberate act, recorded, time-bound.

**Interview insight:** the question is asking how you operate a system at scale. Naming components (CODEOWNERS routing, expiry-based deletion, weekly Slack metrics) shows you've thought beyond "we'd just add a retry."

### Q14. You're reviewing a PR that adds a `time.sleep(2)` to a test to fix a flake. What do you say in code review?

**Answer:**

**Decline the PR with feedback.** The fix masks the symptom but enshrines the bug. Suggest a proper fix.

**Code review comment:**

> A `time.sleep(2)` makes the test pass on this CI runner today, but it has three problems:
>
> 1. **It still flakes.** The next time CI is loaded, 2 seconds isn't enough. We've seen this exact pattern: `sleep(0.1)` → `sleep(0.5)` → `sleep(2)` → `sleep(10)` over six months.
>
> 2. **It slows the suite linearly.** If 50 tests do this, that's 100 seconds of pure wait per run. At 200 PRs/day that's ~5.5 hours/day of compute and developer wait time.
>
> 3. **It hides what we're synchronising on.** The test is waiting for *something*. What? Whatever it is, polling for that condition explicitly will be both faster (returns as soon as the condition is true) and more correct (no magic timing).
>
> Could we replace this with a `wait_until` poll? Something like:
>
> ```python
> def wait_until(predicate, timeout: float = 10, interval: float = 0.05):
>     deadline = time.monotonic() + timeout
>     while time.monotonic() < deadline:
>         if predicate():
>             return
>         time.sleep(interval)
>     raise TimeoutError(f"{predicate.__name__} did not become true within {timeout}s")
>
> wait_until(lambda: db.get_user(user_id) is not None)
> ```
>
> If you can't identify the condition you're synchronising on, that's worth investigating — the test may be racing with something we don't fully understand.

**Underlying principle to articulate:**

Tests are code. A `sleep(N)` is the test equivalent of `# TODO fix later`. It compounds: future authors copy the pattern, the suite grows slower, and one day the team sleeps the suite into uselessness.

**Compromise positions (if the fix is genuinely urgent):**

- Accept with a tracked TODO and a 30-day SLA
- Quarantine the test instead — fail loudly that it's flaky
- Time-box the underlying race investigation

**Don't accept:** "I added a sleep, the flake went away, ship it." That's how 90-minute test suites are born.

**Interview insight:** the question tests whether you understand technical debt and engineering culture, not just YAML. The right answer is short, principled, and constructive — exactly what good code review looks like.

### Q15. How do you test infrastructure code (Terraform, Kubernetes manifests) in a CI pipeline?

**Answer:**

Infrastructure code isn't tested the same way as application code. There's no "unit test of `aws_s3_bucket`" — but there are useful layers.

**Layer 1 — Static analysis (fast, runs on every PR):**

```yaml
- run: terraform fmt -check -recursive
- run: terraform validate
- uses: aquasecurity/tfsec-action@v1     # security linting
- uses: bridgecrewio/checkov-action@v12  # policy as code
- uses: terraform-linters/setup-tflint@v4
- run: tflint --recursive
```

These catch: syntax errors, deprecated arguments, security misconfigurations (public S3 bucket, missing encryption), policy violations (resource without `tag:Owner`).

For Kubernetes:

```yaml
- run: kubectl --dry-run=client -f manifests/
- uses: kubescape/github-action@main
- uses: stackrox/kube-linter-action@v1
- run: conftest test manifests/    # OPA policies
```

**Layer 2 — Plan inspection (runs on PR, uses real state):**

```yaml
- run: terraform plan -out=tfplan
- run: terraform show -json tfplan > plan.json
- uses: hashicorp/terraform-plan-output@v1
  with: { plan_file: plan.json }
```

Auto-comments the plan on the PR. Reviewers see "this PR will create 1 RDS instance, modify 3 IAM policies, destroy 2 S3 buckets" before approving.

For destructive plans, gate merging:

```yaml
- name: Block destructive change
  run: |
    if jq -e '.resource_changes[] | select(.change.actions[] == "delete")' plan.json; then
      echo "::error::PR includes resource deletion. Add 'allow-destroy' label to proceed."
      exit 1
    fi
```

**Layer 3 — Apply to ephemeral environment (slow, post-merge or nightly):**

```yaml
- run: terraform workspace new pr-${{ github.event.pull_request.number }}
- run: terraform apply -auto-approve
- run: ./tests/integration/run.sh    # actually verify the infra works
- if: always()
  run: terraform destroy -auto-approve
```

Spin up the infrastructure for real, run integration tests against it, tear it down. Caught: race conditions, IAM misconfigurations, networking bugs static analysis missed.

**Layer 4 — Drift detection (continuous, on production):**

```yaml
on:
  schedule: [{ cron: "0 */6 * * *" }]    # every 6 hours
jobs:
  drift:
    steps:
      - run: terraform plan -detailed-exitcode
        continue-on-error: true
        id: plan
      - if: steps.plan.outputs.exitcode == 2
        run: |
          gh issue create --title "Drift detected on prod" --body "$(terraform show)"
```

Exit code 2 = changes pending = something diverged.

**Pitfalls:**

- **State manipulation in tests.** Don't run apply/destroy against your real state. Always use a separate workspace or backend.
- **Cost runaway.** Ephemeral environments must auto-destroy. Tag with TTL; have a sweeper job that destroys anything past TTL.
- **Secrets in plan output.** `terraform plan -json` may leak secrets. Mask or filter before logging.

**Interview insight:** for platform / DevOps roles, expect this question. The tiered model (static → plan → apply → drift) is the modern answer — naming `tfsec`, `OPA/Conftest`, and "ephemeral environment with auto-destroy" demonstrates you've operated this in production.

### Q16. How would you design a CI pipeline for a polyglot monorepo (Go, Python, TypeScript) with 200 services, where you want only affected services to be tested per PR?

**Answer:**

This is a Bazel / Pants / Nx-shaped problem. The pipeline has two phases: (a) determine which services are affected, (b) test only those.

**Phase 1 — Determine affected services.**

**Option A — Path-based (cheap, limited):**

```yaml
- uses: dorny/paths-filter@v3
  id: changes
  with:
    filters: |
      svc-orders:    [services/orders/**, libs/auth/**, libs/db/**]
      svc-payments:  [services/payments/**, libs/auth/**, libs/db/**]
      svc-catalog:   [services/catalog/**, libs/db/**]
```

Maintainable for a few dozen services. Beyond that, the filter file becomes the source of bugs.

**Option B — Build-graph-based (Bazel/Pants/Nx):**

```yaml
- run: bazel query 'rdeps(//..., set('"$(git diff --name-only origin/main | xargs -I{} echo //{} )"'))' --output=label > affected.txt
- run: bazel test $(cat affected.txt)
```

Bazel reads the BUILD files (explicit dep declarations) and computes the precise set of targets affected by a file change. Scales to millions of targets.

**Phase 2 — Run language-appropriate tests for affected services.**

```yaml
jobs:
  detect:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.affected.outputs.services }}
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - id: affected
        run: |
          # Output a JSON array of {service, language}
          echo "services=$(./scripts/affected.sh)" >> $GITHUB_OUTPUT

  test:
    needs: detect
    if: needs.detect.outputs.services != '[]'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect.outputs.services) }}
    steps:
      - uses: actions/checkout@v4
      - name: Setup language
        run: |
          case "${{ matrix.service.language }}" in
            go)  ./scripts/setup-go.sh ;;
            py)  ./scripts/setup-python.sh ;;
            ts)  ./scripts/setup-node.sh ;;
          esac
      - name: Test
        working-directory: services/${{ matrix.service.name }}
        run: make test
```

**Phase 3 — Always run on `main`.**

```yaml
on:
  push:
    branches: [main]
jobs:
  test-all:
    # Bypass affected detection; run everything
    steps: [..., { run: bazel test //... }]
```

**Caching strategy:**

- Per-language toolchain caches (Q4 of `artifact_management`)
- Bazel remote cache (shared across all PRs and `main`) — biggest win for build outputs
- Actions cache for `node_modules`, `go mod`, `pip cache`

**Cross-cutting changes:**

A change to a shared library (`libs/auth`) triggers tests for every service that depends on it. Bazel handles this. Path-based detection requires manual filter maintenance — easy to miss a dep.

**Pitfall:** stale build graph. If a service implicitly depends on something not in its BUILD file (e.g., reads a config from a path), Bazel won't detect the dep and the test won't run on changes to the config. Solution: explicit deps in BUILD files; lint to enforce.

**Pipeline structure:**

```
PR:
  detect-affected → test-affected (matrix) → smoke-affected
  
main (post-merge):
  test-everything → integration-test-everything → publish

nightly:
  test-everything → e2e-everything → security-scan-everything
```

**Interview insight:** monorepo CI is a popular system-design question. Discussing Bazel, build graphs, BUILD files, and remote caches — and contrasting with the simpler path-based approach — shows you understand the trade-off between tooling complexity and pipeline correctness.

### Q17. A pipeline times out at 6 hours and is killed by GitHub Actions. Walk through diagnosis and remediation.

**Answer:**

A 6-hour pipeline is a failure mode, not a configuration issue. The diagnosis maps to "is the pipeline hung, or just slow?"

**Step 1 — Determine: hung or slow?**

Open the run, look at the live job logs. Two patterns:

- **Hung:** logs stop at a specific step, no progress. Process is stuck (deadlock, blocking I/O, network hang).
- **Slow:** logs progress but each step takes too long.

**Step 2 — If hung: identify the hung step.**

```yaml
- run: |
    timeout 600 ./long-step.sh   # fail after 10 min instead of 6 hours
    echo "completed at $(date -Iseconds)"
```

Add `timeout` wrappers to candidate steps. The pipeline now fails fast at the hung step, with a clear "this step exceeded 10 minutes" signal.

**Common hang causes:**

- Network: a `curl` to a dead endpoint with no `--max-time`
- Test waiting on an event that never arrives
- Subprocess holding a lock; parent waiting on it
- Database connection pool exhausted
- Self-hosted runner with no other free runners → job queues forever

**Step 3 — Add observability.**

```yaml
- name: Pre-flight system info
  run: |
    df -h ; free -m ; nproc ; cat /proc/loadavg
- name: Run with PID tree dump on stall
  run: |
    (sleep 1800; pstree -p; ps auxf) &
    DUMP_PID=$!
    ./run.sh
    kill $DUMP_PID
```

A dumped PID tree at minute 30 reveals what process is actually running.

**Step 4 — If slow: profile.**

Add per-step timing:

```yaml
- run: time ./build.sh
- run: time ./test.sh
- run: time ./package.sh
```

The slowest step is the optimisation target. Common culprits:

- Tests not sharded (Q3)
- Dependencies reinstalled each time (Q4)
- Docker layers not cached (Q9 of `artifact_management`)
- E2E tests in PR pipeline (Q12)

**Step 5 — Add a workflow-level timeout.**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30      # fail fast, not at 6 hours
    steps: [...]
```

The default is 360 minutes (6 hours). Always set it to a value that's painful but useful — usually 2-3× expected duration.

**Step 6 — Concurrency hygiene.**

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

Prevents a runaway pipeline from blocking subsequent PR runs.

**Step 7 — If the runner is the problem (self-hosted).**

- Check if the runner host is starved for CPU/memory
- Check if the runner pool is saturated (jobs queueing for hours)
- Implement runner health checks; auto-deregister unhealthy runners

**Long-term fix:**

After diagnosis:

- Document the root cause and the fix in a postmortem
- Add monitoring: alert when pipeline duration p95 exceeds threshold
- Set a SLO: PR pipeline p95 < 15 min, time-out at 30 min
- Periodic CI health review: review longest pipelines monthly

**Interview insight:** the trap question is "how do you fix a 6-hour pipeline?" — candidates dive into solutions. The senior answer is "I don't know yet; let me characterise the failure first." The diagnostic discipline is the differentiator.

### Q18. Design a pipeline-quality program for an org with 30 teams whose CI signal has degraded over time. How do you measure, diagnose, and improve?

**Answer:**

This is an organisational problem, not a tooling problem. Treat it like an SRE engagement.

**Phase 1 — Establish ground truth (4 weeks).**

Define and capture a small set of leading indicators per team:

| Metric | Aggregation |
|--------|-------------|
| PR pipeline duration p50, p95 | Weekly per team |
| Pipeline success rate | Weekly per team |
| Test flake rate | Weekly per team |
| `main` red-time (minutes per week `main` was broken) | Weekly per team |
| Time spent re-running pipelines (clicked "rerun" count) | Weekly per team |

Pull from JUnit XML, GitHub Actions API, audit logs. Land these in a dashboard accessible to all teams.

**Phase 2 — Surface the data (1 week).**

Publish a weekly leaderboard / scorecard:

```
WEEK OF 2025-04-15 — CI HEALTH SCORECARD

Team             | PR p95  | Success  | Flake%  | main red |
─────────────────┼─────────┼──────────┼─────────┼──────────┤
identity         |  6 min  |  99.2%   |  0.8%   |   0 min  |  HEALTHY
payments         | 18 min  |  91%     |  6.2%   |  43 min  |  ATTENTION
catalog          |  4 min  |  99.7%   |  0.4%   |   0 min  |  HEALTHY
search           | 32 min  |  84%     | 12.1%   | 180 min  |  CRITICAL
...
```

Publishing the data is half the win — teams discover they're outliers.

**Phase 3 — Identify systemic vs team-specific issues (2 weeks).**

Look for patterns across teams:

- **Systemic** (most teams have it): slow shared runner pool, slow base image build, monorepo build graph slow. Fix once for everyone.
- **Team-specific** (one team, atypical): their test suite is uncached, they have known flaky E2E, they don't shard. Coach the team.

**Phase 4 — Top-down: fix the systemic stuff (4 weeks).**

Examples of high-leverage systemic fixes:

- Roll out a shared "fast-CI base image" with deps pre-installed (saves 60 s per job × 50,000 jobs/week)
- Stand up a remote build cache (Bazel, sccache) — share toolchain build outputs across teams
- Provision a larger runner pool, or auto-scale ephemeral runners
- Ban `actions-v1` (mutable tag); enforce SHA-pinning org-wide
- Standardise the quarantine system (Q13)

**Phase 5 — Bottom-up: coach the worst teams (ongoing).**

For the 3-5 worst-performing teams:

- Embed a CI/CD specialist for one sprint to diagnose and fix
- Targeted training on their pain (sharding, caching, deflaking)
- Set goals: "PR p95 from 32 min → 12 min by end of next quarter"
- Re-measure; celebrate movement

**Phase 6 — Sustain (ongoing).**

- Quarterly review of metrics with leadership
- "CI health" as a regular topic in engineering all-hands
- Onboarding includes "this is how we measure CI health"
- Decay alarm: alert if any team's metrics degrade > 20% week-over-week

**Cultural shifts to drive:**

- "Don't rerun without diagnosing" — every rerun click should ideally include a comment
- "Flaky tests are bugs, not weather" — quarantine and fix, don't tolerate
- "If it's slow, it's broken" — slow CI hides bugs and burns goodwill
- "CI health is a shared responsibility" — platform team provides infra, teams own their pipelines

**Anti-patterns to avoid:**

- **Team rankings as competition.** It can devolve to "we passed by skipping tests." Pair leaderboard with quality metrics (escapes to prod).
- **Centralising all pipelines.** Removes team ownership; becomes a platform-team bottleneck. Keep ownership distributed; provide standards and tooling.
- **One-shot fix-it weeks.** Improvement reverts within a quarter. Make it ongoing measurement and continuous coaching.

**Interview insight:** for staff/principal interviews, this is a near-certain question domain. Articulating the *programme* (measure → publish → fix systemic → coach specific → sustain) demonstrates you've operated at the org level, not just the repo level. Naming specific systemic levers (shared base image, remote cache, runner pool) makes it concrete.

### Q19. What are golden tests, parity tests and differential tests, and when does numeric or simulation code need them?

**Answer:**

Unit tests check behaviour ("returns a list of three"). Numeric code, simulators and ports need tests that pin **numbers**:

| Test | What it compares | Example |
|---|---|---|
| **Golden (headline-number) test** | Output against a saved, reviewed answer | CI asserts a simulator's baseline result is still exactly `13.94` ms per operation |
| **Parity test** | Two implementations of the same model, on fixed inputs | A JavaScript port must be **bit-exact** with the Python reference (identical floats) |
| **Differential test** | Two implementations on **random** inputs | A Rust kernel and the Python simulator run the same generated workloads; outputs must match |

Practicalities:

- **Fixtures** (the saved inputs and answers) live in git. A broad `.gitignore` rule such as `*.json` can hide a new fixture: the test passes locally and fails in CI with "file not found". Add an explicit exception.
- **Re-blessing** a golden value must be a deliberate, reviewed change with a reason in the commit, never a reflex when the test goes red.
- **Exact vs tolerant:** compare integer and plain arithmetic results exactly; give transcendental functions (`exp`, `log`, `pow`) a tolerance, because language runtimes may round them differently. Keep the floating-point operation order identical across ports, or bit-exactness is lost.
- Add **property-based** tests (Hypothesis, proptest) for invariants such as conservation ("tokens out equal tokens in") and **mutation testing** to check the suite would catch a planted bug.

**Interview insight:** say what a golden test can't tell you: that the saved answer was right in the first place. Pair it with an independent check (a reference implementation, an analytic case) when the golden value is created.

### Q20. Why run end-to-end tests against a production build, and what else can make CI disagree with production?

**Answer:**

A framework's **dev server** differs from what ships: no minification or tree-shaking, different module loading, more permissive error handling, hot reload. Running e2e (Playwright, Cypress) against the dev server tests a different program.

Run e2e against the production build instead:

```ts
// playwright.config.ts (CI branch)
webServer: {
  command: "pnpm build && node --no-experimental-require-module node_modules/next/dist/bin/next start -p 3000",
  url: "http://localhost:3000",
}
```

The extra Node flag in that command is there because CI's Node (20.19+/22.12+) allows `require()` of an ES module, but the hosting platform's function loader did not, so a dependency that worked in CI returned 500 in production. Running CI under the stricter rule turned that class of bug into a failing PR.

Other CI-vs-production gaps to close or check after deploy:

- **Runtime version** (pin it in both places).
- **Environment variables** that exist in production only (or are empty locally because they are write-only).
- **Database state**: CI migrates a fresh database; production may be unmigrated or have real data that exercises different code paths.
- **File system and network**: read-only file systems, no outbound access, cold starts.

Whatever can't be made identical needs a **post-deploy smoke check** against the live system.
