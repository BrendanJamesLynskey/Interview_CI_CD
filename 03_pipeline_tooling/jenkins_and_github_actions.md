# Jenkins, GitHub Actions, and CI Tooling — Interview Questions

**Subject:** CI/CD
**Topic:** Jenkins (declarative vs scripted), GitHub Actions (workflows, matrix builds, reusable workflows, OIDC), GitLab CI, CircleCI
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is Jenkins, and what is the difference between declarative and scripted pipelines?

**Answer:**

**Jenkins** is an open-source automation server, originally a fork of Hudson (2011). It is the most widely deployed CI tool in regulated and on-premise environments because it is self-hostable, extensible (1,800+ plugins), and has been around long enough to predate everything else.

A **Jenkins pipeline** is a code definition of a CI/CD workflow stored in a `Jenkinsfile` in the repo. Jenkins supports two syntaxes:

**Declarative pipeline** — opinionated, structured, validated up front:

```groovy
pipeline {
    agent { label 'linux' }
    options { timeout(time: 30, unit: 'MINUTES') }
    stages {
        stage('Build')   { steps { sh 'make build' } }
        stage('Test')    { steps { sh 'make test' } }
        stage('Publish') {
            when { branch 'main' }
            steps { sh 'make publish' }
        }
    }
    post {
        failure { mail to: 'team@example.com', subject: "Failed: ${env.JOB_NAME}" }
    }
}
```

**Scripted pipeline** — full Groovy, imperative, more flexible but less safe:

```groovy
node('linux') {
    stage('Build') { sh 'make build' }
    stage('Test')  { sh 'make test' }
    if (env.BRANCH_NAME == 'main') {
        stage('Publish') { sh 'make publish' }
    }
}
```

| Aspect | Declarative | Scripted |
|--------|-------------|----------|
| Syntax | Opinionated DSL | Full Groovy |
| Validation | Up front (linter catches errors) | At runtime |
| Conditionals | `when {}` blocks | Arbitrary `if`/loops |
| Best for | 95% of pipelines | Edge cases needing dynamic logic |

**Interview insight:** mention you prefer declarative because the linter catches typos before the pipeline runs, and because reading a declarative pipeline is closer to reading YAML than reading code.

### Q2. What is GitHub Actions, and how is it structured?

**Answer:**

**GitHub Actions** is a CI/CD service built into GitHub, launched in 2019. Workflows live in `.github/workflows/*.yml` and execute on GitHub-hosted or self-hosted runners.

**Hierarchy:**

- **Workflow** — a YAML file triggered by events
- **Job** — a unit that runs on a single runner; jobs run in parallel by default
- **Step** — a single command or action invocation within a job
- **Action** — a reusable unit (Docker, Node.js, or composite) referenced by `uses:`

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4              # an action
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt    # a step
      - run: pytest

  lint:
    runs-on: ubuntu-latest                      # parallel with `test`
    steps:
      - uses: actions/checkout@v4
      - run: pipx run ruff check .
```

**Strengths:**

- Native GitHub integration (PR checks, environments, OIDC)
- Generous free tier for public repos
- Marketplace of pre-built actions

**Weaknesses:**

- YAML-only (no full programming language)
- Limited cross-workflow orchestration
- Self-hosted runners need careful sandboxing for public repos

### Q3. What is a matrix build, and how do you set one up in GitHub Actions?

**Answer:**

A **matrix build** runs the same job multiple times with different parameter combinations — testing against multiple OS, language versions, or dependency versions in parallel.

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
            python: "3.13"
            experimental: true
        exclude:
          - os: windows-latest
            python: "3.10"
    continue-on-error: ${{ matrix.experimental == true }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python }}
      - run: pytest
```

**Mechanics:**

- Cartesian product of `matrix` keys → one job per combination (here 3 OS x 3 Python = 9 jobs, plus 1 included, minus 1 excluded = 9)
- `include` adds extra combinations
- `exclude` removes specific combinations
- `fail-fast: false` lets all combinations run even if one fails (useful for "where does it break" analysis)

