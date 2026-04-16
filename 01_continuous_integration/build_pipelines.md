# Build Pipelines — Interview Questions

**Subject:** CI/CD
**Topic:** Pipeline Stages, Triggers, Build Agents, Caching, Parallelism
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is a CI build pipeline, and what are its typical stages?

**Answer:**

A **CI build pipeline** is an automated sequence of steps that validates a code change after it lands in source control. The pipeline's job is to answer one question as fast as possible: *is this change safe to integrate?*

**Typical stages (in order):**

1. **Checkout** — clone the repository at the commit under test
2. **Setup / Install** — install toolchains and dependencies
3. **Lint / Static analysis** — style, type checking, linting
4. **Build / Compile** — produce artefacts (binaries, wheels, images)
5. **Unit tests** — fast, hermetic tests run in parallel
6. **Integration tests** — exercise real databases, queues, services
7. **Package** — bundle artefacts for downstream stages
8. **Publish** — upload to artefact repository with provenance

**GitHub Actions example:**

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - run: pip install -r requirements.txt
      - run: ruff check .
      - run: mypy src/
      - run: pytest --cov=src --junit-xml=junit.xml
      - uses: actions/upload-artifact@v4
        with:
          name: junit
          path: junit.xml
```

**Design principle:** order stages so the cheapest, most discriminating checks run first. Lint failures should cancel the pipeline before you wait for integration tests.

### Q2. What triggers typically start a CI pipeline, and what are they used for?

**Answer:**

Different triggers serve different purposes:

| Trigger | Purpose |
|---------|---------|
| **push** | Run on every commit to a branch (especially `main`) |
| **pull_request** | Validate proposed changes before merge |
| **schedule (cron)** | Nightly runs, dependency updates, security scans |
| **manual / workflow_dispatch** | On-demand runs (deploys, chaos tests) |
| **tag** | Release builds when a version tag is pushed |
| **webhook / repository_dispatch** | Cross-repository triggers, external events |
| **workflow_run** | Chain one workflow after another completes |

**GitHub Actions example covering several:**

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 3 * * *"     # nightly at 03:00 UTC
  workflow_dispatch:
    inputs:
      environment:
        required: true
        type: choice
        options: [staging, production]
  push:
    tags: ["v*.*.*"]
```

**Interview insight:** candidates who mention `schedule` for nightly dependency audits and `workflow_dispatch` for production deploys show they've run real pipelines, not just demo projects.

### Q3. What is a build agent (or runner), and what's the difference between shared and self-hosted agents?

**Answer:**

A **build agent** (or **runner**) is the machine that executes pipeline steps. Agents can be:

- **Shared / SaaS-hosted** — e.g., GitHub-hosted runners, GitLab SaaS runners, CircleCI cloud executors. Fresh VM per job.
- **Self-hosted** — machines you manage, inside your network, often in Kubernetes or on dedicated hardware.

**Trade-offs:**

| Aspect | Shared runners | Self-hosted runners |
|--------|----------------|---------------------|
| Setup cost | Zero | Requires provisioning, maintenance |
| Isolation | Strong (fresh VM) | Depends on your setup |
| Access to private resources | Limited (needs VPN tunnels, bastions) | Native (in-VPC) |
| Cost at scale | Expensive per-minute | Cheaper if utilised |
| Hardware choice | Limited | Any CPU / GPU / RAM |
| Supply chain risk | Trust the provider | Your responsibility |

**When self-hosted is required:**

- Workloads that need GPUs, FPGAs, or specific hardware
- Access to on-prem databases or staging environments
- Workloads too large for standard runners (ML training, hardware simulation)
- Compliance requiring on-prem execution

**GitHub Actions runner selection:**

```yaml
jobs:
  # SaaS
  test:
    runs-on: ubuntu-latest
  # self-hosted with label
  gpu-train:
    runs-on: [self-hosted, linux, gpu, a100]
```

**Security caveat:** never run self-hosted runners for public repos without ephemeral, sandboxed containers. A malicious PR can execute arbitrary code on your runner.

### Q4. Why is caching important in CI, and what should you cache?

**Answer:**

Caching turns a 10-minute pipeline into a 2-minute pipeline by avoiding repeated downloads and recomputation. Every CI pipeline should cache:

**What to cache:**

1. **Package manager caches** — `~/.cache/pip`, `~/.m2`, `node_modules`, `~/.gradle/caches`
2. **Compiler output** — `ccache`, `sccache`, Bazel remote cache, Gradle build cache
3. **Docker layers** — via BuildKit cache mounts or a remote cache
4. **Test dependencies** — downloaded test fixtures, sample data

**What NOT to cache:**

- Build outputs for release artefacts (they must be reproducible)
- Anything containing secrets or user data
- Lock file outputs that would mask dependency drift

