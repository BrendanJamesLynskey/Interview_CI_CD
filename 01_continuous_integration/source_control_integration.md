# Source Control Integration — Interview Questions

**Subject:** CI/CD
**Topic:** Git Workflows, Branch Protection, Webhooks, Monorepo Strategies
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is GitFlow and what are its drawbacks for modern CI/CD?

**Answer:**

**GitFlow** (Vincent Driessen, 2010) is a branching model with five branch types:

- `main` (or `master`) — production-ready code
- `develop` — integration branch for next release
- `feature/*` — new features, branched from `develop`
- `release/*` — stabilisation branches for upcoming releases
- `hotfix/*` — urgent production fixes, branched from `main`

**Flow:** feature → develop → release → main → tag.

**Drawbacks for CI/CD:**

1. **Long-lived branches cause merge conflicts.** A feature branch open for two weeks diverges from `develop` significantly. Conflicts slow everyone down.
2. **`develop` and `main` diverge.** The same fix has to be cherry-picked or merged back and forth.
3. **Incompatible with continuous deployment.** You cannot deploy `develop` straight to production; you must cut a release branch, stabilise it, then merge to main.
4. **Hides integration problems.** Developers only see merge conflicts at the end. By then, it's hard to untangle.
5. **Too many branches to protect.** Each needs its own CI rules and approvals.

**When GitFlow makes sense:**

- Shrink-wrapped software with explicit versioned releases (think: desktop apps, firmware)
- Multiple versions supported in parallel (v1.x, v2.x)
- Teams that cannot deploy on demand

**When to avoid it:**

- SaaS / continuously deployed products — use trunk-based development
- Small teams — the ceremony outweighs the benefit

**Interview insight:** stating that "GitFlow is an anti-pattern for SaaS" is a reasonable position; more nuanced is "GitFlow is a correct fit for its era and context, but continuous deployment needs trunk-based development."

### Q2. What is trunk-based development, and why is it preferred for CI/CD?

**Answer:**

**Trunk-based development (TBD)** is a branching model where all developers integrate into a single shared branch (`main` / `trunk`) at least daily. Feature branches — if used at all — live for hours, not weeks.

**Core practices:**

1. Commit or merge to `main` multiple times per day
2. Keep `main` always releasable (green CI, feature-flagged)
3. Short-lived branches (< 24 hours)
4. Feature flags for in-progress work, hiding unfinished features behind runtime toggles

**Why it suits CI/CD:**

- No long-lived branches to diverge and conflict
- `main` is always deployable → continuous delivery is natural
- Integration bugs surface immediately, not at release time
- Branch protection is simple: protect one branch

**Variants:**

- **Strict TBD** — direct commits to `main`. Requires excellent test coverage and strong individual contributors. Google, Facebook.
- **Scaled TBD** — short-lived branches with PRs. Most modern teams. Still TBD as long as branches are < 1 day.

**Typical workflow:**

```
git checkout -b fix-payment-validation main
# ... edits ...
git commit -m "Validate payment amount is positive"
git push origin fix-payment-validation
# Open PR, CI runs, reviewer approves
# Squash-merge to main within hours
```

**Feature flag pattern for incomplete features:**

```python
from featureflags import is_enabled

def checkout(user, cart):
    if is_enabled("new_checkout_ui", user=user):
        return new_checkout_flow(user, cart)
    return old_checkout_flow(user, cart)
```

This lets you merge half-built features to `main` every day, incrementally, without exposing them to users.

**Interview insight:** connect TBD to DORA metrics — teams with high deployment frequency and low lead time overwhelmingly use TBD. Mention the State of DevOps Report findings.

### Q3. What is a pull request (or merge request), and what roles does it serve?

**Answer:**

A **pull request** (GitHub, Gitea) or **merge request** (GitLab) is a proposal to merge changes from one branch into another. It serves several distinct purposes:

1. **Code review** — peer evaluation of correctness, style, and design
2. **CI trigger** — automated validation of the proposed change
3. **Discussion forum** — inline comments tied to specific lines
4. **Gate** — merge is blocked until approvals and checks pass
5. **Audit trail** — who reviewed, who approved, what CI ran, when merged
6. **Documentation** — the PR title + description become part of the historical record

**Anatomy of a good PR:**

- Small (< 400 lines changed, ideally < 200)
- Focused on one concern
- Clear title explaining *why*, not just *what*
- Description includes context, screenshots if UI, test instructions
- Passes required checks before requesting review

**CI-PR integration:**

Every PR triggers the CI workflow scoped to that branch. The CI reports status back to the PR via the provider's Checks API:

```yaml
name: PR Checks
on: pull_request
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make test
      - if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: 'Tests failed. See logs.'
            })
```