**Pitfall:** matrix size can blow up quickly. 5 OS x 5 Python x 3 dependency versions = 75 jobs. Each consumes a runner slot. Trim aggressively.

### Q4. What are reusable workflows in GitHub Actions, and how do they differ from composite actions?

**Answer:**

Both reduce duplication, but at different scopes.

**Composite action** — a packaged sequence of steps callable as a single `uses:` step:

```yaml
# .github/actions/setup-python-deps/action.yml
name: Setup Python and install deps
inputs:
  python-version: { default: "3.12" }
runs:
  using: composite
  steps:
    - uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.python-version }}
    - run: pip install -r requirements.txt
      shell: bash
```

```yaml
# Calling it
- uses: ./.github/actions/setup-python-deps
  with:
    python-version: "3.11"
```

**Reusable workflow** — a whole workflow file callable from another workflow:

```yaml
# .github/workflows/test-template.yml
on:
  workflow_call:
    inputs:
      python-version: { type: string, required: true }
    secrets:
      pypi-token: { required: false }

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-python-deps
        with:
          python-version: ${{ inputs.python-version }}
      - run: pytest
```

```yaml
# Calling it from another workflow
jobs:
  test-3-11:
    uses: ./.github/workflows/test-template.yml
    with:
      python-version: "3.11"
  test-3-12:
    uses: ./.github/workflows/test-template.yml
    with:
      python-version: "3.12"
```

| Aspect | Composite action | Reusable workflow |
|--------|------------------|-------------------|
| Scope | Steps within a job | Whole jobs (one or more) |
| Runner | Inherits caller's | Independent runner |
| Secrets | Inherits | Must be passed explicitly |
| Logging | Folded into caller's logs | Separate run with own URL |

**Rule of thumb:** composite for short reusable step sequences, reusable workflow for entire pipeline phases shared across repos.

### Q5. What is a GitLab CI pipeline, and how does it compare to GitHub Actions?

**Answer:**

GitLab CI is GitLab's built-in CI/CD, configured via `.gitlab-ci.yml` at the repo root.

```yaml
stages: [build, test, deploy]

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.pip-cache"

build:
  stage: build
  image: python:3.12
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths: [.pip-cache/]
  script:
    - pip install -r requirements.txt
    - python -m build
  artifacts:
    paths: [dist/]
    expire_in: 1 week

test:
  stage: test
  image: python:3.12
  needs: [build]
  script:
    - pip install dist/*.whl
    - pytest

deploy:
  stage: deploy
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - ./deploy.sh
  environment:
    name: production
    url: https://app.example.com
```

| Aspect | GitLab CI | GitHub Actions |
|--------|-----------|----------------|
| Configuration | `.gitlab-ci.yml` (single file or includes) | `.github/workflows/*.yml` (multiple files) |
| Stages | First-class concept (sequential by default) | Encoded via `needs:` |
| Artefacts | First-class with TTL | Via `actions/upload-artifact` |
| Environments | Built-in with deployment tracking | Built-in (`environment:` key) |
| Self-hosted | GitLab Runner (mature, used heavily on-prem) | GitHub Actions Runner |
| OIDC | Yes (`CI_JOB_JWT_V2`) | Yes (`id-token: write`) |

GitLab's `stages` model makes the visual pipeline graph cleaner — every job in stage N runs in parallel, then stage N+1. GitHub's `needs:` model is more flexible (DAG) but the graph is harder to read.

### Q6. What is OIDC federation in CI, and why has it become the recommended way to access cloud resources?

**Answer:**

**OIDC (OpenID Connect) federation** lets a CI job exchange a short-lived signed token from the CI provider for cloud credentials, replacing long-lived static access keys.

**Why it matters:**

Before OIDC, CI jobs used long-lived AWS access keys stored as repository secrets. If leaked (logs, malicious action, compromised runner), the key worked indefinitely. After OIDC, the token is valid for minutes and is scoped to the specific workflow / branch / environment.

**GitHub Actions → AWS example:**

```yaml
permissions:
  id-token: write    # required to mint the OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deploy
          aws-region: eu-west-1
      - run: aws s3 sync ./dist s3://my-app-bucket/
```