**GitHub Actions cache example:**

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.cache/pip
      ~/.cache/pre-commit
    key: ${{ runner.os }}-pip-${{ hashFiles('requirements*.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

**Cache key principle:**

- **Exact key** — includes the hash of the lock file; only matches when deps are identical
- **Restore keys** — prefix match for partial reuse (e.g., install only new deps)

**Pitfall:** unbounded cache growth. Most providers evict on LRU after a size limit, but specifying a scope (branch, ref) prevents `main`'s cache from being overwritten by an experimental branch's giant node_modules.

### Q5. What does "failing fast" mean in a CI pipeline, and how do you implement it?

**Answer:**

**Failing fast** means detecting problems at the earliest possible stage, so the pipeline terminates before burning more time and compute on a doomed run.

**Implementation techniques:**

1. **Order stages cheapest-first** — lint before unit tests before integration tests
2. **`fail-fast: true` in matrix builds** — cancel siblings on first failure
3. **Required checks at PR-open time** — lint runs before expensive tests
4. **Pre-commit hooks** — catch failures before they hit CI at all
5. **Cancel-in-progress for superseded commits** — stop old runs when a new commit lands

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    strategy:
      fail-fast: true
      matrix:
        python: ["3.10", "3.11", "3.12"]
```

**Contrast with "fail slow" (useful for release validation):**

```yaml
strategy:
  fail-fast: false   # run all matrix entries to collect full failure info
```

You want fail-fast for PR feedback (save compute) and fail-slow for nightly runs (gather all flakes in one pass).

### Q6. What is the difference between CI, continuous delivery, and continuous deployment?

**Answer:**

| Concept | Definition | Human gate? |
|---------|------------|-------------|
| **Continuous Integration** | Every change is automatically built and tested against the mainline. | N/A (no deploy) |
| **Continuous Delivery** | Every validated change is automatically promoted through non-prod environments and *could* be deployed to prod at any time. | Yes — a human clicks "deploy" |
| **Continuous Deployment** | Every validated change is automatically deployed to production. | No |

**Examples:**

- **CI only:** Green tests on `main`, but release is a separate manual process. Most regulated industries.
- **CD (delivery):** `main` is always deployable to prod; release manager approves production. Typical for enterprise SaaS.
- **CD (deployment):** Every merge reaches prod within minutes (behind feature flags). Common at Google, Netflix, Etsy.

**Interview insight:** interviewers sometimes use "CD" ambiguously. Ask which one they mean, or define both and pick the context appropriate to the company you're interviewing at.

---

## Intermediate

### Q7. How do you parallelise tests in CI, and what are the trade-offs?

**Answer:**

Test parallelisation strategies fall into two categories:

**1. Within-process parallelism** (multiple workers in one runner):

```yaml
- run: pytest -n auto --dist loadfile    # pytest-xdist
```

Works for hermetic unit tests but breaks if tests share global state (databases, ports, temp dirs).

**2. Across-runner parallelism** (sharding):

```yaml
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4, 5, 6, 7, 8]
    steps:
      - run: pytest --shard=${{ matrix.shard }}/8
```

Each shard gets 1/N of the test files. Needs a splitting strategy that balances duration.

**Splitting strategies (best to worst):**

1. **By historical duration** — record each test's runtime, assign to shards to minimise max-duration (bin-packing). CircleCI and Buildkite provide this built-in.
2. **By count** — equal number of files per shard. Simple, but straggler tests create skew.
3. **By hash** — deterministic random split. Reasonable without history.

**Trade-offs:**

| Aspect | Within-process | Across-runner |
|--------|----------------|---------------|
| Startup cost | One (setup once) | N (setup per runner) |
| Isolation | Process-local | Full VM |
| Cost | 1 runner | N runners |
| Max speedup | CPU cores on one runner | Unbounded (add more shards) |

Most pipelines use **both**: shard across 8 runners, each running `pytest -n 4`.

**Interview insight:** mention that parallelism has a sweet spot. Beyond it, the per-shard setup cost dominates and total wall-clock time stops decreasing.

### Q8. Design a caching strategy for a monorepo with Go, Node.js, and Python services.

**Answer:**

A monorepo has two caching concerns: (a) per-language toolchain/package caches and (b) build output caching across services.

**Per-language caches:**

```yaml
- uses: actions/cache@v4
  with:
    path: ~/go/pkg/mod
    key: go-${{ hashFiles('**/go.sum') }}

- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('**/package-lock.json') }}

- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: pip-${{ hashFiles('**/requirements.txt', '**/poetry.lock') }}
```

**Build output cache (harder, and more valuable):**

Use a build tool with content-addressable caching: Bazel, Pants, Nx, Turborepo, or Gradle. Build outputs are keyed by `hash(inputs)`, so if nothing changed, you fetch from the cache.

```yaml
- uses: bazelbuild/setup-bazelisk@v3
- run: bazel build //... --remote_cache=grpcs://cache.example.com
```

**Layered strategy:**

1. **L0 — runner tmpfs:** `node_modules`, `dist/` within a single job
2. **L1 — CI provider cache:** per-branch, hashed by lockfile
3. **L2 — remote build cache:** shared across all branches and all users, keyed by input hash

**Cache key tips:**

- Include language version in the key (`go-1.22-<hash>`)
- Include OS (`ubuntu-22.04-pip-<hash>`)
- Use restore keys for graceful fallback

**Pitfall:** a naive `hashFiles('**/*.lock')` traverses `node_modules` on cache miss and takes forever. Always constrain globs to source directories.

### Q9. How should you structure a pipeline for fast PR feedback vs. thorough main-branch validation?

**Answer:**

PRs need speed — developers context-switch if feedback takes longer than a coffee break. `main` needs thoroughness — a broken `main` blocks everyone.

**PR pipeline (fast, ~5 min):**

- Lint (parallel with typecheck)
- Unit tests (parallel shards)
- Build (no publish)
- Smoke integration test (one happy path)

**Main pipeline (thorough, ~30 min):**

- Everything in PR pipeline
- Full integration tests
- End-to-end tests
- Performance benchmarks
- Security scans (SAST, SCA, container scan)
- Publish artefacts with provenance

**GitHub Actions two-workflow pattern:**

```yaml
# .github/workflows/pr.yml
on:
  pull_request:
jobs:
  fast-checks:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - run: make lint test-unit

# .github/workflows/main.yml
on:
  push:
    branches: [main]
jobs:
  thorough-checks:
    runs-on: ubuntu-latest
    timeout-minutes: 45
    steps:
      - uses: actions/checkout@v4
      - run: make lint test-unit test-integration test-e2e bench scan publish
```

**Important:** required status checks on PR merge should *not* include the slow end-to-end tests. If E2E is flaky, it blocks merges. Run E2E on `main` and alert on failure — `main` staying green is the team's shared responsibility.

**Interview insight:** mention "mergify"-style auto-merge after required checks pass. Discuss how this pattern plus feature flags enables trunk-based development with high throughput.

### Q10. What is a build matrix, and when would you use one?

**Answer:**

A **build matrix** runs the same job with different parameter combinations — different OS, language version, or dependency version.

**Example — testing a library across Python versions:**

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        python: ["3.10", "3.11", "3.12"]
        include:
          - os: ubuntu-latest
            python: "3.13-dev"
        exclude:
          - os: windows-latest
            python: "3.10"
    steps:
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python }}
      - run: pytest
```

This generates 3 × 3 = 9 jobs minus excludes plus includes = 9 jobs.

**When to use matrices:**

- Libraries supporting multiple Python/Node/Go versions
- Cross-platform software (Linux + macOS + Windows)
- Hardware variations (x86_64 + arm64)
- Database compatibility (Postgres 14, 15, 16)

**When NOT to use matrices:**

- Application code you deploy to a single runtime — testing against 4 Python versions wastes compute
- Things you can express with a single test parameterisation (`pytest.mark.parametrize`)

**Matrix patterns worth knowing:**

- **fail-fast: false** — keep running to see all failures
- **include** — add specific combinations not in the cross product
- **exclude** — remove combinations that aren't supported
- **max-parallel** — throttle to avoid runner starvation

### Q11. How do you make a pipeline reproducible?

**Answer:**

A **reproducible pipeline** produces bit-identical (or semantically identical) outputs given the same inputs. This matters for supply chain security and for debugging "works on my machine" failures.

**Sources of non-reproducibility, and fixes:**

| Source | Fix |
|--------|-----|
| Floating language/tool versions | Pin in `.tool-versions`, `.python-version`, `Dockerfile`, `setup-node@v4 with: node-version-file` |
| Floating dependencies | Lock files (`poetry.lock`, `package-lock.json`, `go.sum`, `Cargo.lock`) — commit them |
| Floating base images | Pin by digest, not tag: `python@sha256:abc...` |
| Timestamps baked into artefacts | `SOURCE_DATE_EPOCH`, `reproducible-jar`, `--mtime` on tar |
| Random test order | Record seed, make reproducible (`pytest --randomly-seed=N`) |
| Network flakiness | Retry plus mirror/proxy (nexus, Artifactory) |
| CPU-dependent outputs | Fixed compiler flags, `-march=x86-64-v2` |

**Dockerfile reproducibility example:**

```dockerfile
# BAD
FROM python:3.12

# GOOD
FROM python@sha256:2f7a2f05e56a81db82bf9e8c4ac6c8cbf6a0a9e9f2e8b6b4fcaf0c8b5f72aebd

COPY requirements.txt .
RUN pip install --no-deps --require-hashes -r requirements.txt
```

**Verifying reproducibility:**

Build twice on different runners, diff the artefacts:

```bash
diffoscope build1/app.tar build2/app.tar
```

**Why this matters:** reproducibility is the foundation of SLSA Level 3+. Without it, you cannot independently verify what the build pipeline produced.

### Q12. What is a pipeline as code, and what are its benefits over configured pipelines?

**Answer:**

**Pipeline as code** means the pipeline definition lives in the repository, versioned alongside the code it builds. Contrast with older systems where you configured jobs via a web UI (classic Jenkins freestyle, TeamCity GUI).