**Interview insight:** some companies use **stacked PRs** (Graphite, Phabricator) where one logical change is split into a chain of small reviewable commits. Mention it if the interviewer asks about reviewing large changes.

### Q4. What is branch protection, and what rules should you enforce on `main`?

**Answer:**

**Branch protection** prevents direct pushes and enforces checks on a branch. It's the policy layer that keeps `main` safe.

**Recommended rules for `main`:**

1. **Require pull request before merging** — no direct commits
2. **Require approvals** — usually 1-2, with CODEOWNERS for sensitive paths
3. **Dismiss stale approvals when new commits push** — re-review after changes
4. **Require status checks to pass** — CI, security scans, linters
5. **Require branches to be up to date** before merging — ensures tested state
6. **Require signed commits** — verifies author identity
7. **Require linear history** — no merge commits (forces rebase or squash)
8. **Include administrators** — rules apply even to admins
9. **Restrict who can push** — only after PR merge; no force-pushes

**GitHub branch protection via API:**

```bash
gh api -X PUT repos/org/repo/branches/main/protection \
  -f required_status_checks.strict=true \
  -f required_status_checks.contexts[]='ci/test' \
  -f required_status_checks.contexts[]='ci/lint' \
  -f enforce_admins=true \
  -f required_pull_request_reviews.required_approving_review_count=2 \
  -f required_pull_request_reviews.dismiss_stale_reviews=true \
  -f required_pull_request_reviews.require_code_owner_reviews=true \
  -f required_linear_history=true \
  -f allow_force_pushes=false \
  -f allow_deletions=false
```

**GitHub Rulesets (newer, preferred):**

```yaml
# .github/rulesets/main.yml-ish (via API)
# Rulesets can target tags/branches across multiple repos
```

**Common failure modes:**

- "Include administrators" off → admins push broken code at 5 pm Friday
- Status checks not marked required → CI failures ignored
- CODEOWNERS out of date → wrong people reviewing security-critical paths

**Interview insight:** branch protection is where CI/CD meets governance. SOC2/ISO27001 audits inspect these settings; answers should mention enforceability and audit trail.

### Q5. What is a webhook, and how does it trigger CI pipelines?

**Answer:**

A **webhook** is an HTTP callback. When an event occurs (push, PR opened, issue commented), the source-control system sends a POST request to a configured URL with a JSON payload describing the event.

**Flow:**

```
git push → GitHub receives push
        → GitHub POSTs to https://ci.example.com/webhooks/github
        → CI server validates signature, parses payload
        → CI server enqueues build for the pushed ref
```

**Payload example (push event, truncated):**

```json
{
  "ref": "refs/heads/feature-xyz",
  "repository": {"full_name": "org/repo"},
  "commits": [{"id": "abc123", "message": "Add feature"}],
  "pusher": {"name": "brendan"}
}
```

**Security: HMAC signature verification.**

Every webhook includes a header like `X-Hub-Signature-256: sha256=<hex>`. The CI server recomputes the HMAC using a shared secret and compares:

```python
import hmac, hashlib

def verify(body: bytes, signature_header: str, secret: str) -> bool:
    expected = "sha256=" + hmac.new(
        secret.encode(), body, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature_header)
```

**Common problems:**

- **Webhook deliveries fail when CI is offline.** Providers retry with exponential backoff, but after N attempts, deliveries are dropped. Build a reconciliation job that polls for missed events.
- **Duplicate deliveries.** Webhooks are at-least-once. Treat handlers as idempotent.
- **Replay attacks.** Include the delivery ID and timestamp in a short-lived cache; reject duplicates.

**Modern alternative: GitHub Actions / GitLab CI built-in.**

Managed CI that lives in the source-control system eliminates the webhook plumbing — the events are processed inside the same service. You only need external webhooks for external CI systems (Jenkins, custom servers).

### Q6. What's the difference between merge commits, squash merges, and rebase merges?

**Answer:**

Three ways to integrate a feature branch into `main`:

| Method | Commits on main | Parent links | History |
|--------|-----------------|--------------|---------|
| **Merge commit** | All feature commits + 1 merge commit | 2 parents | Preserves branch structure |
| **Squash merge** | 1 commit (all squashed) | 1 parent | Flat, no branch trace |
| **Rebase merge** | All feature commits, linearised | 1 parent each | Flat, preserves individual commits |

**Merge commit:**

```
*   merge branch 'feature-xyz'
|\
| * Add validation tests
| * Add validation logic
|/
* main: previous commit
```

**Squash merge:**

```
* Add input validation (squashed from feature-xyz) [#123]
* main: previous commit
```

**Rebase merge:**

```
* Add validation tests
* Add validation logic
* main: previous commit
```

**When to use each:**