**On AWS, the IAM role's trust policy:**

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"
    }
  }
}
```

**Benefits:**

- No long-lived keys in secrets
- Per-branch / per-environment scoping via the `sub` claim
- Audit log shows which workflow assumed the role
- Rotation is automatic (token expires after each run)

**Interview insight:** if the interviewer asks "how do you authenticate CI to AWS?" and your answer mentions IAM users with access keys, you've dated yourself. OIDC has been the recommended pattern since ~2022.

---

## Intermediate

### Q7. How do you build a multi-stage GitHub Actions workflow with job dependencies and conditional execution?

**Answer:**

Use `needs:` for ordering and `if:` for conditions. Outputs from one job feed into another via `outputs`.

```yaml
name: Build, test, deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4
      - id: meta
        run: echo "version=$(git describe --tags --always)" >> $GITHUB_OUTPUT
      - run: make build VERSION=${{ steps.meta.outputs.version }}
      - uses: actions/upload-artifact@v4
        with: { name: app, path: dist/ }

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with: { name: app, path: dist/ }
      - run: ./dist/run-tests.sh

  deploy-staging:
    needs: [build, test]
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: deploy-to staging --version ${{ needs.build.outputs.version }}

  deploy-prod:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main' && !contains(github.event.head_commit.message, '[skip prod]')
    runs-on: ubuntu-latest
    environment: production    # gates approval
    steps:
      - run: deploy-to prod --version ${{ needs.build.outputs.version }}
```

**Key constructs:**

- `needs: [a, b]` — wait for both `a` and `b` to succeed
- `outputs:` — publish values from a job for downstream jobs
- `if:` — supports expressions on `github.*`, `needs.*.result`, and step status
- `environment:` — uses GitHub Environments to gate with required reviewers

**Pitfall:** if any job in `needs` is skipped, downstream jobs are also skipped by default. To run anyway, use `if: always()` or `if: !cancelled()`.

### Q8. What is the Jenkins shared library pattern, and what are its trade-offs?

**Answer:**

A **Jenkins shared library** packages reusable Groovy code that any Jenkinsfile in the organisation can import. It enables centralised CI logic without copy-pasting Jenkinsfiles into every repo.

**Structure (in a separate repo):**

```
my-shared-lib/
├── vars/
│   ├── buildPython.groovy        # exposes a `buildPython` step
│   └── deployToK8s.groovy
├── src/
│   └── com/example/Notify.groovy # classes
└── resources/                    # files loadable via libraryResource
```

**`vars/buildPython.groovy`:**

```groovy
def call(Map config = [:]) {
    def pythonVersion = config.pythonVersion ?: '3.12'
    docker.image("python:${pythonVersion}").inside {
        sh 'pip install -r requirements.txt'
        sh 'pytest'
    }
}
```

**Using it from a Jenkinsfile:**

```groovy
@Library('my-shared-lib@v2.1') _

pipeline {
    agent any
    stages {
        stage('Test') {
            steps {
                buildPython(pythonVersion: '3.11')
            }
        }
    }
}
```

**Trade-offs:**

| Pro | Con |
|-----|-----|
| One change updates all consumers | One bug breaks all consumers |
| Centralised security/lint logic | Versioning and rollout become a project |
| Promotes consistency | Easy to over-abstract; debugging is hard |
| Reduces Jenkinsfile churn | Library code runs with high privileges (security risk) |

**Best practice:** version the library and pin consumers to specific tags (`@Library('my-shared-lib@v2.1')`), not `@main`. Treat the library like production code — code review, tests, semantic versioning.

### Q9. How do you securely handle secrets in GitHub Actions, and what are the common pitfalls?

**Answer:**

**Secret types in GitHub Actions:**

- **Repository secrets** — visible to all workflows in the repo
- **Environment secrets** — scoped to a specific GitHub Environment (production, staging)
- **Organisation secrets** — shared across many repos with access controls

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production              # only environment-secrets visible here
    steps:
      - run: deploy.sh
        env:
          API_TOKEN: ${{ secrets.PROD_API_TOKEN }}
```

**Built-in protections:**