**Benefits:**

1. **Version control** — pipeline changes go through PRs and code review
2. **Reproducibility** — check out an old commit and get the old pipeline
3. **Refactorability** — grep, rename, extract shared templates
4. **Auditability** — who changed the deploy step, when, and why
5. **Environment parity** — every branch can run its own pipeline iteration

**Examples:**

```groovy
// Jenkinsfile
pipeline {
    agent any
    stages {
        stage('Build') { steps { sh 'make build' } }
        stage('Test')  { steps { sh 'make test'  } }
    }
}
```

```yaml
# .github/workflows/ci.yml
jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - run: make build
      - run: make test
```

```yaml
# .gitlab-ci.yml
stages: [build, test]
build:
  stage: build
  script: make build
test:
  stage: test
  script: make test
```

**Trade-offs:**

- UI-configured pipelines are easier for non-engineers to tweak
- Pipeline-as-code can grow unwieldy — extract reusable workflows, composite actions, or shared libraries

**Interview insight:** treat pipeline code as production code. Lint it (`actionlint`, `yamllint`), test it (run workflows in a test repo), and refactor aggressively.

---

## Advanced

### Q13. Design a CI pipeline for a monorepo that only rebuilds and retests changed services.

**Answer:**

For a monorepo with 50 services, building everything on every PR is wasteful. Selective CI is essential.

**Strategy: compute an affected-set from the git diff.**

```yaml
name: Monorepo CI
on: [pull_request]

jobs:
  compute-changes:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            api:       services/api/**
            worker:    services/worker/**
            frontend:  services/frontend/**
            shared:    libs/shared/**

  build-changed:
    needs: compute-changes
    if: ${{ needs.compute-changes.outputs.services != '[]' }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: ${{ fromJson(needs.compute-changes.outputs.services) }}
    steps:
      - uses: actions/checkout@v4
      - run: make -C services/${{ matrix.service }} ci
```

**Handling shared dependencies:**

If `libs/shared` changed, everything downstream must rebuild. Encode the dependency graph:

```python
# scripts/affected.py
import json
import subprocess

DEPS = {
    "api":      ["libs/shared"],
    "worker":   ["libs/shared", "libs/queue"],
    "frontend": ["libs/ui"],
}

changed_paths = subprocess.check_output(
    ["git", "diff", "--name-only", "origin/main...HEAD"]
).decode().splitlines()

affected = set()
for service, deps in DEPS.items():
    if any(p.startswith(f"services/{service}/") for p in changed_paths):
        affected.add(service)
    if any(any(p.startswith(d + "/") for p in changed_paths) for d in deps):
        affected.add(service)

print(json.dumps(sorted(affected)))
```

**For large monorepos, use a proper build system:**

- **Bazel** — content-addressable targets, `bazel query 'rdeps(//..., //libs/shared)'`
- **Nx** — JS/TS monorepos, `nx affected:test`
- **Turborepo** — lightweight, good for Node.js
- **Pants** — Python-heavy monorepos

These systems track fine-grained dependencies and cache at the target level. A 50-service monorepo can test in 2 minutes when nothing changed, because every target hits the cache.

**Interview insight:** this is a staple of platform engineering interviews. Mention that affected-set is also how you scale code review (`CODEOWNERS` patterns match changed paths) and deployment (deploy only changed services).

### Q14. How do you debug a pipeline failure that reproduces only in CI, never locally?

**Answer:**

"Works on my machine" is the archetypal pipeline bug. A systematic approach:

**1. Diff the environments.**

```bash
# on developer machine
env | sort > local.env
pip freeze > local.deps
uname -a > local.uname

# in CI, as a debug step
env | sort > ci.env
pip freeze > ci.deps
uname -a > ci.uname
```

Upload both as artefacts and diff. Common culprits: `PATH`, `LANG`, `TZ`, `HOME`, `HTTP_PROXY`.

**2. Pin the same toolchain versions.**

Match your local Python/Node/Go/JDK version exactly to CI. Tools like `asdf` or `mise` read `.tool-versions` from the repo.

**3. Reproduce the CI runner image.**

GitHub Actions publishes its runner images. Pull one:

```bash
docker run -it --rm \
  -v $(pwd):/workspace -w /workspace \
  ghcr.io/actions/runner-images:ubuntu-22.04 bash
```

Or use `act` to run GitHub Actions workflows locally:

```bash
act pull_request
```

**4. SSH into a failed runner.**

For GitHub Actions, `mxschmitt/action-tmate` opens a tmate session mid-run (treat as disposable, never on production repos):

```yaml
- name: Debug
  if: failure()
  uses: mxschmitt/action-tmate@v3
  timeout-minutes: 15
```

Jenkins, CircleCI, and Buildkite all offer equivalent "rerun with SSH" features.

**5. Rule out flakiness.**

Rerun the job. If it passes, it's a flake — quarantine the test and open a ticket. If it fails again, it's deterministic.