- **Squash merge** — default for most teams. One PR = one commit. Clean history. Easy to revert. Loses fine-grained commit history (not usually valuable).
- **Rebase merge** — teams that value atomic, bisect-able commits on `main`. Requires disciplined commit hygiene.
- **Merge commit** — preserves branch topology. Useful for release branches or long-lived work where the structure matters. Rare in trunk-based teams.

**Bisecting implications:**

`git bisect` works best with small, atomic commits. Squash-merging hurts bisect resolution (one commit might span hundreds of lines). Rebase-merge helps, but only if each individual commit actually built and passed tests.

**Interview insight:** "Which merge strategy does your team use?" is a good early question — it reveals cultural priorities. A team using merge commits is prioritising history preservation; a team using squash prioritises simplicity.

---

## Intermediate

### Q7. What is a CODEOWNERS file, and how does it integrate with CI and PRs?

**Answer:**

`CODEOWNERS` is a file that declares ownership of specific paths within the repository. When a PR touches those paths, the owners are auto-requested for review, and branch protection can require their approval.

**File location:** `.github/CODEOWNERS`, `docs/CODEOWNERS`, or repo root.

**Syntax:**

```
# Default owners
*                               @org/platform-team

# Frontend
frontend/                       @org/frontend-team
*.tsx                           @org/frontend-team

# Security-critical
infra/terraform/                @org/platform-team @org/security-team
/.github/workflows/             @org/platform-team @org/security-team
SECURITY.md                     @org/security-team

# Specific files
/go.mod                         @org/platform-team
/package.json                   @org/platform-team

# Negation (exclude)
frontend/                       @org/frontend-team
frontend/legacy/                # no owner; anyone can approve
```

**Integration with branch protection:**

Enable "Require review from Code Owners" in branch protection. Now any PR touching `infra/terraform/` requires approval from both `platform-team` and `security-team`.

**CI integration patterns:**

- **Path-filtered workflows** — run different checks per path (see Q8 and `build_pipelines.md` Q13)
- **Automated security review** — if a PR touches secrets or auth code, trigger extra SAST runs

**Pitfalls:**

- **Drift.** Teams reorganise; CODEOWNERS doesn't get updated. Stale owners block PRs.
- **Over-broad defaults.** `* @org/platform-team` means platform is a bottleneck on every PR.
- **Missing coverage.** Paths with no owner fall through to the default; critical paths like `.github/workflows/` might be missed.

**Audit script:**

```python
# scripts/audit_codeowners.py
import subprocess, pathlib

owners_file = pathlib.Path("CODEOWNERS").read_text()
patterns = [line.split()[0] for line in owners_file.splitlines() if line and not line.startswith("#")]

all_files = subprocess.check_output(["git", "ls-files"]).decode().splitlines()
# Check every file maps to at least one rule...
```

**Interview insight:** mention that CODEOWNERS is how you scale review to a large monorepo. With 500 services, no single team can review everything; CODEOWNERS routes reviews automatically.

### Q8. Design a CI strategy for a monorepo with selective builds based on changed paths.

**Answer:**

See `build_pipelines.md` Q13 for the full answer. Here, focus on the source-control integration aspects.

**Path-filter-driven workflow triggering:**

GitHub Actions' `paths` filter at the workflow level:

```yaml
# .github/workflows/frontend.yml
on:
  pull_request:
    paths:
      - "frontend/**"
      - "libs/ui/**"

# .github/workflows/backend.yml
on:
  pull_request:
    paths:
      - "backend/**"
      - "libs/shared/**"
```

**Required checks gotcha:**

If you configure `frontend` as a required check but the PR only touches `backend`, the `frontend` workflow won't run — and the PR will be stuck waiting for a required check that will never start.

**Fix: use a "always-runs" gate job:**

```yaml
# .github/workflows/ci-gate.yml — always runs
on: pull_request
jobs:
  gate:
    runs-on: ubuntu-latest
    steps:
      - run: echo "ci gate passed"
```

Mark `gate` as required. Move the per-path workflows to optional; let the downstream gate aggregate. Alternatively, use a skip-aware pattern:

```yaml
jobs:
  filter:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.f.outputs.frontend }}
      backend:  ${{ steps.f.outputs.backend }}
    steps:
      - uses: dorny/paths-filter@v3
        id: f
        with:
          filters: |
            frontend: 'frontend/**'
            backend:  'backend/**'

  frontend-ci:
    needs: filter
    if: needs.filter.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    steps: [...]

  backend-ci:
    needs: filter
    if: needs.filter.outputs.backend == 'true'
    runs-on: ubuntu-latest
    steps: [...]

  all-green:
    needs: [frontend-ci, backend-ci]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: |
          if [[ "${{ needs.frontend-ci.result }}" == "failure" || \
                "${{ needs.backend-ci.result }}"  == "failure" ]]; then
            exit 1
          fi
```