- Secrets are masked in logs (`***`)
- Secrets are not passed to workflows triggered by PRs from forks
- Cannot be read back through the API (write-only)

**Common pitfalls:**

1. **Echoing into logs.** `echo $API_TOKEN | base64` defeats masking — log redaction is exact-string only. Test with `--debug` off.

2. **Passing secrets to third-party actions.** A compromised action with `${{ secrets.* }}` access can exfiltrate everything. Pin actions to a commit SHA, not a tag:

   ```yaml
   - uses: third-party/action@a1b2c3d4...    # not @v1
   ```

3. **`pull_request_target` events.** Unlike `pull_request`, this trigger has access to secrets *and* runs in the context of the base repo. Combined with `actions/checkout` of the PR branch, this is a privilege escalation vector. Use `pull_request` for untrusted PRs.

4. **Self-hosted runners + public repos.** A malicious PR can run arbitrary code on your runner with access to anything on the host. Use ephemeral, sandboxed runners (e.g., `actions-runner-controller` on Kubernetes).

5. **Long-lived secrets.** Prefer OIDC federation (Q6). If you must use static secrets, rotate them; auditable rotation cadence is a SOC 2 control.

### Q10. Compare GitHub Actions caching to GitLab CI caching to CircleCI caching.

**Answer:**

All three offer key-based caches stored remotely; the differences are in scope, eviction, and ergonomics.

**GitHub Actions:**

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: pip-
```

- Cache scoped to the branch + base branch fallback
- 10 GB total per repo; LRU eviction after 7 days unused
- One key per cache; `restore-keys` provides prefix-match fallback

**GitLab CI:**

```yaml
build:
  cache:
    key:
      files: [requirements.txt]
    paths: [.cache/pip]
    policy: pull-push      # or pull-only / push-only
```

- Per-project (or per-runner) storage on GitLab's object store
- `policy:` controls whether the job can push the cache
- TTL configurable; no global limit on self-managed instances

**CircleCI:**

```yaml
- restore_cache:
    keys:
      - pip-{{ checksum "requirements.txt" }}
      - pip-
- run: pip install -r requirements.txt
- save_cache:
    paths: [~/.cache/pip]
    key: pip-{{ checksum "requirements.txt" }}
```

- Caches are immutable once written (must change the key to update)
- 30-day TTL by default
- Workspaces (separate concept) pass files within a workflow

| Feature | GitHub Actions | GitLab CI | CircleCI |
|---------|---------------|-----------|----------|
| Mutability | Mutable per key | Mutable | Immutable |
| Branch isolation | Yes (with main fallback) | Configurable | None (org-wide) |
| Total size | 10 GB / repo | Configurable | 15 GB / project |
| Restore-key fallback | Yes (prefix) | No | Yes (list) |
| Best feature | Auto fallback to main | `pull-push` policy | Workspaces |

**Cross-tool insight:** all three benefit from a tiered caching strategy — fast L1 (CI provider) for branch caches, L2 remote build cache (Bazel, sccache, Turborepo) for cross-team sharing.

### Q11. How does CircleCI's orbs system work, and what problem does it solve?

**Answer:**

**Orbs** are CircleCI's reusable configuration packages — a hybrid of GitHub Actions and reusable workflows, but published to a central registry.

```yaml
version: 2.1
orbs:
  python: circleci/python@2.1.1
  aws-cli: circleci/aws-cli@4.1.0

jobs:
  test:
    docker:
      - image: cimg/python:3.12
    steps:
      - checkout
      - python/install-packages:
          pkg-manager: pip
          pip-dependency-file: requirements.txt
      - run: pytest

  deploy:
    docker:
      - image: cimg/base:stable
    steps:
      - aws-cli/install
      - aws-cli/setup:
          role-arn: arn:aws:iam::123:role/deploy    # OIDC
      - run: aws s3 sync ./dist s3://bucket/

workflows:
  build-deploy:
    jobs:
      - test
      - deploy:
          requires: [test]
          filters: { branches: { only: main } }