**6. Rule out concurrency.**

CI runners often have 2-4 cores. Your laptop has 16. Tests that pass locally because of timing may race in CI. Run locally with `taskset -c 0,1 pytest` to limit to 2 cores.

**7. Check for filesystem case sensitivity.**

macOS is case-insensitive by default. Linux CI is case-sensitive. An import of `Utils.py` that works locally will fail in CI if the actual filename is `utils.py`.

**8. Inspect timestamps and ordering.**

Some failures appear only because CI creates files in a specific order (alphabetical) that differs from your local workflow. Tests that depend on iteration order are classic.

**Interview insight:** candidates who mention `act`, `tmate`, and case sensitivity have debugged real pipelines.

### Q15. How do you design a pipeline that supports hermetic, reproducible builds for supply-chain security?

**Answer:**

A **hermetic build** produces identical outputs given identical inputs, with no access to the network and no dependency on the machine's ambient state. This is the gold standard for SLSA Level 4.

**Design principles:**

1. **All inputs are declared.** Source code, toolchains, dependencies — everything is pinned by content hash.
2. **Build runs in a sealed environment.** No network, no access to /home, no `/tmp` persistence.
3. **Outputs include provenance.** A signed attestation binds outputs to inputs and builder identity.

**Implementation options:**

| Tool | Approach |
|------|----------|
| **Bazel** | Sandboxed actions, hermetic toolchains, remote execution |
| **Nix** | Content-addressed derivations, no network during build |
| **Buck2** | Meta's Bazel successor, similar sandboxing |
| **`docker build --network=none`** | Poor-man's hermeticity for one image |

**Nix example:**

```nix
{ pkgs ? import (fetchTarball {
    url = "https://github.com/NixOS/nixpkgs/archive/24.05.tar.gz";
    sha256 = "sha256:...";
  }) {}
}:
pkgs.stdenv.mkDerivation {
  name = "myapp-1.0.0";
  src = ./.;
  buildInputs = [ pkgs.go_1_22 ];
  buildPhase = ''
    export GOFLAGS="-mod=vendor"
    go build -trimpath -ldflags='-buildid= -s -w' -o $out/bin/myapp ./cmd/myapp
  '';
}
```

`-trimpath` strips build paths. `-buildid=` removes the random build ID. `-s -w` strips symbol tables. Build twice: identical hashes.

**GitHub Actions provenance generation:**

```yaml
jobs:
  build:
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: make release
      - name: Generate SLSA provenance
        uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.0.0
        with:
          base64-subjects: ${{ steps.hash.outputs.hashes }}
```

**Interview insight:** most interviewers don't expect a full Nix derivation. They want you to articulate:
- inputs must be pinned by hash
- the build environment must be controlled
- outputs must be signed and attestable
- you should be able to rebuild from scratch and get the same bits

### Q16. How do you handle secret injection in pipelines safely?

**Answer:**

Secrets — API keys, signing keys, database passwords — must reach your pipeline without leaking into logs, artefacts, or forked PRs.

**Principles:**

1. **Never commit secrets.** Not even encrypted, unless you use a dedicated tool (SOPS, git-crypt, sealed-secrets).
2. **Inject at runtime from a secret store.** CI provider vaults, HashiCorp Vault, AWS Secrets Manager, 1Password.
3. **Prefer OIDC federation to long-lived keys.** GitHub's OIDC token is trusted by AWS/GCP/Azure; no static AWS keys in CI.
4. **Mask secrets in logs.** CI providers do this for configured secrets, but your own echoes can leak them.
5. **Restrict secret scope.** Don't give every workflow every secret.

**OIDC to AWS (no static credentials):**

```yaml
jobs:
  deploy:
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deploy
          aws-region: us-east-1
      - run: aws s3 cp dist/ s3://bucket/ --recursive
```

The GitHub runner exchanges its OIDC token for temporary AWS credentials. No `AWS_ACCESS_KEY_ID` stored anywhere.

**Masking in logs:**

```yaml
- name: Get token
  id: token
  run: |
    TOKEN=$(curl -s https://token.example.com)
    echo "::add-mask::$TOKEN"
    echo "token=$TOKEN" >> "$GITHUB_OUTPUT"
```

**Fork PR safety:**

Never expose secrets to workflows triggered by `pull_request` from forks. Use `pull_request_target` carefully — it runs with the base branch's context and *does* have secrets, so you must not check out and execute PR code under this trigger without sandboxing.

**Secret rotation:**

Build secret rotation into the pipeline. A quarterly automated job rotates all keys, updates the secret store, and redeploys. Catches integrations that silently cached an old key.

**Interview insight:** OIDC federation is the 2024+ answer. Candidates still suggesting long-lived `AWS_SECRET_ACCESS_KEY` in CI look out of date.

### Q17. How do you deal with pipeline cost at scale?

**Answer:**

CI at scale can cost more than production. At 1000 developers with 10 CI runs per day at 10 minutes per run on a $0.008/minute runner, you're spending $800/day, $290k/year. Cost must be engineered.

