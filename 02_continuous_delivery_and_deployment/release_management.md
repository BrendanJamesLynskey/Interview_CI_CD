# Release Management — Interview Questions

**Subject:** CI/CD
**Topic:** Semantic Versioning, Changelogs, Release Gates, Rollback Strategies
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is Semantic Versioning (SemVer), and what do the numbers mean?

**Answer:**

**Semantic Versioning** (semver.org) is a versioning convention with the format `MAJOR.MINOR.PATCH`:

- **MAJOR** — incompatible API changes (breaking)
- **MINOR** — new functionality, backwards-compatible
- **PATCH** — backwards-compatible bug fixes

Example progression:

```
1.0.0 → first stable release
1.0.1 → bug fix
1.1.0 → new feature (backwards-compatible)
2.0.0 → breaking change
```

**Pre-release and build metadata:**

```
1.0.0-alpha.1        pre-release
1.0.0-beta           pre-release
1.0.0-rc.1           release candidate
1.0.0+20260315.sha   build metadata (ignored by version comparison)
```

**Ordering:** pre-releases come before the released version:

```
1.0.0-alpha < 1.0.0-beta < 1.0.0-rc.1 < 1.0.0
```

**Key rules:**

1. Once a version is released, never re-release it with different content.
2. MAJOR of 0 means "under initial development; anything may change."
3. After 1.0.0, any MAJOR bump signals breaking changes — users can depend on this.

**Interview insight:** the value of SemVer isn't the number format; it's the *social contract* it creates. Users can depend on `^1.2.3` (all 1.x versions) safely, knowing only PATCH and MINOR can change. Violate this, and users lose trust.

### Q2. What is a changelog, and why is it important?

**Answer:**

A **changelog** is a human-readable document that records the notable changes in each version of a project.

**Format — "Keep a Changelog" (keepachangelog.com):**

```markdown
# Changelog

## [Unreleased]

## [1.2.0] - 2026-03-15
### Added
- New `--parallel` flag for `run` command.
- Support for PostgreSQL 16.

### Changed
- `export` output format now includes schema metadata.

### Deprecated
- `--legacy-format` flag (will be removed in 2.0.0).

### Fixed
- Crash when processing empty input files (#1234).

### Security
- Upgraded `libfoo` to 2.3.1 to fix CVE-2026-12345.

## [1.1.0] - 2026-02-01
...
```

**Why it matters:**

1. **Users upgrading need to know what changed.** "What's in this version?" should be answerable in 30 seconds.
2. **Incident response.** When a regression is suspected, the changelog narrows the search.
3. **Marketing and product.** Release notes for the customer are often a filtered changelog.
4. **Compliance and audit.** Regulators (FDA, FAA) want change records per release.

**Machine-readable alternative: Conventional Commits.**

If every commit follows a convention (`feat:`, `fix:`, `chore:`), tooling generates the changelog automatically:

```
feat(auth): support OAuth2 device flow
fix(billing): handle negative invoice totals
chore(deps): bump axios to 1.7.5
feat!: remove deprecated --legacy-format flag  # ! = breaking change
```

**Tools:**

- **git-cliff**, **release-please**, **semantic-release** — parse Conventional Commits, generate CHANGELOG.md, bump version
- **GitHub auto-generated release notes** — configurable via `.github/release.yml`

**`.github/release.yml` example:**

```yaml
changelog:
  categories:
    - title: Breaking changes
      labels: [breaking]
    - title: New features
      labels: [feature]
    - title: Bug fixes
      labels: [bug]
    - title: Dependencies
      labels: [dependencies]
```

**Interview insight:** the machine-readable + human-curated hybrid is the modern sweet spot. Generate from commits; tidy for humans; publish as part of every release.

### Q3. What are release gates, and what gates would you include in a production pipeline?

**Answer:**

A **release gate** is a check that must pass before a deployment advances to the next stage. Gates enforce policy, quality, and compliance.

**Common gates, in order:**

| Gate | Purpose | Blocker? |
|------|---------|----------|
| **Unit tests** | Functional correctness | Yes |
| **Linting / static analysis** | Code quality, style | Yes |
| **Type checking** | Type safety | Yes |
| **Security scan (SAST)** | Known code vulnerabilities | Yes |
| **Dependency scan (SCA)** | Known library CVEs | Yes |
| **Container scan** | Base image vulnerabilities | Yes |
| **Licence compliance** | GPL-banned, licence inventory | Yes |
| **Integration tests** | Cross-component correctness | Yes |
| **Performance regression** | P99 > threshold | Yes or warning |
| **Manual approval** | Human gate for prod | Yes (prod only) |
| **Canary health** | SLIs in acceptable range | Yes |
| **Error budget** | Budget remaining > threshold | Yes (optional policy) |
| **Change freeze** | Within allowed deploy window | Yes |

**GitHub Actions example with multiple gates:**

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make test

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: registry/app:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: 1
      - uses: github/codeql-action/analyze@v3
      - run: snyk test --severity-threshold=high

  deploy-staging:
    needs: [test, security]
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: ./deploy.sh --env=staging

  e2e:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - run: make e2e

  deploy-prod:
    needs: e2e
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    # Environment protection rules require approval
    steps:
      - run: ./deploy.sh --env=prod --mode=canary