```

**What orbs provide:**

- **Commands** — reusable step sequences (similar to composite actions)
- **Jobs** — reusable jobs (similar to reusable workflows)
- **Executors** — reusable runner definitions

**Versioning:** orbs use semver (`@2.1.1`). The orb registry verifies orbs from "certified" publishers, providing a trust signal absent from the GitHub Actions Marketplace.

**Trade-off:** lock-in. Orbs are CircleCI-specific. A migration off CircleCI requires rewriting every orb usage in the destination tool's idiom.

### Q12. What are GitHub Actions environments, and how do you use them for deployment gating?

**Answer:**

**GitHub Environments** are named deployment targets (production, staging, preview) with optional protection rules.

**Protection rules:**

- **Required reviewers** — N humans must approve before the job runs
- **Wait timer** — mandatory delay (e.g., 30 minutes for canary observation)
- **Branch restrictions** — only specific branches can deploy
- **Environment secrets** — scoped to this environment only

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - run: deploy --env staging
        env:
          API_KEY: ${{ secrets.STAGING_API_KEY }}

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production              # required reviewers configured in repo settings
      url: https://app.example.com
    steps:
      - run: deploy --env prod
        env:
          API_KEY: ${{ secrets.PROD_API_KEY }}
```

**What the deployment view gives you:**

- A history of deploys per environment
- "Active" deployment shown in the GitHub UI
- API to query current and past deployments

**Compliance angle:** environments give you SOC 2 evidence for "production deploys require approval" — the approval is recorded in the GitHub audit log, the workflow run, and the deployment object.

---

## Advanced

### Q13. Design a multi-repo CI strategy where 30 microservices share a common pipeline definition. Discuss reusable workflows, composite actions, and template repos.

**Answer:**

The goal: change the standard pipeline once, propagate to all 30 services, while leaving room for service-specific overrides.

**Layered approach:**

**Layer 1 — Composite actions (in a `ci-actions` repo) for atomic operations:**

```yaml
# ci-actions/.github/actions/build-python/action.yml
name: Build Python service
inputs:
  python-version: { default: "3.12" }
runs:
  using: composite
  steps:
    - uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.python-version }}
        cache: pip
    - run: pip install -r requirements.txt
      shell: bash
    - run: python -m build
      shell: bash
```

**Layer 2 — Reusable workflows (in a `ci-workflows` repo) for full pipeline phases:**

```yaml
# ci-workflows/.github/workflows/python-service-ci.yml
on:
  workflow_call:
    inputs:
      python-version: { type: string, default: "3.12" }
      run-integration: { type: boolean, default: true }

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: my-org/ci-actions/.github/actions/build-python@v1
        with: { python-version: ${{ inputs.python-version }} }
      - run: pytest
      - if: ${{ inputs.run-integration }}
        run: pytest tests/integration
```

**Layer 3 — Per-service workflow file (in each service repo) — minimal:**

```yaml
# service-foo/.github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  ci:
    uses: my-org/ci-workflows/.github/workflows/python-service-ci.yml@v3
    with:
      python-version: "3.11"   # this service is on 3.11
```

**Layer 4 — Template repo for new services:**

GitHub's template repos let `New Repository → Use this template` scaffold a new service with the standard structure, including the per-service workflow file pre-wired.

**Versioning the shared workflows:**

- Tag releases (`v1`, `v1.2.3`) on `ci-workflows`
- Consumers pin to a major version (`@v1`) and get patch/minor updates automatically
- Use Dependabot to bump pinned versions across consumer repos

**Rollout safety:**

1. Make the change on a feature branch in `ci-workflows`
2. Test against one canary service repo by pinning its workflow to the branch
3. Tag release once green
4. Dependabot opens PRs across all 30 repos to bump

**Pitfall:** "abstraction debt." After three years, the shared workflow has 47 input parameters covering every edge case. Periodically refactor and force consumers to upgrade — don't carry old paths forever.

### Q14. How do you migrate a 5,000-job Jenkins instance to GitHub Actions? Walk through the strategy.

**Answer:**

This is a multi-quarter project, not a weekend rewrite. Treat it as a systems migration.

**Phase 1 — Discovery and categorisation (4-6 weeks):**