**Levers, in order of impact:**

**1. Compute less (selective CI).**

Affected-set detection is the biggest lever — 70% of PRs may touch one service, so building 50 is waste. Bazel/Nx/Turborepo pay off quickly.

**2. Cache aggressively.**

A remote build cache with high hit-rate turns 10-minute builds into 1-minute builds. Measure cache hit rate and invest in raising it.

**3. Right-size runners.**

Don't run lint on a 16-core runner. Use the smallest class that fits. Most CI providers price linearly in resources.

**4. Self-host for steady-state.**

At scale, self-hosted Kubernetes runners (ARC for GitHub Actions, Gitlab Runner Kubernetes executor) are 3-10x cheaper than SaaS. Amortise across all teams.

**5. Spot / preemptible instances.**

CI jobs are stateless and restartable — perfect for spot. Retry logic handles preemption.

**6. Parallelism ceilings.**

Sharding to 64 runners cuts wall-clock time but multiplies cost. At some point, you pay $64 for the job that used to cost $10 in 10x the wall-clock. Optimise for developer time vs. dollar cost.

**7. Cancel superseded runs.**

`concurrency.cancel-in-progress: true` on PRs. If a developer pushes a fix 30 seconds after a typo, don't finish the old run.

**8. Move expensive checks off the hot path.**

E2E, soak, benchmark — run nightly on `main`, not on every PR.

**9. Report and chargeback.**

Teams that see a monthly CI bill optimise their own pipelines. A dashboard showing per-team, per-pipeline cost changes behaviour faster than any technical intervention.

**Example measurement script:**

```python
# gh api to pull workflow run durations
import subprocess, json, collections

runs = json.loads(subprocess.check_output([
    "gh", "api", "repos/org/repo/actions/runs",
    "--paginate", "-X", "GET",
    "-f", "per_page=100",
]))

cost_per_min = 0.008
per_workflow = collections.defaultdict(float)
for r in runs["workflow_runs"]:
    mins = (r["run_duration_ms"] or 0) / 60000
    per_workflow[r["name"]] += mins * cost_per_min

for name, cost in sorted(per_workflow.items(), key=lambda x: -x[1]):
    print(f"${cost:8.2f}  {name}")
```

**Interview insight:** talk about it as an optimisation problem with a clear metric — cost per merged PR, or cost per engineer-day. Teams that can't measure it can't manage it.

### Q18. What is the "pipeline of pipelines" pattern, and when would you use it?

**Answer:**

A **pipeline of pipelines** (also called **orchestrator pipeline** or **parent pipeline**) is a top-level pipeline that triggers and coordinates child pipelines. Used when:

- A release spans multiple repositories
- Different teams own different stages (CI in one pipeline, deploy in another)
- You need environment progression across long-running stages (bake times, manual gates)

**GitLab's multi-project pipeline:**

```yaml
# parent .gitlab-ci.yml
stages: [build, e2e, promote]

build:
  trigger:
    project: org/backend
    branch: main
    strategy: depend   # wait for child to finish

e2e:
  trigger:
    include:
      - project: org/e2e-suite
        file: .gitlab-ci.yml
    strategy: depend
  needs: [build]

promote:
  trigger:
    project: org/infra
    branch: main
  needs: [e2e]
```

**GitHub Actions equivalent with `repository_dispatch`:**

```yaml
# orchestrator
jobs:
  trigger-deploy:
    runs-on: ubuntu-latest
    steps:
      - run: |
          gh api repos/org/infra/dispatches \
            -f event_type=deploy \
            -f client_payload[sha]=${{ github.sha }}
```

**Argo Workflows / Tekton:**

For Kubernetes-native pipelines, Argo and Tekton model pipelines as DAGs of tasks across namespaces. Scales to thousands of concurrent steps with backpressure and retry semantics that GitHub Actions lacks.

**Trade-offs:**

| Aspect | Single pipeline | Pipeline of pipelines |
|--------|-----------------|----------------------|
| Debuggability | Easier (one place) | Harder (follow the chain) |
| Ownership | One team | Multiple teams |
| Duration | Bounded by CI timeout | Can span hours/days |
| Bake time / soak | Hard | Natural |

**When not to use:** for a simple service with CI + CD, one pipeline is clearer. Reach for pipeline-of-pipelines only when you have genuine multi-repo or multi-team coordination.

### Q19. How do you handle long-running tests or build steps that exceed CI timeouts?

**Answer:**

Most CI providers cap jobs at 6 hours (GitHub Actions) or less. Long-running steps need architectural solutions.

**Strategy 1: Split the work.**

Break the long step into shards. Run in parallel. This is the same lesson as test parallelisation — scale out, not up.

**Strategy 2: Offload to a specialised runner.**

Hardware simulation, ML training, fuzzing — move to a dedicated self-hosted runner with longer timeouts and more resources.