Make `all-green` the required check. It succeeds if all triggered jobs succeeded (skipped counts as neutral).

**Shared dependency handling:**

When `libs/shared/` changes, all consumers must rebuild. The dependency graph must be declared somewhere — `CODEOWNERS`-like file, or derived from `go.mod`/`package.json` workspace definitions.

### Q9. What are Git hooks, and how do pre-commit hooks relate to CI?

**Answer:**

**Git hooks** are scripts Git runs at specific points in its workflow. The client-side hooks relevant to CI:

- **pre-commit** — runs before `git commit`; can reject the commit
- **commit-msg** — validates the commit message
- **pre-push** — runs before `git push`

**Why hooks shift work left:**

If CI rejects a PR because of trailing whitespace, the developer wasted a round-trip (15 minutes) on something a hook could have caught in 200 milliseconds locally.

**The `pre-commit` framework (most popular):**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: [--maxkb=500]
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.2
    hooks:
      - id: mypy
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
```

```bash
pip install pre-commit
pre-commit install  # installs .git/hooks/pre-commit
```

**Run the same checks in CI:**

```yaml
- uses: pre-commit/action@v3.0.1
```

Now your local hooks and CI run identical checks — fail-fast locally, and if someone skips hooks with `git commit --no-verify`, CI catches them.

**Key principles:**

1. **Hooks must be fast.** > 10 seconds and developers will skip them.
2. **Hooks must be idempotent.** Running twice on unchanged code does nothing.
3. **Hooks must be opt-out-able.** `--no-verify` exists for a reason (WIP commits). But CI enforces.
4. **Don't put secrets or network access in hooks.** They run on untrusted developer machines.

**server-side Git hooks:**

For true enforcement, server-side `pre-receive` hooks on the Git server reject pushes that violate policy. Rare in practice — branch protection is usually sufficient.

**Interview insight:** candidates who mention the pre-commit framework and its re-use in CI show that they think about developer experience and feedback loops.

### Q10. How do you manage secrets that live in the repository but shouldn't be in plain text?

**Answer:**

Some configuration (TLS certs, encrypted env files, Kubernetes secrets) needs to live in Git for versioning and audit, but must not be readable in plain text.

**Three mature approaches:**

**1. SOPS (Mozilla / getsops):**

Encrypts values in YAML/JSON while keeping structure visible. Keys come from KMS, GPG, or age.

```yaml
# secrets.enc.yaml
db_password: ENC[AES256_GCM,data:xxxxx,iv:yyyy,tag:zzzz,type:str]
api_key: ENC[...]
sops:
    kms:
      - arn: arn:aws:kms:us-east-1:123:key/abc
    lastmodified: "2026-01-15T10:00:00Z"
```

```bash
sops -e -i secrets.yaml       # encrypt in place
sops secrets.enc.yaml         # decrypt in $EDITOR
sops -d secrets.enc.yaml      # print decrypted
```

**CI integration:**

```yaml
- name: Decrypt secrets
  env:
    AWS_ROLE_ARN: arn:aws:iam::123:role/ci-sops
  run: sops -d config/secrets.enc.yaml > config/secrets.yaml
```

**2. git-crypt:**

Transparent file-level encryption. You add `*.secret filter=git-crypt diff=git-crypt` to `.gitattributes`; users with the key see plaintext, others see ciphertext.

Best for small repos with a static set of users.

**3. Sealed Secrets (Kubernetes):**

A Bitnami project. Encrypts Kubernetes Secrets with a public key; the controller in-cluster decrypts. The encrypted SealedSecret safely lives in Git.

```bash
kubectl create secret generic db-pass --from-literal=password=hunter2 \
  --dry-run=client -o yaml | \
  kubeseal --controller-namespace kube-system > sealed-db-pass.yaml
git add sealed-db-pass.yaml
```

**Anti-patterns:**

- **Base64 encoding.** Not encryption. Trivially reversed.
- **`.env` files in Git.** Even gitignored, they leak via git history or developer machine exposure.
- **Hard-coded fallbacks.** `os.getenv("SECRET", "my-real-secret")`. Embarrassingly common.

**Detection:**

Scan every commit with `gitleaks`, `trufflehog`, or GitHub's own secret scanning. Configure push protection so pushes with detected secrets are rejected.

```yaml
- uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Interview insight:** for high-security environments, the answer is often "no secrets in Git at all — fetch from Vault at runtime." Mention this as the ideal, with SOPS/Sealed Secrets as pragmatic intermediate solutions.

### Q11. What is a deploy key, and when would you use one?

**Answer:**