- Export all Jenkins job configs (use the API or `jenkinsfile-runner`)
- Classify by type: build, test, deploy, scheduled, infrastructure
- Identify shared library usage and plugins in use
- Identify which jobs are still active (last run < 90 days)
- Sunset dead jobs first; that's free progress

**Phase 2 — Standard pipeline templates (4 weeks):**

Build reusable workflows for the common patterns identified in Phase 1:

```yaml
# .github/workflows/standard-python-ci.yml
on: { workflow_call: { inputs: { ... } } }
jobs: { ... }
```

Cover 80% of jobs with 5-10 templates. Long-tail edge cases get bespoke workflows.

**Phase 3 — Pilot migration (4 weeks):**

- Pick 3 services covering different patterns (one with a Docker build, one with shared library usage, one with a deploy)
- Migrate them and run Jenkins + GitHub Actions in parallel for two weeks
- Compare outputs, debug differences
- Document gotchas in a migration playbook

**Phase 4 — Bulk migration (8-12 weeks):**

- Open Dependabot-style PRs to add the GitHub Actions workflow to each repo
- Keep Jenkins running in parallel; teams flip the "source of truth" when ready
- Track migration progress on a dashboard
- Provide office hours for stuck teams

**Phase 5 — Decommission (4 weeks):**

- Confirm zero traffic on Jenkins jobs for 30 days
- Archive Jenkins job configs (audit requirement in regulated industries)
- Snapshot Jenkins controller VM and shut it down
- Cancel licences

**Key risks and mitigations:**

| Risk | Mitigation |
|------|------------|
| Build numbers reset (release tooling depends on monotonic IDs) | Seed `github.run_number` offset or move to commit SHA-based versioning |
| Jenkins shared library functions don't have GHA equivalents | Re-implement as composite actions in a shared repo |
| Self-hosted Jenkins agents have specific tooling | Use self-hosted GHA runners with the same image |
| Concurrency / queueing semantics differ | Test with `concurrency.group` patterns; run parallel for two weeks |
| Plugins (1,800+) have no equivalent | Inventory critical plugins early; build/buy alternatives |

**Interview insight:** mention that you'd track "migration burn-down" (jobs migrated / total) as a leading indicator and "Jenkins traffic" (runs per day) as a lagging indicator. Senior interviewers want to hear you've thought about how to *measure* migration progress, not just how to do the YAML.

### Q15. You inherit a GitHub Actions workflow that takes 45 minutes per PR. Walk through how you'd diagnose and reduce that.

**Answer:**

Treat this as performance engineering: measure first, optimise hot spots, verify each change.

**Step 1 — Get the timing breakdown.**

Open the slowest run and look at the "Total duration" in each job and step. Look for:

- Jobs running serially that could run in parallel (`needs:` chains too long)
- A single job with disproportionate runtime (test job 30 minutes, others 5)
- Setup steps consuming disproportionate time (deps install 10 minutes)

**Step 2 — Check parallelism.**

```yaml
# Bad — serial
jobs:
  lint:    { ... }
  test:    { needs: lint }
  build:   { needs: test }

# Good — parallel where possible
jobs:
  lint:    { ... }
  test:    { ... }                            # parallel with lint
  build:   { needs: [lint, test] }
```

Lint and test rarely depend on each other; let them race.

**Step 3 — Cache aggressively.**

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: "3.12"
    cache: pip                  # built-in cache
- uses: actions/cache@v4
  with:
    path: ~/.cache/pre-commit
    key: precommit-${{ hashFiles('.pre-commit-config.yaml') }}
```

Audit "install dependencies" steps — each unnecessary network round-trip is 30 seconds.

**Step 4 — Shard the test suite.**

```yaml
test:
  strategy:
    fail-fast: false
    matrix:
      shard: [1, 2, 3, 4, 5, 6]
  steps:
    - run: pytest --shard=${{ matrix.shard }}/6