```yaml
jobs:
  fuzz:
    runs-on: [self-hosted, large, fuzzing]
    timeout-minutes: 1440   # 24 hours
    steps:
      - run: ./fuzz.sh --duration=12h
```

**Strategy 3: Decouple via queue.**

The CI job enqueues a task; a separate worker processes it; the CI job polls for results or checks a status endpoint.

```yaml
- name: Submit long-running job
  id: submit
  run: |
    JOB_ID=$(curl -X POST https://ml-cluster.internal/submit \
      -d @spec.json | jq -r .id)
    echo "job_id=$JOB_ID" >> "$GITHUB_OUTPUT"

- name: Wait for completion (with polling, timeout 4h)
  run: |
    for i in {1..240}; do
      STATUS=$(curl -s https://ml-cluster.internal/jobs/${{ steps.submit.outputs.job_id }} | jq -r .status)
      [[ "$STATUS" == "SUCCESS" ]] && exit 0
      [[ "$STATUS" == "FAILED"  ]] && exit 1
      sleep 60
    done
    exit 2
```

**Strategy 4: Nightly / scheduled runs.**

Not every check belongs on every commit. Nightly-only:

- Full performance regression suite
- 24-hour soak test
- Fuzzing campaigns
- Mutation testing

**Strategy 5: Incremental / resumable work.**

For fuzzing and property-based testing, store a corpus between runs. Each CI run adds an hour to an ongoing campaign rather than starting from scratch.

**Interview insight:** naming specific long-running workloads (fuzzing, ML training, hardware simulation) and their dedicated patterns — queues, nightly runs, persistent corpora — shows real experience. Candidates who only know "increase the timeout" haven't hit the wall.

### Q20. How do you test pipeline code itself?

**Answer:**

Pipeline code is production code. Breaking `main.yml` is as bad as breaking production deploy code.

**Techniques, from cheap to thorough:**

**1. Lint.**

```bash
actionlint .github/workflows/*.yml
yamllint .github/workflows/
```

`actionlint` understands GitHub Actions semantics — catches `uses` typos, undefined matrix keys, unreferenced secrets.

**2. Schema validation.**

Jenkins has a `declarative-linter` endpoint; GitLab has `CI Lint`; CircleCI has `circleci config validate`. All take a pipeline file and return syntax/semantic errors.

**3. Local execution.**

`act` runs GitHub Actions locally in Docker:

```bash
act pull_request -j test
```

GitLab Runner can execute `.gitlab-ci.yml` locally:

```bash
gitlab-runner exec docker test
```

**4. Test in a sandbox repo.**

Fork or branch, point the workflow at a test service, verify end-to-end behaviour before merging.

**5. Integration test the pipeline.**

For complex reusable workflows and composite actions, write a dedicated test workflow that exercises every input combination:

```yaml
# .github/workflows/test-reusable.yml
name: Test reusable workflow
on: pull_request

jobs:
  test-happy-path:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
    secrets: inherit

  test-invalid-env:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: invalid-name
    continue-on-error: true
```

**6. Staged rollout of pipeline changes.**

For major refactors — switching from Jenkins to GitHub Actions, or adopting Bazel — run both pipelines in parallel for a week. Compare outcomes (success rate, duration, flakes). Only decommission the old one when the new is at parity.

**7. Monitor pipeline health.**

Instrument CI metrics: success rate, duration P50/P95, queue time, flake rate. Alert when they degrade. Treat the pipeline like a service with an SLO.

**Interview insight:** senior candidates should talk about pipelines as a *product* with users (developers), SLIs (success rate, duration), and SLOs (P95 under 10 minutes). This reframes CI/CD from "plumbing" to "platform engineering".

### Q21. How do you design a CI pipeline that works both online (SaaS CI) and air-gapped (classified / regulated environments)?

**Answer:**

Some environments prohibit outbound internet — defence, finance, healthcare. The pipeline must run entirely inside a private network while preserving the developer experience.

**Design requirements:**

1. **Internal mirrors for all public registries.** pip, npm, maven, Go modules, Docker registries — run Artifactory or Nexus in-house, pre-populated and monitored.
2. **All dependencies pinned by content hash.** No floating versions that could introduce unreviewed code.
3. **Build agents inside the network.** No SaaS runners. Often self-hosted GitLab, Jenkins, or GitHub Enterprise.
4. **Offline-capable tools.** Any CI image must work without network — `apt`/`pip`/`go get` must point at internal mirrors.
5. **Artefact signing with internal PKI.** Sigstore's keyless flow uses public Fulcio/Rekor — air-gapped needs self-hosted equivalents or traditional GPG/x509.
6. **Audit trails to internal SIEM.** All pipeline events — who triggered what, which artefacts were built — stream to Splunk/ELK inside the enclave.

**Dual-pipeline architecture:**

```
external dev machine --> [DMZ proxy] --> internal git --> CI --> artefact repo --> deploy

                        ^ one-way data diode
                        ^ scanning + approval gate
```