A **deploy key** is an SSH key configured on a single repository with read-only (optionally read/write) access. Used for automated systems that need to clone the repo without a user account.

**Typical uses:**

- A production server fetching code at deploy time
- A CI system in a different org pulling a dependency repo
- IoT devices pulling firmware

**Why not use a personal access token?** Tokens are scoped to a user; if the user leaves, tokens die. Deploy keys live with the repository.

**Why not use a bot user?** Bot users with PAT are fine but require a paid seat in most GitHub plans. Deploy keys are free.

**Creation:**

```bash
ssh-keygen -t ed25519 -f deploy_key -N ""
# Upload deploy_key.pub to Repo → Settings → Deploy keys
# Store deploy_key (private) wherever your automation runs
```

**Alternative — GitHub Apps:**

For richer integrations (commit statuses, PR comments, cross-repo access), a **GitHub App** is preferable. Apps authenticate with a JWT and can be installed on multiple repos. They have much finer-grained permissions than deploy keys.

**Trade-offs:**

| Aspect | Deploy key | PAT | GitHub App |
|--------|-----------|-----|------------|
| Scope | One repo | User-wide | Per-repo, granular |
| Lifetime | Repo lifetime | User lifetime | App lifetime |
| Permissions | Clone/push | Everything the user can do | Configured per scope |
| Identity | Ambiguous in audit | Clear (user) | Clear (app) |

**Interview insight:** in modern setups, prefer GitHub Apps or OIDC federation over deploy keys. Deploy keys are still useful for simple cases — especially air-gapped or third-party systems — but they don't scale.

### Q12. How do you handle merge conflicts in long-running feature branches without losing work?

**Answer:**

Ideally, branches live hours, not weeks (see Q2 — trunk-based development). But sometimes — big refactors, security vulnerabilities kept private — a long-lived branch is unavoidable.

**Strategy 1: Rebase frequently.**

```bash
git fetch origin
git rebase origin/main
# Resolve conflicts as they appear
```

Rebasing keeps your branch's history linear on top of `main`. Conflicts surface incrementally, one commit at a time.

**Strategy 2: Merge main into the branch periodically.**

```bash
git merge origin/main
```

Creates merge commits that preserve context. Conflicts resolved at merge time. Easier for novices, but messier history.

**Strategy 3: Use `git rerere` (reuse recorded resolution).**

```bash
git config --global rerere.enabled true
```

Git remembers how you resolved conflicts. The next time the same conflict appears, Git applies the resolution automatically. Useful when you rebase repeatedly against a moving `main`.

**Strategy 4: Stack the work behind feature flags.**

Merge half-complete work to `main` daily, hidden behind a flag. The branch effectively disappears. This is the trunk-based answer — restructure the problem so long-lived branches aren't needed.

**When conflicts occur — process:**

1. **Pull latest main.** `git fetch origin && git rebase origin/main`.
2. **Run the build.** Many "conflicts" are not textual but semantic — the merge succeeds textually but breaks at runtime.
3. **Run the full test suite.** Confirm nothing regressed.
4. **Get a second opinion.** Ask the original author of the conflicting code to review.

**Danger: losing history.**

`git reset --hard` or aggressive rebases can lose commits. Use `git reflog` to recover:

```bash
git reflog
# ... find the commit you lost ...
git reset --hard HEAD@{42}
```

`reflog` entries live 90 days by default.

**Interview insight:** the best answer starts with "I'd avoid long-lived branches in the first place." Tactical advice (rebase, rerere) is secondary. Senior candidates reframe the problem.

---

## Advanced

### Q13. Design a monorepo source-control strategy for a company with 500 services and 300 engineers.

**Answer:**

At this scale, structural decisions outweigh individual workflow choices.

**1. One monorepo vs. polyrepo:**

- **Monorepo** — Google, Meta, Uber. One repo, unified tooling, atomic cross-cutting refactors.
- **Polyrepo** — Amazon (historically), many startups. Separate repos per service, independent CI, easy ownership.

At 500 services / 300 engineers, neither extreme is ideal. Most organisations this size use a **few large repos** (e.g., one per product line) plus shared libraries in their own repos, or a true monorepo with strong tooling investment.

**2. Directory layout for a monorepo:**

```
/services/            # deployable applications
  /api/
  /worker/
  /frontend/
/libs/                # shared libraries
  /shared/
  /ui/
  /proto/             # generated code from .proto
/tools/               # internal developer tooling
/infra/               # terraform, k8s, helm
/docs/
/.github/
/BUILD                # Bazel workspace file
/CODEOWNERS
```

**3. Build system:**

Hand-rolled Make/shell breaks at this scale. Use Bazel (Google), Pants (Twitter), Buck2 (Meta), or Nx/Turborepo (JS-heavy). These provide:

- Fine-grained dependency graph
- Content-addressable remote cache
- `affected` query (`bazel query 'rdeps(//..., //libs/shared)'`)
- Parallel, sandboxed execution

**4. CI architecture:**

```
Git push → webhook → build scheduler
                  → compute affected targets
                  → dispatch N parallel jobs to runner pool
                  → each job hits remote cache → only rebuild misses
                  → aggregate results → post status to PR
```

Self-hosted Kubernetes runner pool with autoscaling. 500-2000 runners at peak. SaaS runners would bankrupt you.

**5. Ownership:**

- CODEOWNERS with per-directory ownership
- Each service has a `service.yaml` with on-call team, SLO, dependencies
- A lint in CI enforces that `service.yaml` matches reality (e.g., paths exist, on-call is a real rotation)

**6. Release process:**

- Trunk-based; every merge to `main` triggers per-service deploy pipelines
- Feature flags for gradual rollout
- Atomic cross-service changes possible (the monorepo win)

**7. Developer experience:**

- `mise` / `asdf` for toolchain management
- Pre-commit hooks (fast subset of CI)
- `devcontainer.json` for consistent IDE setup
- Docs searchable via Backstage or similar

**8. Common failure modes:**

- **Cache correctness bugs.** A stale cache entry returning wrong output is a nightmare. Invest in cache validation tooling.
- **Unstable test ordering.** At scale, flaky tests block everyone. See `testing_in_pipelines.md` Q10-Q14.
- **Clone time.** A multi-GB monorepo takes minutes to clone. Use partial clone (`git clone --filter=blob:none`) and sparse checkout.
- **Mergeability.** Many PRs land per hour. Use merge queues (GitHub Merge Queue, Bors, Mergify) to serialise final validation.

**Interview insight:** at this scale, source control, CI, build system, and deployment are one integrated platform problem. Mentioning merge queues, sparse checkout, and Bazel-remote-cache shows staff-level breadth.

### Q14. What is a merge queue, and why is it essential for high-throughput repos?

**Answer:**

A **merge queue** serialises the final validation before merge. Each PR, after passing review and the first round of CI, enters the queue. The queue rebases each PR onto the latest `main`, runs CI again, and merges only if green.

**Problem it solves:**

Without a merge queue, two PRs can both pass CI against an older `main`, then both merge, leaving `main` broken. With many developers, this happens constantly.

Example scenario without a queue:

```
main: commit A
PR #1: adds function foo()     — tests pass against main@A
PR #2: renames module X → Y    — tests pass against main@A
PR #1 merges → main@B
PR #2 merges → main@C          — but foo() now imports from old X
main is broken
```

**With a merge queue:**

```
main: commit A
PR #1 + main@A → CI passes → merges → main@B
PR #2 rebased on main@B → CI re-runs → fails (import error)
          → PR #2 kicked back to author
```

**Implementations:**

- **GitHub Merge Queue** — native feature, fully integrated with branch protection
- **Bors (bors-ng)** — open-source, origin at Rust project
- **Mergify** — commercial, rich policy language
- **Zuul** — OpenStack's, optimised for large-scale CI

**GitHub Merge Queue configuration:**

In Settings → Branches → `main` protection rules → Require merge queue.

```yaml
# Workflow must declare merge_group trigger
on:
  pull_request:
  merge_group:
```

**Speculative merging (the optimisation that makes this fast):**

A naive queue serialises PRs and incurs a CI run per PR, so 10 PRs = 10× CI time. Speculative merging tests `main + PR1 + PR2 + PR3` in parallel, assuming all pass. If PR2 fails, the queue re-tests `main + PR1 + PR3`. Asymptotically O(log N).

**When you need it:**

| Merges/day | Recommendation |
|------------|----------------|
| < 10 | Manual merging is fine |
| 10-50 | Auto-merge after CI; watch for conflicts |
| 50+ | Merge queue essential |

**Failure mode: queue starvation.**

A slow or flaky test stalls the entire queue. Measure queue wait time and treat CI duration as a first-class metric.

**Interview insight:** merge queues are a 2022+ addition to GitHub. Candidates who have used Bors or equivalents at scale bring real experience. Mention speculative merging if you want to sound sharp.

### Q15. How do you handle third-party dependencies that are also mirrored in the monorepo (vendoring)?

**Answer:**

**Vendoring** means committing copies of third-party dependencies into the monorepo. Rare at the ecosystem level (JS, Python) but universal in Go and common in safety-critical codebases.

**Why vendor:**