```

Six shards on a 30-minute test suite ≈ 5 min/shard. Combined with `pytest-xdist` for within-shard parallelism, often 10x speedup.

**Step 5 — Cancel superseded runs.**

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

If a developer pushes 3 commits in 10 minutes, only the latest runs.

**Step 6 — Move expensive checks off the PR path.**

End-to-end tests and security scans don't need to gate every PR. Run them on `main` post-merge. Required PR checks should be the fastest, most discriminating ones.

**Step 7 — Use bigger runners for the bottleneck.**

GitHub offers larger runners (8, 16, 32 cores). The break-even is when wall-clock savings cost less than the per-minute uplift. Test compilation often benefits.

**Typical result:** 45 min → 8 min by combining sharding, caching, parallelism, and moving E2E to main.

### Q16. Compare GitHub Actions and Jenkins for a regulated environment that requires on-premise execution and audit trails.

**Answer:**

Both can satisfy regulated requirements, but with different operational profiles.

| Requirement | Jenkins | GitHub Actions |
|-------------|---------|----------------|
| On-premise execution | Native (Jenkins controller + agents on-prem) | GitHub Enterprise Server (GHES) self-hosted, or self-hosted runners with GitHub.com |
| Audit trail | Jenkins audit plugin; per-job logs | GitHub Audit Log API (read-only, immutable) |
| Approval workflows | Manual stage with approvers | GitHub Environments with required reviewers |
| Pipeline-as-code | Jenkinsfile in Git | YAML workflows in Git (native) |
| Secrets vault integration | HashiCorp Vault plugin (mature) | OIDC to Vault, or community actions |
| Air-gapped operation | Yes (fully self-contained) | GHES supports air-gap; GitHub.com requires egress |
| Compliance certifications | Depends on your install | GitHub.com / GHES inherit GitHub's SOC 2, FedRAMP |

**Where Jenkins still wins:**

- Air-gapped environments with no GitHub footprint
- Highly bespoke build pipelines (complex Groovy logic)
- Existing investment in Jenkins shared libraries and plugins
- Regulated industries with deep ties to Jenkins audit tooling

**Where GitHub Actions wins:**

- Lower operational overhead (no controller to patch)
- Native PR integration (no separate "GitHub PR check" plugin)
- Modern security model (OIDC, SHA-pinned actions, scoped tokens)
- Marketplace ecosystem reduces bespoke code

**Hybrid pattern:** use GitHub for source control, PR review, and required checks (lint, unit test); use Jenkins for production deploys that need access to on-prem databases or specialised hardware. The PR-side and prod-side don't have to be on the same tool.

**Interview insight:** in regulated industries the question is rarely "which is technically better" but "which audit story is easier to defend." Auditors are familiar with Jenkins; if your team is too, the migration cost may not be justified.

### Q17. How do you implement a self-service "deploy to production" workflow in GitHub Actions that requires two-person approval and produces a SOC 2-compliant audit trail?

**Answer:**

The pattern combines GitHub Environments, OIDC, and the audit log to provide approvals, separation of duties, and immutable evidence.

**Workflow definition:**

```yaml
# .github/workflows/deploy-prod.yml
name: Deploy to production
on:
  workflow_dispatch:
    inputs:
      version:
        description: "Release tag to deploy (e.g., v2.4.1)"
        required: true
        type: string
      change_ticket:
        description: "Change management ticket (e.g., CHG-1234)"
        required: true
        type: string