Developers work on copies outside the enclave. Code is reviewed, scanned, and promoted across a one-way gate into the enclave, where the real CI runs.

**Implementation sketch (on-prem GitLab + internal mirrors):**

```yaml
# .gitlab-ci.yml (air-gapped)
image: registry.internal.example/python:3.12@sha256:...

variables:
  PIP_INDEX_URL: https://nexus.internal.example/repository/pypi/simple
  NPM_CONFIG_REGISTRY: https://nexus.internal.example/repository/npm/
  DOCKER_CONFIG: /secrets/docker-config  # points at internal registry

stages: [build, test, sign, publish]

sign:
  image: registry.internal.example/cosign:2.2@sha256:...
  variables:
    COSIGN_PASSWORD: $COSIGN_PASSWORD
  script:
    - cosign sign --key /secrets/cosign.key registry.internal.example/app@$DIGEST
```

**Compliance-specific additions:**

- **FIPS-validated cryptography** for signing and TLS
- **STIG-compliant base images** — tempered RHEL/Ubuntu from DISA
- **Mandatory dual approval** on production deploys
- **Tamper-evident logging** — write-once storage for pipeline logs

**Interview insight:** very senior roles — especially at defence, fintech, or healthtech — expect awareness of these constraints. Mentioning cross-domain transfer, one-way diodes, or FIPS mode demonstrates breadth.

### Q22. How would you migrate a team from Jenkins to GitHub Actions without disrupting development?

**Answer:**

A migration like this is a common senior-engineer project. Treat it as a product rollout, not a weekend rewrite.

**Phase 0 — Discovery (1-2 weeks):**

- Inventory every Jenkins job, its triggers, and its importance (blocking merge? production deploy?)
- Interview teams about which jobs they rely on and which are cargo-cult
- Measure current state: job count, pipeline duration, success rate, flake rate, monthly cost

**Phase 1 — Foundation (2-4 weeks):**

- Set up GitHub Actions runner infrastructure (ARC on Kubernetes for self-hosted; or budget for SaaS)
- Establish shared building blocks: composite actions, reusable workflows, starter templates
- Define and document migration conventions — directory layout, naming, secret management via OIDC

**Phase 2 — Parallel run (4-8 weeks):**

Migrate one team at a time. For each:

1. Translate their Jenkinsfile to a GitHub Actions workflow
2. Run both pipelines on every PR for 2 weeks
3. Compare success rate, duration, and developer feedback
4. Make the GitHub Actions workflow the required check
5. Disable the Jenkins job (but keep it as read-only for 2 more weeks)

**Translation example:**

```groovy
// Jenkinsfile
pipeline {
    agent { label 'linux && docker' }
    environment {
        DOCKER_REGISTRY = 'registry.example.com'
    }
    stages {
        stage('Build') {
            steps {
                withCredentials([string(credentialsId: 'docker-token', variable: 'DOCKER_TOKEN')]) {
                    sh 'docker login -u ci -p $DOCKER_TOKEN $DOCKER_REGISTRY'
                    sh 'docker build -t $DOCKER_REGISTRY/app:$BUILD_NUMBER .'
                    sh 'docker push $DOCKER_REGISTRY/app:$BUILD_NUMBER'
                }
            }
        }
    }
    post {
        failure { mail to: 'team@example.com', subject: "Build failed", body: "..." }
    }
}
```

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

env:
  DOCKER_REGISTRY: registry.example.com

jobs:
  build:
    runs-on: [self-hosted, linux, docker]
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.DOCKER_REGISTRY }}
          username: ci
          password: ${{ secrets.DOCKER_TOKEN }}
      - run: |
          docker build -t $DOCKER_REGISTRY/app:${{ github.run_number }} .
          docker push $DOCKER_REGISTRY/app:${{ github.run_number }}

  notify:
    needs: build
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - uses: dawidd6/action-send-mail@v3
        with: { ... }
```

**Phase 3 — Decommission (2-4 weeks):**

- Archive Jenkins job configs (often required for audit)
- Remove Jenkins agents from the fleet
- Cancel Jenkins licences, shut down controllers
- Celebrate; write the postmortem

**Key pitfalls:**

1. **Hidden Jenkins shared libraries.** Jenkins has a "Global Shared Library" mechanism many teams depend on. Those functions must be re-implemented as composite actions or reusable workflows.
2. **Build number continuity.** `$BUILD_NUMBER` in Jenkins starts at 1. `github.run_number` in GitHub Actions starts at 1 for the new workflow. Release tooling that depends on monotonic build numbers needs attention.
3. **Concurrency differences.** Jenkins can run one build at a time per agent label; GitHub Actions concurrency has different semantics (`concurrency.group`). Teams relying on Jenkins queuing behaviour may see surprises.
4. **Plugin equivalents.** Jenkins has 1,800+ plugins. Not all have GitHub Actions equivalents; some need bespoke actions.

**Interview insight:** this is a "systems migration" question as much as a CI/CD question. Show that you think about developer experience, risk, and rollback — not just YAML translation.