1. **Reproducibility.** The code that built this binary is right there, even if the upstream disappears.
2. **Hermetic builds.** No network during build (SLSA Level 3+).
3. **Security review.** All dependencies are physically in the repo, readable, reviewable.
4. **Patching.** Apply local patches to upstream code without forking.
5. **Compliance.** Air-gapped environments often mandate it.

**Trade-offs:**

1. **Repo bloat.** Vendored `node_modules` equivalent is gigabytes.
2. **Update friction.** `dependabot` and `renovate` generate noisy PRs.
3. **Merge conflicts.** Two branches both bump the same dep.

**Go example (native support):**

```bash
go mod tidy
go mod vendor     # writes vendor/ directory
go build -mod=vendor ./...
```

`vendor/` goes in Git. Builds are deterministic and offline-capable.

**Python with vendored wheels:**

```bash
pip download -r requirements.txt -d vendor/wheels --no-binary=:all:
# in CI
pip install --no-index --find-links vendor/wheels -r requirements.txt
```

**Partial vendoring with sparse checkout:**

For monorepos where not every developer needs the vendored tree:

```bash
git clone --filter=blob:none --sparse https://github.com/org/repo
cd repo
git sparse-checkout set services/api libs/shared
```

**CI-side caching of vendored builds:**

Even with vendored deps, building from scratch is slow. Combine vendoring with a remote build cache — deps are reproducible and fast.

**Modern alternative — pinned lockfiles + internal mirror:**

Instead of vendoring, lock everything (poetry.lock, package-lock.json) and proxy the public registry through an internal Nexus/Artifactory. You get reproducibility without 5 GB of third-party code in Git.

**Interview insight:** the answer depends on the environment. Startups: lock + mirror. Defence / critical infrastructure: vendor. Mention both and articulate when each fits.

### Q16. Design a source control workflow for a company that maintains multiple supported versions in parallel (e.g., v2.x, v3.x, v4.x).

**Answer:**

This is one of the legitimate cases for non-trivial branching. You have a development trunk *and* multiple release branches receiving bugfixes.

**Branch model:**

```
main              ← development (future v4.0)
release/v3.x      ← supported release branch
release/v2.x      ← supported release branch (security only)
release/v1.x      ← EOL; no changes accepted
```

**Workflow for a bugfix:**

1. Fix lands on `main` first (most recent truth)
2. Cherry-pick to `release/v3.x` and `release/v2.x` if applicable
3. Each release branch cuts its own tags (`v3.5.1`, `v2.9.3`) and releases independently

**Tooling support:**

**Backporting with gh:**

```bash
gh pr merge 1234 --squash          # merges to main
# Then:
git checkout release/v3.x
git cherry-pick -x <commit-sha>    # -x records origin commit
git push origin release/v3.x
```

**Automated backport labels (Mergify, backport-action):**

```yaml
# .github/workflows/backport.yml
on:
  pull_request_target:
    types: [labeled, closed]
jobs:
  backport:
    if: github.event.pull_request.merged && contains(github.event.pull_request.labels.*.name, 'backport')
    runs-on: ubuntu-latest
    steps:
      - uses: tibdex/backport@v2
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

Label a PR `backport:release/v3.x`, merge it, and a bot opens a corresponding PR against `release/v3.x`.

**CI structure:**

Every release branch has its own CI pipeline, possibly subtly different:

- `release/v3.x` tests against Python 3.9-3.11
- `main` tests against Python 3.11-3.13

Matrix definitions differ per branch.

**Release cadence:**

- `main` → continuous delivery, internal dogfooding
- `release/v3.x` → monthly minor releases, patch as needed
- `release/v2.x` → quarterly patch releases, security-only after v4.0 GA
- `release/v1.x` → EOL — no CI, no releases

**SemVer implications:**

- `main` accumulates breaking changes (future 4.0)
- `release/v3.x` accepts only backwards-compatible changes
- `release/v2.x` accepts only bugfixes and security fixes

See `release_management.md` for deeper SemVer coverage.

**When to end-of-life a branch:**

- Stated support policy (e.g., 18 months after next major)
- Customers have migrated (measured via telemetry / poll)
- Security-only mode for a final period, then fully EOL

**Interview insight:** this is close to a real library interview question. Most candidates know trunk-based, but "how do I support LTS releases?" tests whether you can adapt the model to different product constraints.

### Q17. How do you enforce signed commits and why does it matter?

**Answer:**

**Signed commits** bind a commit to a cryptographic identity (GPG key, SSH key, or X.509 certificate). Git records the signature in the commit object, and verification tools check it.

**Threat model:**

Without signing, `git` accepts any author name in a commit. An attacker who compromises a developer's credentials (or a GitHub token) can push commits claiming to be anyone.

**Enforcement layers:**

1. **Developer side.** Configure Git to sign every commit:

```bash
# GPG
git config --global user.signingkey 0xABCDEF1234567890
git config --global commit.gpgsign true