permissions:
  id-token: write
  contents: read
  deployments: write

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ inputs.version }}
      - run: |
          # Validate the tag is signed and matches an approved release
          git tag --verify ${{ inputs.version }}
      - name: Verify change ticket
        run: |
          curl -fsSL -H "Authorization: Bearer ${{ secrets.JIRA_TOKEN }}" \
            "https://example.atlassian.net/rest/api/3/issue/${{ inputs.change_ticket }}" \
            | jq -e '.fields.status.name == "Approved"'

  deploy:
    needs: validate
    runs-on: ubuntu-latest
    environment:
      name: production            # configured with TWO required reviewers
      url: https://app.example.com
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ inputs.version }}
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123:role/deploy-prod
      - run: ./deploy.sh ${{ inputs.version }}
      - name: Record deployment
        run: |
          gh api repos/${{ github.repository }}/deployments \
            -f ref=${{ inputs.version }} \
            -f environment=production \
            -f description="CHG=${{ inputs.change_ticket }}"
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
```

**Environment configuration (in GitHub UI):**

- Required reviewers: 2 (from a designated `prod-deployers` team)
- Deployment branches: only `main` and tags matching `v*.*.*`
- Wait timer: 5 minutes (window to abort)

**SOC 2 evidence produced:**

| Control | Artefact |
|---------|----------|
| Authorisation | Two named approvals in workflow run + audit log |
| Change tracking | `change_ticket` input cross-references CHG record |
| Code integrity | Signed tag verification step |
| Access control | OIDC with scoped IAM role, no long-lived keys |
| Audit trail | Workflow run log, GitHub audit log, AWS CloudTrail |
| Separation of duties | Code author (dev) cannot approve their own deploy (configurable) |

**Hardening:**

- Restrict who can trigger `workflow_dispatch` (collaborators only)
- Forbid the requester from approving their own deploy (Environment setting)
- Pin third-party actions to commit SHAs to prevent supply chain tampering
- Forward GitHub audit log to an external SIEM (Splunk, Datadog) with WORM storage

**Interview insight:** auditors look for traceability — given a deploy, can you produce: who approved it, what changed, what ticket authorised it, who triggered it, what credentials were used? This pattern produces all five from native GitHub features without bespoke tooling.

### Q18. What are the security risks of third-party GitHub Actions, and how do you mitigate them at scale across 200 repositories?

**Answer:**

Third-party actions execute arbitrary code with access to the runner, the repo's secrets (if granted), and the workflow token. A compromised popular action is a supply chain disaster — see the `tj-actions/changed-files` compromise in 2025, which leaked secrets across thousands of repos.

**Risk taxonomy:**

| Risk | Example |
|------|---------|
| Action publisher account compromised | Attacker pushes malicious update to existing tag |
| Action gets sold/transferred | New owner publishes malicious version |
| Tag reused (`@v1` mutable) | Mitigation defeated by tag mutation |
| Transitive action dependencies | Action you trust uses an action you don't |
| Token exfiltration | Action posts `${{ secrets.GITHUB_TOKEN }}` to remote URL |

**Mitigations at scale:**

**1. Pin actions to commit SHAs, not tags:**

```yaml
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11    # v4.1.1
- uses: hashicorp/setup-terraform@a1502cd9e758c50496cc9ac5308c4843bcd56d36
```

The SHA is immutable. Even if the tag is moved, your workflow runs the original code.

**2. Use Dependabot to keep pinned SHAs current:**

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule: { interval: weekly }
```

Dependabot opens PRs to bump pinned SHAs when new versions are released. Review the diff before merging.

**3. Maintain an allow-list of approved actions:**

In repository settings (or organisation-wide), restrict to a curated list:

```
actions/*, my-org/*, hashicorp/setup-terraform@a1502cd9...
```

New actions need review and addition to the allow-list, slowing supply chain compromise.

**4. Minimise permissions:**

```yaml
permissions: {}              # default to none
jobs:
  build:
    permissions:
      contents: read         # add only what's needed
      pull-requests: write
```

A token with no permissions can't exfiltrate much.

**5. Avoid `pull_request_target` with checkout of PR head:**

This combination gives untrusted PR code access to secrets. Restrict to `pull_request` (no secrets) for forks, or use a two-workflow pattern where the privileged second workflow runs only after a `workflow_run` from a trusted reviewed workflow.

**6. Run a continuous scan:**

Tools like `zizmor`, `actionlint`, and `octoscan` flag risky patterns (mutable tags, `pull_request_target` misuse, overly broad permissions). Run them in your CI lint job and across all 200 repos via a scheduled scan.

**7. Vendor critical actions:**

For very high-value workflows (production deploys, secret rotation), copy the action's code into your own org's repo and reference the internal copy. You trade Dependabot ergonomics for full control over the supply chain.

**Interview insight:** mention specific incidents (Codecov 2021, `tj-actions/changed-files` 2025) by name. It signals you keep up with industry events, not just textbooks. Then map each mitigation to a real attack pattern, demonstrating you understand the threat model rather than just listing rules.