```

**GitHub Environment protection rules** provide:

- Required reviewers (manual gate)
- Wait timer (bake time)
- Branch restrictions (only `main` deploys to prod)

**Designing gates — principles:**

1. **Fast gates first.** Lint before integration tests before manual approval.
2. **Every gate must have a clear failure signal.** "Gate failed because X" — not "something's wrong."
3. **Avoid spurious gates.** A flaky test gate erodes trust; it'll be bypassed.
4. **Gates at every stage.** Not just before deploy; gates between deploy and release, and between canary stages.

**Interview insight:** name a specific set of gates and explain the rationale for each. Candidates who say "we have CI" without enumerating show they haven't thought through it.

### Q4. What is the difference between a rollback, a roll-forward, and a revert?

**Answer:**

All three undo the effect of a bad change, but differently.

**Rollback:**

Deploy the previous version's artefact. The change is un-done by going backwards.

```bash
kubectl rollout undo deployment/app
# or re-apply manifest pointing to previous image tag
```

**Pros:** Fast. The previous version is known-good.
**Cons:** Irreversible schema changes don't roll back cleanly. "Previous state" may not be clearly defined in complex systems.

**Roll-forward:**

Commit a fix on top of the bad change and deploy *forward*. The change remains in history; a follow-up fixes it.

```bash
git commit -m "Fix regression from v1.5.0"
git push
# CI/CD promotes the fix through normal pipeline
```

**Pros:** Works when rollback is impossible (schema changes). Preserves forward progress. No "revert of revert of revert" history.
**Cons:** Takes longer — must write fix, test it, deploy through full pipeline. Users see broken behaviour for longer.

**Revert (git revert):**

A git-level undo: create a new commit that is the inverse of the bad commit.

```bash
git revert <bad-sha>
git push
```

**Pros:** Preserves history. Clean audit trail. Standard practice for trunk-based teams.
**Cons:** Requires the inverse to be meaningful. Some changes can't be automatically reverted (merge commits, changes already built upon).

**When to use which:**

| Scenario | Choice |
|----------|--------|
| Outage, new code at fault | Rollback (fast) |
| Schema migration already applied | Roll-forward |
| Merged commit that shouldn't have been | Revert + redeploy |
| Partial failure on one region | Rollback that region |
| Configuration error | Rollback or revert the config commit |

**Interview insight:** don't conflate them. A senior engineer distinguishes "un-deploy" (rollback), "apply fix" (roll-forward), and "undo in git" (revert). An operations runbook uses all three at different points.

### Q5. How do you version a container image?

**Answer:**

Container images have two addressability mechanisms: **tags** (mutable) and **digests** (immutable).

**Tag conventions:**

```
registry.example.com/myapp:latest          # worst — what is "latest" pointing at?
registry.example.com/myapp:v1.5.0          # SemVer tag — good for releases
registry.example.com/myapp:1.5             # floating minor — users get latest 1.5.x
registry.example.com/myapp:1               # floating major — gets latest 1.x.x
registry.example.com/myapp:main            # tracks branch — changes over time
registry.example.com/myapp:pr-1234         # per-PR, for dev/review environments
registry.example.com/myapp:abc1234         # git short SHA — pinned to commit
registry.example.com/myapp:v1.5.0-abc1234  # version + SHA — unique and traceable
```

**Digest (immutable):**

```
registry.example.com/myapp@sha256:2f7a2f05e56a81db82bf9e8c4ac6c8cbf6a0a9e9f2e8b6b4fcaf0c8b5f72aebd
```

Digests are content-addressed. They never change — same digest always pulls the same image.

**Best practice: tag multiple ways, pin by digest.**

When building:

```bash
DIGEST=$(docker buildx build \
  -t registry/myapp:v1.5.0 \
  -t registry/myapp:1.5 \
  -t registry/myapp:abc1234 \
  --push \
  --metadata-file=metadata.json \
  . )
IMAGE_DIGEST=$(jq -r '.["containerimage.digest"]' metadata.json)
```

When deploying:

```yaml
# In Kubernetes manifest — use digest to guarantee exact bits
containers:
  - name: app
    image: registry/myapp@sha256:2f7a2f05e56a81db82bf9e8c4...
```

**Why digest pinning matters:**

- A tag can be re-pushed. `myapp:1.5` pointed at digest A; someone pushes a new 1.5 build; digest is now B. Anything that pulled A has silently diverged.
- Supply chain attacks modify tags, leaving the attacker-controlled image live.
- Reproducibility — a manifest that pins a digest will always deploy the same bits.

**Version tagging in CI:**

```yaml
- name: Set version tags
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: registry/myapp
    tags: |
      type=semver,pattern={{version}}      # v1.5.0
      type=semver,pattern={{major}}.{{minor}}  # 1.5
      type=semver,pattern={{major}}        # 1
      type=ref,event=branch                # main
      type=ref,event=pr                    # pr-1234
      type=sha,prefix=,format=short        # abc1234

- uses: docker/build-push-action@v5
  with:
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
```

**Interview insight:** senior candidates mention digest pinning. It's the foundation of SLSA provenance and reproducibility.

### Q6. What is a release candidate (RC), and how is it used?

**Answer:**

A **release candidate** is a pre-release build considered ready to ship unless blocking issues surface.

**Lifecycle:**

1. Cut `v2.0.0-rc.1` from the release branch
2. Deploy to staging / pre-prod
3. Run full test suite, performance tests, UAT
4. If bugs found: fix, cut `v2.0.0-rc.2`
5. If RC is stable for N days/weeks: re-tag as `v2.0.0` final

**Trunk-based alternative:**

Teams with continuous deployment rarely have explicit RCs. Every commit to `main` is a candidate; promotion to production happens via canary, not via RC tags. "RC" is a release-train artefact.

**When RCs make sense:**

- Library releases used by downstream consumers who want to validate
- Major releases (2.0.0) worth external testing
- Regulated software where a formal UAT phase is required
- Open source projects inviting community testing

**Process example:**

```bash
# Feature branch reaches main; we cut release branch
git checkout -b release/2.0 main
git tag v2.0.0-rc.1
git push origin release/2.0 v2.0.0-rc.1

# CI picks up tag, builds RC artefact
# Deploy to staging, run tests, invite beta users

# Bug found; fix on main (if applicable) and cherry-pick
git cherry-pick -x <fix-sha>
git tag v2.0.0-rc.2
git push origin v2.0.0-rc.2

# RC is good after a week
git tag v2.0.0
git push origin v2.0.0
```

**Semver RC ordering:**

```
2.0.0-rc.1 < 2.0.0-rc.2 < 2.0.0-rc.10 (if numeric) < 2.0.0
```

Note: `rc.10` > `rc.2` because comparison is numeric when both sides are digits.

**CI integration:**

```yaml
on:
  push:
    tags:
      - "v*.*.*"
      - "v*.*.*-rc.*"

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make build
      - name: Publish (pre-release if RC)
        run: |
          gh release create ${{ github.ref_name }} \
            --generate-notes \
            --prerelease=${{ contains(github.ref_name, '-rc.') }}
```

**Interview insight:** mention RCs as a release-train pattern and contrast with continuous deployment. Don't present RC as universal best practice — it's context-dependent.

---

## Intermediate

### Q7. Design a fully automated release process using Conventional Commits and semantic-release.

**Answer:**

A fully automated release process uses commit messages to drive version bumps, changelogs, and publishing.

**Workflow:**

1. Every commit follows **Conventional Commits**.
2. On merge to `main`, CI analyses commits since last release.
3. Calculates the appropriate version bump (major/minor/patch).
4. Generates the CHANGELOG.
5. Tags the commit.
6. Publishes artefacts and creates a GitHub release.
7. Posts release notes to Slack/Discord/email.

**Commit convention:**

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

Types and their effect on version:

| Type | Version bump |
|------|--------------|
| `fix:` | PATCH |
| `feat:` | MINOR |
| `feat!:` or `fix!:` or `BREAKING CHANGE:` in footer | MAJOR |
| `chore:`, `docs:`, `style:`, `refactor:`, `test:`, `ci:` | No bump |

**Examples:**

```
fix(auth): handle expired refresh tokens gracefully
feat(api): add /v2/users endpoint with pagination
feat(api)!: remove deprecated /v1/users endpoint