# Or SSH-based (since Git 2.34)
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

2. **Platform side.** GitHub/GitLab verify signatures and display "Verified" badges.

3. **Branch protection.** "Require signed commits" rejects unsigned pushes.

4. **CI verification.** Runners can verify signatures as part of the pipeline:

```yaml
- name: Verify commit signatures
  run: |
    git log --pretty='format:%H %G?' main..HEAD | awk '$2 != "G" { exit 1 }'
```

**Why `%G?` matters:**

- `G` — good signature, valid key
- `B` — bad signature
- `U` — good signature, unknown validity
- `N` — no signature
- `E` — can't check (missing key)

Only `G` is trustworthy.

**Sigstore's gitsign:**

Modern keyless signing. Developers sign with their OIDC identity (Google, GitHub); signatures are backed by Fulcio certs and Rekor transparency log. No long-lived keys to rotate or lose.

```bash
gitsign init-config
git config --global commit.gpgsign true
git config --global gpg.x509.program gitsign
git config --global gpg.format x509
```

**Verification:**

```bash
gitsign verify <commit-sha>
```

**Practical considerations:**

- **Merge commits lose author signing.** GitHub's web UI signs on your behalf with its own key ("Verified by GitHub"). Some orgs disable web merging.
- **Cherry-picks re-author.** Signatures don't survive cherry-pick; the cherry-picker signs the new commit.
- **Bot commits need identities.** Dependabot and Renovate sign with their own keys.
- **Key loss.** Developers losing keys lock themselves out. Provide recovery paths and consider keyless (gitsign).

**Interview insight:** signed commits show up in supply chain security discussions (SolarWinds aftermath, SLSA, FedRAMP). Candidates who mention Sigstore demonstrate current awareness.

### Q18. Design a workflow for cross-repository changes (e.g., an API change in one repo that requires consumer updates in three others).

**Answer:**

The classic monorepo argument is "we can make atomic cross-cutting changes." In polyrepo land, coordinating changes across repos requires discipline.

**Strategy 1: Expand-contract (expand, migrate, contract).**

For a breaking API change:

1. **Expand:** Release v1.1 of the API repo that supports both old and new behaviour.
2. **Migrate:** Update each consumer repo to the new behaviour. Each merges independently.
3. **Contract:** Release v2.0 of the API repo, removing old behaviour, once all consumers have migrated.

This decouples the changes in time. No atomic cross-repo merge needed.

**Strategy 2: Coordinated PRs (contracts in test).**

1. Open PRs simultaneously in API, consumer A, consumer B, consumer C.
2. API PR publishes a pre-release artefact (`v1.2.0-rc.1`).
3. Consumer PRs pin to that pre-release and run their CI.
4. All four PRs merge within the same release window.
5. API releases `v1.2.0` final; consumers bump to non-pre-release version.

This is fragile at scale but works when coordinated teams are already in close communication.

**Strategy 3: Internal mono-repo for the API + clients.**

If the API and its N clients are owned by the same organisation, move them into one repo for this class of change. Polyrepo for outside-the-org, monorepo for inside.

**Tooling patterns:**

**Pre-release publishing:**

```yaml
# .github/workflows/publish.yml
on:
  pull_request:

jobs:
  publish-prerelease:
    if: github.event.pull_request.head.repo.fork == false
    steps:
      - uses: actions/checkout@v4
      - run: |
          VERSION="1.2.0-pr.${{ github.event.pull_request.number }}"
          npm version $VERSION --no-git-tag-version
          npm publish --tag pr-${{ github.event.pull_request.number }}
```

Consumer PRs can then install `@company/api@pr-1234`.

**Repository dispatch (coordinating CI across repos):**

```yaml
# in API repo, after merge
- uses: peter-evans/repository-dispatch@v3
  with:
    token: ${{ secrets.CROSS_REPO_PAT }}
    repository: org/consumer-a
    event-type: api-updated
    client-payload: '{"version": "${{ needs.release.outputs.version }}"}'
```

Consumer repos listen for `api-updated` events and open a bump PR automatically.

**Pitfalls:**

- **Deadlock.** PR A needs PR B to merge first; PR B needs PR A. Break the cycle with expand-contract or feature flags.
- **Stale branches.** A consumer PR opened six months ago pinning `1.2.0-rc.1` is now drift. Add a freshness check.
- **Rollback complexity.** Rolling back a cross-repo change affects N repos. The simpler the individual revert, the better.

**Interview insight:** this question exposes candidates who've only worked in monorepos or only in polyrepos. The mature answer is "it depends, here are the patterns". Name specific tooling — repository_dispatch, pre-release tags, expand-contract — to ground your answer.