BREAKING CHANGE: /v1/users no longer exists; use /v2/users
```

**GitHub Actions with semantic-release:**

```yaml
name: Release
on:
  push:
    branches: [main]

permissions:
  contents: write
  issues: write
  pull-requests: write
  id-token: write  # for npm provenance / OIDC

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
      - run: npm test
      - run: npm run build
      - name: Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: npx semantic-release
```

**`.releaserc.yaml`:**

```yaml
branches:
  - main
  - name: alpha
    prerelease: true
plugins:
  - - "@semantic-release/commit-analyzer"
    - preset: conventionalcommits
  - - "@semantic-release/release-notes-generator"
    - preset: conventionalcommits
  - - "@semantic-release/changelog"
    - changelogFile: CHANGELOG.md
  - "@semantic-release/npm"
  - - "@semantic-release/git"
    - assets: [CHANGELOG.md, package.json]
      message: "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
  - "@semantic-release/github"
```

**Alternative: release-please (Google).**

Better for Go, Python, Rust, and monorepos. Opens a PR with version bump + changelog; merging the PR triggers the release.

```yaml
- uses: googleapis/release-please-action@v4
  with:
    release-type: python
    package-name: myapp
```

**Benefits:**

1. **Consistency.** Every release follows the same process. No human forgot to update changelog.
2. **Speed.** Merge to main → release in minutes.
3. **Correctness.** Version bumps correctly classified — no accidental PATCH when API changed.

**Trade-offs:**

1. **Requires commit discipline.** One bad commit without proper prefix = wrong version bump. Enforce with `commitlint` in PR checks.
2. **Less control over timing.** If you want to bundle multiple features into a "big release," automation gets in the way.
3. **Monorepos need per-package logic.** Use release-please or tooling that supports multi-package repos (Nx, Turborepo, Changesets).

**Interview insight:** semantic-release has been around since 2015; it's well-understood. Mention it as a concrete tool, plus commitlint to enforce Conventional Commits in PRs.

### Q8. What is a rollback-safe migration, and how do you design one?

**Answer:**

A **rollback-safe migration** can be undone without data loss if the deployment that introduced it fails.

**Key insight:** the migration itself and the deployment that uses it must be decoupled. The migration lands *before* the code that needs it and is *backwards-compatible* with the previous code.

**Expand-contract pattern (see `deployment_strategies.md` Q7):**

1. **Expand** — add new schema, make it optional, keep old schema.
2. **Dual-write** — code writes to both; reads from old.
3. **Migrate reads** — code reads from new (still writes both).
4. **Stop dual-writes** — only new schema touched.
5. **Contract** — drop old schema.

Each step is independently deployable and rollback-safe.

**Example — adding a not-null column:**

**Bad (not rollback-safe):**

```sql
-- Migration v1 (irreversible — old code can't insert without the column)
ALTER TABLE orders ADD COLUMN channel VARCHAR(50) NOT NULL DEFAULT 'web';
```

If the deploy fails and you roll back the code, old code inserts without `channel` — fails on NOT NULL even with default (PostgreSQL handles this, MySQL < 8 might not).

**Good (rollback-safe):**

```sql
-- Migration v1: expand
ALTER TABLE orders ADD COLUMN channel VARCHAR(50);

-- Deploy code v2: writes and reads channel (nullable still)
-- Verify in production for a week

-- Migration v2: backfill + enforce
UPDATE orders SET channel = 'web' WHERE channel IS NULL;
ALTER TABLE orders ALTER COLUMN channel SET NOT NULL;

-- Deploy code v3: treats channel as required
```

**Reversible migration example (Alembic):**

```python
def upgrade():
    op.add_column('orders', sa.Column('channel', sa.String(50), nullable=True))

def downgrade():
    op.drop_column('orders', 'channel')
```

The `downgrade` is the inverse. If code deploy fails after upgrade, `downgrade` cleanly reverts.

**Irreversible-in-practice operations:**

- **DROP COLUMN** — data lost forever.
- **DELETE large rows** — can't undo.
- **ALTER COLUMN TYPE when not round-trip safe** (e.g., truncating varchar).
- **Partitioning operations** that restructure storage.

For these, the migration itself is one-way. Plan accordingly:

1. Run the migration only when confident.
2. Back up the data first (`pg_dump`, snapshot).
3. Split into smaller reversible steps if possible.
4. Be explicit: label the migration "IRREVERSIBLE" so reviewers know.

**Migration as a separate step in CD:**

```yaml
jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run migration
        run: ./scripts/migrate.sh up
      - name: Verify migration
        run: ./scripts/verify_schema.sh

  deploy:
    needs: migrate
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/deploy.sh
```

If deploy fails, migrate down:

```yaml
  rollback:
    if: failure()
    needs: deploy
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/deploy.sh --version=previous
      - run: ./scripts/migrate.sh down 1
```

**Interview insight:** this is one of the most asked migration questions. Strong answers mention expand-contract explicitly, distinguish reversible from irreversible operations, and mention tooling (Alembic, Flyway, Liquibase).

### Q9. How do you decide when to cut a release, and how often should you release?

**Answer:**

"When to release" depends on the product and the organisation.

**Event-driven release:**

Release when a feature is complete, or when a bug is critical enough to ship.

- Small teams, high urgency.
- Every release is intentional; no spurious releases.
- Hard to coordinate across teams.

**Time-driven release (release train):**

Fixed cadence: weekly, fortnightly, monthly. Whatever is in `main` at the cut goes out.

- Predictable for downstream consumers.
- Features miss a train and wait for the next.
- Easier to coordinate docs, comms, QA.

**Continuous release:**

Every merge to `main` is released. No "cutting" — the release is each commit.

- Requires high confidence in automated tests and rollback.
- Decouples release from marketing events (via feature flags).
- Smallest change per release = smallest blast radius per change.

**Release frequency recommendations, by product type:**

| Product | Typical cadence | Rationale |
|---------|----------------|-----------|
| Web SaaS | Multiple per day | Users don't install; feature flags enable gradual exposure |
| Mobile app | Weekly to monthly | App store review, user install cadence |
| On-prem SaaS | Monthly to quarterly | Customers install updates on their schedule |
| Embedded / firmware | Quarterly | Hardware coordination, field upgrades costly |
| Desktop software | Monthly to quarterly | Auto-updater can smooth this |
| OSS library | Event-driven | Release when meaningful change accrues |

**DORA metrics and deployment frequency:**

DORA research categorises teams by deployment frequency:

- **Elite:** Multiple times per day
- **High:** Once per day to once per week
- **Medium:** Once per week to once per month
- **Low:** Less often than monthly

High-performing teams ship more often. Correlation, not causation — frequent deploys require good practices, and good practices enable frequent deploys.

**Signals it's time to cut a release:**

- Feature completion plus customer demand
- Critical bug fix ready
- Security vulnerability addressed
- Regular cadence reached (Tuesday for weekly)
- Error budget healthy, canary signals clean

**Signals to hold release:**

- Error budget exhausted
- Ongoing incident
- Change freeze (holiday, Black Friday)
- Major dependency just broke downstream
- Insufficient on-call coverage (Friday evening?)

**Interview insight:** the answer is context-dependent. The mature position: "I aim for the shortest-safe cadence. If we can safely deploy hourly, we do. If the context limits us to monthly, I make each release as small as possible within that constraint."

### Q10. What is a feature freeze, and how do you implement one?

**Answer:**

A **feature freeze** is a period during which only bug fixes and stabilisation changes are allowed into a branch. New features are held back.

**Typical uses:**

1. **Before a major release.** Cut a release branch at feature freeze; only bug fixes go in; release from stable branch.
2. **Holiday / high-risk periods.** Black Friday, year-end. No risky changes near peak traffic.
3. **Pending compliance audit.** Code stability during certification (ISO, SOC2).
4. **Incident response.** After a major outage; freeze change until root cause addressed.

**Implementation:**

**Branch-based freeze:**

Cut a `release/vX.Y` branch at freeze time. Only PRs labelled `bugfix` can target this branch. Features continue merging to `main` for the next release cycle.

**Label-based freeze with bot:**

```yaml
# .github/workflows/freeze-check.yml
name: Feature freeze check
on:
  pull_request:
    branches: [main]

jobs:
  check-freeze:
    runs-on: ubuntu-latest
    steps:
      - name: Check if in freeze window
        run: |
          if [ -f .freeze ]; then
            LABELS='${{ join(github.event.pull_request.labels.*.name, ',') }}'
            if [[ "$LABELS" != *"bugfix"* && "$LABELS" != *"freeze-exempt"* ]]; then
              echo "In feature freeze. PR must be labelled 'bugfix' or 'freeze-exempt'."
              exit 1
            fi
          fi
```

Maintainers add / remove the `.freeze` file to toggle the gate.

**Freeze-exempt approval workflow:**

```yaml
# Rules: only the release manager can add the `freeze-exempt` label.
# The label grants exemption; audit trail shows who approved.
```

**Comms:**

A freeze is a cultural decision. Announce it in Slack, update the README, put a banner on the PR template:

```markdown
<!-- PR_TEMPLATE.md -->
## Feature freeze active until 2026-03-20

New features must be labelled `freeze-exempt` and require release-manager approval.
Bug fixes and security patches proceed as normal.
```

**Over-used anti-pattern:**

Perpetual freezes. If every time is "near a release" or "high-risk," teams route around the freeze (merge to a side branch, un-merge later). Freezes should be short and rare.

**Alternative: feature flags.**

If you trust your deployment process, freezes may not be necessary. Land features behind flags; don't release them during the freeze window. The code is merged, the feature isn't live.

**Interview insight:** freezes reveal tension between speed and risk. Candidates who articulate the trade-off (and prefer feature flags over freezes where possible) show senior judgment.

### Q11. How do you handle the release of multiple interdependent libraries in a monorepo?

**Answer:**

A monorepo with `lib-a` that depends on `lib-b` that depends on `lib-c` needs coordinated versioning.

**Two main approaches:**

**1. Fixed versioning (all same version):**

Every package in the monorepo shares the same version number. Release bumps everything.

```
packages/
  lib-a/  v1.5.0
  lib-b/  v1.5.0
  lib-c/  v1.5.0
```

If `lib-c` gets a bug fix, everything bumps to `v1.5.1`, even though `lib-a` and `lib-b` are unchanged.

**Pros:** Simple. Consumers know "use everything at v1.5.0."
**Cons:** Version numbers don't match individual library change history.

Used by: React (all packages at 18.2.0), Angular, Next.js.

**2. Independent versioning (per package):**

Each package versions separately based on its changes.

```
packages/
  lib-a/  v2.3.1
  lib-b/  v1.8.4
  lib-c/  v0.9.2
```

**Pros:** Versions reflect actual change history per package. Consumers can pin precisely.
**Cons:** More bookkeeping. Changesets / tooling required.

Used by: most modern monorepos — Babel, Jest, etc.

**Tooling:**

**Changesets (JavaScript):**

```bash
# Developer adds a changeset with every PR
npx changeset
# Prompts: which packages, what kind of bump, summary
```

Creates `.changeset/random-name.md`:

```markdown
---
"@org/lib-a": minor
"@org/lib-b": patch
---

Added new `parse` option. Fixed edge case in serialization.
```

On merge:

- `changeset version` — consumes all pending changesets, bumps versions, updates CHANGELOGs
- `changeset publish` — publishes all bumped packages to npm

**release-please (Google, multi-language):**

```yaml
release-please-config.json:
{
  "packages": {
    "packages/lib-a": { "release-type": "node" },
    "packages/lib-b": { "release-type": "node" },
    "packages/lib-c": { "release-type": "node" }
  }
}
```

Opens a release PR per package (or combined); merging triggers the release.

**Dependency cascade:**

If `lib-c` bumps patch → `lib-b` (depending on `lib-c`) may or may not need a bump. Changesets lets you declare this:

```markdown
---
"@org/lib-c": patch    # bug fix
"@org/lib-b": patch    # re-release with updated lib-c
---
```

**Release order:**

Changesets and similar tools handle this. Publish in topological order:

```
Step 1: Publish lib-c@1.0.1
Step 2: Publish lib-b@1.8.4 (depends on lib-c@1.0.1)
Step 3: Publish lib-a@2.3.1 (depends on lib-b@1.8.4)
```

If published in the wrong order, `lib-a` would pin `lib-b` that hasn't been published — broken.

**Internal version ranges:**

Prefer `^1.8.0` inside the monorepo — consumers get latest compatible. Some teams use `workspace:*` (pnpm) to always use local during dev, swap for real version on publish.

**Interview insight:** mention Changesets or release-please by name. Discuss the "fixed vs. independent" choice. Bring up topological publish order — a common footgun.

---

## Advanced

### Q12. Design a rollback strategy for a service with complex external dependencies (queues, third-party APIs, databases).

**Answer:**

Rolling back a service affects more than its own state.

**Taxonomy of external effects:**

1. **Messages written to queues.** Old code might not understand messages produced by new code.
2. **Calls to third-party APIs.** New code might have called an API with a side effect (payment, email). Can't un-send.
3. **Database state changes.** Schema or data changes that old code doesn't grok.
4. **Cache entries.** Format changes between versions can poison a shared cache.
5. **Scheduled jobs.** New version scheduled tasks; old version doesn't know about them.

**Strategies per effect:**

**Queue messages:**

- **Backwards-compatible messages.** Every message format is additive; old code ignores unknown fields. This is the main defence.
- **Version in every message.** Consumer checks version; ignores or dead-letters unknown versions.
- **Quarantine queue for incompatible messages.** On rollback, dump new-version messages to a quarantine queue; process them later.

```python
# Producer (always includes version)
message = {"version": 2, "order_id": "abc", "channel": "mobile", "new_field": "x"}

# Consumer (tolerates unknown fields)
def handle(msg):
    if msg["version"] > 2:
        dead_letter(msg, reason="Unknown message version")
        return
    process_v2(msg)
```

**Third-party API calls:**

- **Idempotency.** Every API call has a key; retries don't duplicate.
- **Sagas + compensating actions.** If the deploy fails after an external side effect, run a compensating action (refund the charge, cancel the shipment).
- **Outbox pattern.** Write intent to an outbox table; a separate worker makes the API call. On rollback, undo the outbox intent.

**Database state:**

- Rollback-safe migrations — see Q8.
- Forward-only migrations with forward-only code — if you can't roll back the schema, you can't roll back the code either; plan to roll-forward.

**Cache entries:**

- Version cache keys (`cache:v2:user:123`). Rollback goes back to `cache:v1:*`. Old cache entries persist separately.
- Short TTLs reduce the poison window.
- On rollback, explicitly flush the v2 cache namespace.

**Scheduled jobs:**

- New scheduled jobs should be gated by feature flags or version checks.
- Job schedulers should handle "unknown job type" gracefully (log + skip, not crash).

**Rollback checklist — concrete example:**

```
ROLLBACK PROCEDURE — order-service

1. Flip feature flag: new_checkout_flow = off
   - Controls user-facing behaviour; fastest revert
2. Roll back deployment to previous image
   - kubectl rollout undo deployment/order-service
3. Quarantine any v2-format messages in the queue
   - scripts/quarantine_v2_messages.sh
4. Flush v2 cache namespace
   - redis-cli DEL $(redis-cli KEYS 'cache:v2:*')
5. Verify schema is at version N (pre-deploy migration level)
   - If schema was migrated: roll-forward required (see IRREVERSIBLE note)
6. Notify downstream consumers
   - Slack #order-service-consumers, status page
7. Run reconciliation job
   - scripts/reconcile_orders.sh --since=$DEPLOY_TIME
```

**Post-rollback analysis:**

Every rollback triggers a blameless post-mortem:

- Why did the deploy fail?
- Did any side effects reach external systems?
- Is the system in a correct state?
- What do we change so this doesn't happen again?

**Interview insight:** this question separates "I've deployed" from "I've been on-call." Candidates who name the outbox pattern, saga compensation, and versioned cache keys show they've dealt with real distributed rollbacks.

### Q13. How do you design a release process for software that runs on customer premises (on-prem)?

**Answer:**

On-prem deployment inverts the usual model: you don't control when the customer upgrades, and you must support a wide matrix of versions in the wild.

**Key differences from SaaS:**

| Aspect | SaaS | On-prem |
|--------|------|---------|
| Control over upgrade time | Yes | No |
| Versions in the wild | 1-2 (canary + prod) | 10+ |
| Rollback | Fast, centralised | Customer-driven |
| Telemetry | Full | Limited / opt-in |
| Environment variability | Low (known stack) | High (customer infra) |

**Release pipeline for on-prem:**

**1. Full matrix CI.**

Test against the supported combinations:

```yaml
strategy:
  matrix:
    os: [ubuntu-20.04, ubuntu-22.04, rhel-8, rhel-9, windows-2022]
    db: [postgres-14, postgres-15, postgres-16, mysql-8]
    k8s: [1.27, 1.28, 1.29, 1.30]
```

That's 5 × 4 × 4 = 80 combinations. Too many? Sample strategically; add specific combinations customers actually run.

**2. Release artefacts, not deployments.**

Build:
- Docker images (to customer registry or published)
- Helm charts
- Debian/RPM packages
- Windows MSI
- Binary tarballs

Sign every artefact. Generate SBOMs (see `supply_chain_security.md`).

**3. Upgrade path testing.**

Critical: test not just "install v1.5.0" but "upgrade from v1.4.x to v1.5.0." For each supported previous version:

```yaml
jobs:
  upgrade-test:
    strategy:
      matrix:
        from: ["1.4.0", "1.4.5", "1.4.10", "1.3.8"]
    steps:
      - name: Install previous
        run: ./install.sh ${{ matrix.from }}
      - name: Load realistic data
        run: ./load_fixtures.sh
      - name: Upgrade to current
        run: ./upgrade.sh $CURRENT_VERSION
      - name: Run acceptance tests
        run: ./test.sh
```

**4. Release channels.**

Offer channels with different stability guarantees:

- **stable** — recommended, widely tested
- **LTS** — long-term support, security patches only
- **beta** — next version, opt-in testers
- **nightly** — dev builds, not for prod

Customers choose their channel.

**5. Version support policy.**

Publish and stick to it:

```
Versions:
- Current: 1.6.x — full support
- Previous minor: 1.5.x — full support
- Two minors back: 1.4.x — security only
- Older: EOL, no updates
```

Enforce in CI: if a PR targets `release/1.3`, it must be labelled `security`.

**6. Telemetry and usage data.**

Opt-in product telemetry that reports version distribution. You need to know:

- What versions are in use
- Which features are exercised
- Error rates per version

This informs when you can end-of-life an old version.

**7. Release comms.**

On-prem customers need:

- Release notes (marketing-friendly, not dev-oriented)
- Migration guides for breaking changes
- Upgrade runbooks
- Known-issues lists
- Security advisories

**8. Customer-specific hotfixes.**

A large customer hits a critical bug in 1.4.x. You:

- Fix on main (future 1.6.x)
- Cherry-pick to 1.5.x (next supported)
- Cherry-pick to 1.4.x if they can't upgrade quickly
- Build hotfix artefact `1.4.10-hotfix.1`, sign, ship to that customer

**Pipeline structure:**

```yaml
# .github/workflows/release.yml
on:
  push:
    tags: ["v*.*.*", "v*.*.*-lts.*"]

jobs:
  matrix-test: { ... }
  upgrade-test: { ... }
  build-artefacts: { ... }
  sign:
    needs: build-artefacts
    steps:
      - run: cosign sign-blob --key /secrets/cosign.key release.tar.gz
  sbom:
    steps:
      - run: syft dir:./ -o spdx-json=sbom.spdx.json
  publish:
    needs: [sign, sbom]
    steps:
      - run: gh release create $TAG artefacts/* --generate-notes
  notify-channel:
    steps:
      - run: ./notify.sh stable "v${TAG}"
```

**Interview insight:** on-prem release engineering is its own world — very different from SaaS. Mention release channels, LTS, upgrade matrix testing, and SBOMs. If the interviewer works at a SaaS company, pivot to "but we use these techniques even for SaaS because they de-risk big changes."

### Q14. How do you enforce release policies at scale (e.g., "no deploys without two approvals, no deploys during change freeze, every release needs a ticket")?

**Answer:**

Release policy enforcement is where CI/CD meets governance. The policy must be:

1. Clearly specified
2. Automatically enforced (not relying on developers remembering)
3. Auditable after the fact
4. Emergency-overridable (glass-break) with audit trail

**Layered enforcement:**

**Layer 1: Repository-level (GitHub branch protection + environment rules).**

```yaml
# via gh api or Terraform
branch_protection:
  main:
    required_reviews: 2
    required_status_checks: [ci, security]
    enforce_admins: true
    require_signed_commits: true

environments:
  production:
    required_reviewers: [@sre-oncall, @release-manager]
    deployment_branch_policy:
      protected_branches: true
    wait_timer: 30m   # bake time after staging
```

**Layer 2: Pipeline-level (gates in the workflow).**

```yaml
jobs:
  check-policy:
    runs-on: ubuntu-latest
    steps:
      - name: Check for linked ticket
        run: |
          TITLE="${{ github.event.pull_request.title }}"
          if ! echo "$TITLE" | grep -qE '(JIRA-[0-9]+|INC-[0-9]+|OPS-[0-9]+)'; then
            echo "PR title must reference a ticket (JIRA-1234 etc.)"
            exit 1
          fi
      - name: Check change freeze
        run: |
          FREEZE=$(curl -s https://policy.internal/freeze-status | jq -r .active)
          if [ "$FREEZE" = "true" ]; then
            if ! echo '${{ github.event.pull_request.labels }}' | grep -q freeze-exempt; then
              echo "Change freeze active. PR needs 'freeze-exempt' label."
              exit 1
            fi
          fi
      - name: Check deploy window
        run: |
          HOUR=$(date -u +%H)
          DAY=$(date -u +%u)
          if [ "$DAY" -ge 5 ] || [ "$HOUR" -ge 16 ]; then
            echo "Outside allowed deploy window (Mon-Thu 00-16 UTC)."
            exit 1
          fi
```

**Layer 3: Policy-as-code (OPA / Kyverno for Kubernetes).**

```rego
# opa/policy.rego
package deploy

deny[msg] {
    input.kind == "Deployment"
    input.metadata.annotations["ticket"] == ""
    msg := "Deployment must have a 'ticket' annotation"
}

deny[msg] {
    input.kind == "Deployment"
    time.now_ns() > time.parse_rfc3339_ns("2026-11-24T00:00:00Z")
    time.now_ns() < time.parse_rfc3339_ns("2026-12-02T00:00:00Z")
    not input.metadata.labels["freeze-exempt"] == "true"
    msg := "Change freeze active (Thanksgiving)."
}
```

Evaluate via `conftest`:

```yaml
- uses: open-policy-agent/conftest@v0
  with:
    files: k8s/**/*.yaml
    policy: opa/
```

**Layer 4: Workflow approval flows (ServiceNow, Jira Service Management).**

For regulated environments, the change must have a ServiceNow change request (CR) approved before deploy. The pipeline checks:

```yaml
- name: Verify CR approved
  run: |
    CR_ID="${{ github.event.inputs.cr_id }}"
    STATUS=$(curl -u $SN_USER:$SN_PASS "https://snow.example.com/api/now/table/change_request/$CR_ID" | jq -r .result.state)
    if [ "$STATUS" != "approved" ]; then
      echo "CR $CR_ID not approved (state=$STATUS)"
      exit 1
    fi
```

**Glass-break (emergency override):**

Real outages need real overrides. The pattern:

1. A specific label (`emergency-override`) that only an on-call SRE can add.
2. Adding the label bypasses the policy check.
3. But the label itself triggers:
   - A Slack notification to leadership
   - A scheduled post-mortem on why the bypass was needed
   - An audit log entry

```yaml
- name: Check policy (with override)
  run: |
    if echo '${{ join(github.event.pull_request.labels.*.name, ',') }}' | grep -q emergency-override; then
      curl -X POST $SLACK_WEBHOOK -d '{"text":"EMERGENCY OVERRIDE by @${{ github.actor }} on PR #${{ github.event.pull_request.number }}"}'
      exit 0
    fi
    # ... normal policy checks ...
```

**Audit trail:**

Every policy decision — pass or fail — logs to an immutable store (CloudTrail, append-only S3, audit database). SOC2/ISO27001 auditors will ask for this.

**Interview insight:** at the senior+ level, this question tests whether you understand governance — not just tools. Mentioning OPA/Kyverno, glass-break procedures, and audit trails signals maturity. ServiceNow integration signals you've worked in regulated environments.

### Q15. How do you handle a release that contains a mix of features from multiple teams with different readiness levels?

**Answer:**

A realistic scenario: a monthly release train includes features from five teams. Team A's feature is rock-solid; Team B's is still being tested; Team C's was hacked together yesterday.

**Principle: let each feature advance at its own pace; don't hold the train.**

**Mechanism 1: Feature flags.**

Every new feature is behind a flag. The release ships the code; each team turns on their flag when ready. A risky feature stays off for weeks while others go live immediately.

```python
# Each team owns their flag lifecycle
if flags.is_enabled("team_a_new_dashboard", user):
    return new_dashboard_v2(user)
return new_dashboard_v1(user)
```

**Mechanism 2: Per-team canary cohorts.**

Each team has a cohort of beta users. A feature rolls out to its team's cohort first, then wider.

```python
cohorts = {
    "team_a_new_dashboard": ["beta-group-1", "internal-users"],
    "team_b_payment_v2":    ["payment-beta"],
}

def is_enabled(flag, user):
    if user in cohorts.get(flag, []):
        return True
    return global_rollout_percentage(flag, user)
```

**Mechanism 3: Feature branches reviewed before train boarding.**

The train cut-off is strict: only features with complete review, tests, docs, and a designated "release readiness" label can merge before the cut.

```yaml
# Train boarding check
- name: Check train readiness
  if: github.base_ref == 'release/v2.5'
  run: |
    LABELS='${{ join(github.event.pull_request.labels.*.name, ',') }}'
    REQUIRED=(ready-for-release qa-signed-off docs-updated)
    for r in "${REQUIRED[@]}"; do
      if [[ "$LABELS" != *"$r"* ]]; then
        echo "Missing label: $r"
        exit 1
      fi
    done
```

**Mechanism 4: Feature owner attestation.**

In the release PR or release notes, each feature owner signs off:

```markdown
# Release v2.5.0 readiness checklist

- [x] team-a: `@alice` — new dashboard (rolling out 10% → 100% over 2 weeks)
- [x] team-b: `@bob` — payment v2 (dark-launched; users still see v1)
- [ ] team-c: `@carol` — experimental search (HOLDING until review complete)
```

The release doesn't go out until all boxes are checked.

**Mechanism 5: Pre-release environments.**

Each feature team has a pre-release environment (`dev-team-a`, `dev-team-b`) where they can test in isolation. Promote to staging when ready.

**Decoupling deploy from release:**

The team test pattern mimics the SaaS "deploy != release" principle at scale:

- Train deploys on schedule (every 2 weeks, say)
- Each team controls when their feature is released to users
- The release manager only cares that the train is green, not that every feature is live

**Anti-patterns to avoid:**

- **"We'll cherry-pick Team C's changes out of the release."** Cherry-pick conflicts, tests re-run, you've effectively done two releases.
- **"We'll delay the release until Team C is ready."** One slow team blocks everyone.
- **"We'll disable Team C's code at runtime with a config flip."** That's essentially a feature flag — better to design it in from the start.

**Interview insight:** this question tests organisational awareness. The answer is "feature flags + clear contracts about when features can land vs. when they go live." Weak answers focus on branch management; strong answers focus on the product+engineering workflow.

### Q16. Design a release process that supports 99.999% availability (five nines).

**Answer:**

99.999% = 5.26 minutes of downtime per year. Every deployment must contribute virtually zero unavailability.

**Architectural prerequisites:**

Availability is a system property, not just a deployment property. Get these right first:

1. **Redundancy at every layer** — stateless services behind load balancers; databases with replicas across AZs.
2. **No single points of failure** — no singleton jobs that a rolling deploy would kill.
3. **Connection pooling with retry** — clients tolerate transient failures.
4. **Graceful shutdown** — SIGTERM → drain connections → then exit.
5. **Health checks separating liveness from readiness** — unhealthy pods leave the LB, healthy pods stay.

**Deployment strategies for 5 nines:**

**Progressive delivery, always:**

No big-bang deploys. Every change rolls out via canary with analysis gates.

**Multi-region with regional leader election:**

- Traffic served from N regions.
- Deploy to one region at a time.
- A region during deployment has drained traffic.
- Failover routes traffic to other regions.

**Connection draining:**

When a pod terminates, the LB removes it from the backend pool, but existing connections continue. Drain window: 30-60 seconds.

```yaml
# Kubernetes Deployment
terminationGracePeriodSeconds: 60
preStop:
  exec:
    command: ["sh", "-c", "sleep 20 && /app/drain"]
```

**Maximum unavailability = 0:**

```yaml
strategy:
  rollingUpdate:
    maxUnavailable: 0    # never drop below target capacity
    maxSurge: 25%        # can temporarily exceed
```

**Database changes:**

No schema migrations cause downtime. Online schema changes only (pt-osc, gh-ost, pg_repack). Write-path changes use expand-contract.

**Readiness probes that matter:**

The readiness probe must accurately reflect whether the pod can serve traffic:

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 3
  successThreshold: 2
  failureThreshold: 2
```

The `/ready` endpoint checks:
- Database pool initialised and reachable
- Cache connected
- Warmed up (e.g., pre-loaded critical data)

Only after all pass does the LB send real traffic.

**Automated rollback:**

Any SLI violation during rollout = instant automatic rollback.

```yaml
# Argo Rollouts
analysis:
  successCondition: result[0] >= 0.99999  # 5 nines success rate
  failureLimit: 1    # tolerate one failing check, then roll back
```

**Deploy windows:**

Deploys scheduled to minimise user impact:
- Avoid peak hours (region-local)
- Avoid known high-risk times (Black Friday)
- Multiple deploys per day instead of one big deploy (smaller blast)

**Post-deploy verification:**

Every deploy triggers:
- Synthetic tests from multiple regions
- Comparison to previous version's error/latency
- Automated ticket if anomalies found

**Chaos engineering:**

Continuous chaos tests (Gremlin, Litmus, Chaos Monkey) verify the system tolerates partial failures. A deploy that regresses chaos test results is blocked.

**Staff and process:**

- On-call always available
- Runbooks for every known failure mode
- Post-mortems for every incident, even "near misses"

**Game days:**

Quarterly exercises that deliberately cause failures: kill a region, fill a disk, inject network latency. Verify the deploy process and the product both survive.

**Interview insight:** 99.999% is an engineering culture, not just a pipeline. Mention chaos engineering, game days, regional failover, and online schema changes. Candidates who only talk about blue-green deployments haven't faced five-nines reality.

### Q17. How do you handle a security vulnerability that requires an emergency release?

**Answer:**

A CVE drops in a library your service depends on. You need to ship a fix *now*, bypassing normal processes without bypassing all safeguards.

**Playbook:**

**Hour 0: Triage.**

- Confirm the vulnerability affects your service (not every CVE is exploitable in your context).
- Assess severity: CVSS score, whether it's being actively exploited, whether your service is publicly exposed.
- Decide: emergency release or schedule for next regular release?

**Hour 0-1: Prepare the fix.**

- Identify the minimal change (usually a dependency bump).
- Open a PR on main. Label it `security` and `hotfix`.
- If the previous release also needs it, cherry-pick to release branches.

**Hour 1-2: Expedited review.**

- Security team and two senior engineers review immediately.
- CI runs full suite.
- If the fix is small and clearly correct, merge. Don't wait for the usual 24-hour review window.

**Hour 2-3: Deploy.**

- Still deploy progressively. "Emergency" doesn't mean "skip canary."
- Use the shortest-safe canary: 1% → 10% → 100% over 30 minutes instead of 24 hours.
- Monitor intensively; have a rollback plan.

**Hour 3+: Comms.**

- If customers or users are affected, send advisory.
- If it's a CVE you introduced in your product, publish GitHub Security Advisory.

**Pipeline support for emergency releases:**

```yaml
# .github/workflows/emergency-release.yml
on:
  workflow_dispatch:
    inputs:
      reason:
        required: true
        description: "CVE or incident reference"
      cr_id:
        description: "Change request (can be filed post-hoc for emergencies)"

jobs:
  emergency-deploy:
    runs-on: ubuntu-latest
    environment:
      name: production-emergency
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - run: make build test
      - name: Deploy canary at 10% (accelerated)
        run: ./deploy.sh --canary=10 --timeout=5m
      - name: Deploy canary at 50%
        run: ./deploy.sh --canary=50 --timeout=5m
      - name: Full rollout
        run: ./deploy.sh --canary=100
      - name: Log emergency deploy
        run: |
          curl -X POST https://audit.internal/deploys \
            -d @- <<EOF
          {
            "type": "emergency",
            "reason": "${{ github.event.inputs.reason }}",
            "cr_id": "${{ github.event.inputs.cr_id }}",
            "actor": "${{ github.actor }}",
            "commit": "${{ github.sha }}"
          }
          EOF
```

The `production-emergency` environment has different protection rules than `production` — fewer required approvers, shorter wait timer, different audit channel.

**Compensating controls for the shortcut:**

- All emergency deploys have a retrospective within 48 hours.
- The CR is filed post-hoc if needed; it's acceptable because the audit trail is complete.
- Paging leadership notifies them; they can veto if the risk isn't justified.

**Preventing emergency releases (better than handling them):**

- **Dependency bot (Dependabot, Renovate)** — bumps dependencies daily, catches CVEs before they reach prod.
- **Continuous vulnerability scanning** — Trivy, Snyk, Grype on every build.
- **Automated SBOMs** — instant impact analysis when a new CVE drops ("do any of our services use log4j 2.14?").
- **Regular patching cadence** — monthly scheduled patching of base images, dependencies.

**Post-mortem content:**

- Timeline of detection and response
- How long was the vulnerability in our code?
- Could we have caught it earlier?
- What process changes would help?

**Interview insight:** the log4j / Log4Shell incident (Dec 2021) is a canonical example. Senior candidates mention SBOMs (quickly answer "are we affected?"), shorter patch cadences, and emergency-deploy procedures with post-hoc audit.

### Q18. Design a release process that complies with ISO 27001 / SOC 2 controls.

**Answer:**

SOC 2 and ISO 27001 require demonstrable controls over changes to production. They don't mandate specific tools; they mandate evidence.

**Core controls auditors look for:**

1. **Segregation of duties.** The person who writes code cannot unilaterally deploy it.
2. **Formal change approval.** Every production change has an approval.
3. **Testing evidence.** Tests ran; someone reviewed the results.
4. **Audit log.** Who did what, when, with tamper-resistance.
5. **Rollback capability.** Demonstrated, not assumed.
6. **Access review.** Periodic review of who can deploy to production.

**Pipeline design meeting these controls:**

**Pull request approval (segregation + formal approval):**

```yaml
# Branch protection
required_pull_request_reviews:
  required_approving_review_count: 2
  require_code_owner_reviews: true
  dismiss_stale_reviews: true
```

Two reviewers, one must be a code owner. The PR author cannot approve.

**Environment protection (formal approval for production):**

```yaml
# Via GitHub Environment protection rules
production:
  required_reviewers: ["@sre-oncall", "@release-manager"]
  wait_timer: 30m
  deployment_branch_policy: main-only
```

Only specific reviewers can approve. 30-minute wait before deploy runs. Only `main` can deploy.

**Testing evidence:**

Every PR has attached test results:

```yaml
- name: Run tests
  run: pytest --junit-xml=junit.xml --cov=src --cov-report=xml
- uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: |
      junit.xml
      coverage.xml
    retention-days: 365
```

Tests archived for 1 year (SOC 2 evidence window).

**Audit log (tamper-resistant):**

Every deploy writes to an append-only store:

```yaml
- name: Audit deploy
  run: |
    curl -X POST https://audit.internal/api/events \
      -H "Authorization: Bearer $AUDIT_TOKEN" \
      -d @- <<EOF
    {
      "type": "deployment",
      "environment": "production",
      "commit": "${{ github.sha }}",
      "actor": "${{ github.actor }}",
      "approvers": "$(echo '${{ toJSON(github.event.review.approvers) }}')",
      "timestamp": "$(date -u -Iseconds)",
      "workflow_run": "${{ github.run_id }}"
    }
    EOF
```

Backend writes to WORM storage (S3 Object Lock, write-once archival).

**Rollback capability:**

- Drill quarterly: roll back a test deploy, document timing.
- Runbook for rollback published, reviewed yearly.
- Previous version artefact retained for 90 days minimum.

**Access review:**

Quarterly script lists:

- Who has merge rights to `main`
- Who has approve rights on `production` environment
- Who has access to secrets
- Who has admin on the CI/CD system

Management signs off that each is still appropriate.

```python
# scripts/access_review.py
import json, subprocess

admins = json.loads(subprocess.check_output([
    "gh", "api", "repos/org/repo/collaborators",
    "-X", "GET", "-f", "permission=admin"
]))
for a in admins:
    print(f"ADMIN: {a['login']} ({a['role_name']})")
# Export to CSV for quarterly review meeting
```

**Separation between CI and CD:**

CI runs tests; it cannot deploy. A separate CD system has deploy credentials. Compromising a CI job doesn't grant production access.

**Signed commits and signed artefacts:**

- Require signed commits on `main`.
- Container images signed with Cosign.
- Deploy verifies signature before applying.

**Secrets management:**

- Secrets in a vault (HashiCorp Vault, AWS Secrets Manager, 1Password).
- Not stored in the repo, ever.
- OIDC federation for cloud access; no long-lived keys.
- Secret access itself is audit-logged.

**Change categorisation:**

Some changes are lower risk:

- **Standard change** — pre-approved change type (routine patching, scheduled renewal). Doesn't need individual approval each time; the *type* was approved.
- **Normal change** — individual approval needed.
- **Emergency change** — expedited path with post-hoc review (Q17).

**Documentation evidence auditors want:**

- Change management policy document
- Pipeline architecture diagram
- Secret management procedure
- Incident response runbook
- Last 12 months of change records
- Last access review
- Last rollback drill result

**Interview insight:** for senior roles in regulated industries, this question is highly probable. Naming specific controls (segregation of duties, SoD, WORM logs, signed artefacts, Cosign, OIDC federation) and connecting them to the audit framework (SOC 2 CC8.1, ISO 27001 A.12.1) demonstrates that you've been in an audit room, not just read about one.
