# Quiz — Continuous Integration and Delivery

**Subject:** CI/CD
**Topics covered:** Build pipelines, source control integration, branch strategies, deployment strategies, release management, rollbacks, feature flags
**Format:** Multiple-choice and short-answer questions. Answers at the end.

---

### Q1. Which of the following is the primary goal of Continuous Integration?

a) Deploying code to production automatically
b) Integrating code changes frequently and verifying them with automated builds and tests
c) Replacing manual QA entirely
d) Running tests only before a release

---

### Q2. What is the key difference between trunk-based development and GitFlow?

---

### Q3. Which branch protection rule most directly prevents a single engineer from merging unreviewed code into `main`?

a) Required status checks
b) Restrict who can push
c) Required pull request reviews with code owner approval
d) Require signed commits

---

### Q4. Explain the difference between Continuous Delivery and Continuous Deployment.

---

### Q5. A blue-green deployment:

a) Runs two identical environments and switches traffic between them
b) Gradually shifts a percentage of traffic to a new version
c) Deploys one node at a time while the rest serve traffic
d) Runs new code only for a subset of users defined by a feature flag

---

### Q6. What is the advantage of a canary deployment over a blue-green deployment?

---

### Q7. A monorepo CI pipeline is slow because every change triggers a full build. Which technique is most effective?

a) Increase parallelism
b) Use affected-target detection (e.g., Bazel, Nx) to build only what changed
c) Upgrade build agents
d) Run the pipeline less frequently

---

### Q8. What is a "flaky test" and why is it dangerous in a CI pipeline?

---

### Q9. Which of the following best describes semantic versioning (SemVer)?

a) `MAJOR.MINOR.PATCH` — MAJOR for breaking changes, MINOR for new features, PATCH for bug fixes
b) `YEAR.MONTH.DAY` based on release date
c) Incrementing a single integer per release
d) Arbitrary version strings chosen by the release manager

---

### Q10. In a deployment pipeline, what is the purpose of a "release gate"?

---

### Q11. Which of the following is NOT a valid rollback strategy?

a) Redeploying the previous artifact
b) Using a feature flag to disable a new code path
c) Reverting the commit and pushing through the pipeline
d) Editing production files directly to remove the change

---

### Q12. What is the "fail fast" principle in CI pipelines?

---

### Q13. Feature flags (toggles) allow you to:

a) Deploy code without making it visible to users
b) Run the same artifact in different environments with different behaviour
c) Gradually roll out features to a subset of users
d) All of the above

---

### Q14. Why is it considered a bad practice to have a long-lived feature branch?

---

### Q15. Which metric is most useful for measuring deployment frequency?

a) Number of commits per week
b) Number of successful deployments to production per unit time
c) Number of pull requests merged
d) Number of CI pipeline runs

---

### Q16. What problem does a "dark launch" solve?

---

### Q17. Which of the following best describes the "fan-out" pattern in a build pipeline?

a) Running one stage on many build agents
b) Splitting a pipeline into parallel jobs that share state via artifacts
c) Running multiple pipelines concurrently from different triggers
d) Distributing builds across multiple regions

---

### Q18. What is the difference between a pipeline trigger and a pipeline schedule?

---

### Q19. Which of the following DORA metrics measures the time from commit to production?

a) Deployment frequency
b) Lead time for changes
c) Change failure rate
d) Mean time to recovery (MTTR)

---

### Q20. Describe the purpose of an "environment promotion" workflow.

---

### Q21. A rolling deployment:

a) Replaces all instances simultaneously
b) Replaces instances one at a time, maintaining capacity throughout
c) Runs only during off-hours
d) Requires manual approval between each instance

---

### Q22. What is the advantage of immutable artifacts (e.g., promoting the same container image through environments)?

---

### Q23. When would you prefer a scripted Jenkins pipeline over a declarative one?

---

### Q24. Which branching strategy has the lowest merge conflict rate in a team of 50+ engineers, assuming good test coverage?

a) GitFlow with long-lived feature branches
b) Trunk-based development with short-lived branches
c) Release branching per milestone
d) Environment-per-branch

---

### Q25. What is the purpose of a changelog and how should it be generated?

---

## Answers

**A1.** (b). CI's goal is frequent integration with automated verification — catching integration problems early when they're cheap to fix. Deployment is CD, not CI.

**A2.** Trunk-based development uses a single long-lived branch (`main`) with short-lived feature branches (hours to 1-2 days), relying on feature flags to hide incomplete work. GitFlow uses multiple long-lived branches (`develop`, `main`, `release/*`, `hotfix/*`) with feature branches merged via `develop`. Trunk-based suits CI/CD with frequent deploys; GitFlow suits versioned releases with discrete release trains.

**A3.** (c). Required PR reviews with code owner approval enforces that at least one other person (the owner of the changed code) reviews before merge. Status checks enforce CI passes but not human review.

**A4.** Continuous Delivery ensures every change is deployable at any time — the pipeline produces a releasable artifact, but production deployment is a manual decision. Continuous Deployment goes further: every change that passes the pipeline is deployed to production automatically.

**A5.** (a). Blue-green maintains two identical environments (blue = current, green = new) and cuts over traffic at the load balancer. (b) is canary, (c) is rolling, (d) is feature-flag-driven release.

**A6.** Canary exposes the new version to only a small percentage of traffic, allowing you to detect issues with limited blast radius before full rollout. Blue-green is binary — all-or-nothing at cutover — so problems affect 100% of users once switched.

**A7.** (b). Affected-target detection analyses the dependency graph and only rebuilds/tests code impacted by the change. This is the only approach that scales as the repo grows. More agents (a) and faster agents (c) help linearly; target detection helps logarithmically.

**A8.** A flaky test is a test that passes and fails non-deterministically on the same code. It's dangerous because (1) developers learn to ignore failures ("just re-run it"), masking real regressions; (2) it blocks pipelines, reducing deploy frequency; (3) it erodes trust in the entire test suite.

**A9.** (a). SemVer uses MAJOR.MINOR.PATCH with specific semantics: MAJOR for breaking API changes, MINOR for backwards-compatible features, PATCH for backwards-compatible bug fixes. A pre-1.0 version (0.x.y) signals the API is unstable.

**A10.** A release gate is a checkpoint that blocks progression until criteria are met — e.g., manual approval, integration tests passing, security scan clean, change-management ticket approved. Gates encode organisational policy in the pipeline.

**A11.** (d). Editing production files directly is never a valid rollback strategy — it's an outage in progress. All rollback paths must go through the pipeline to preserve the audit trail and ensure the change is reproducible.

**A12.** Fail-fast means running the cheapest checks first (lint, unit tests) and failing the pipeline at the first error rather than continuing. This gives developers feedback in minutes instead of hours and conserves build-agent capacity.

**A13.** (d). All three are valid uses of feature flags: decoupling deploy from release, environment-specific behaviour, and progressive rollout to user cohorts.

**A14.** Long-lived feature branches diverge from `main`, accumulate merge conflicts, and delay integration problems until the merge — the opposite of CI's purpose. They also hide the work from the rest of the team and prevent continuous testing against the current `main`.

**A15.** (b). Deployment frequency (a DORA metric) measures successful production deployments per unit time. Commits, PRs, and pipeline runs are proxies but don't measure actual delivery.

**A16.** Dark launch deploys code to production but doesn't expose it to users — e.g., a new service processes shadow traffic alongside the old one. This lets you validate performance, error rates, and correctness at production scale before cutover.

**A17.** (b). Fan-out splits a sequential pipeline into parallel jobs (e.g., build once, then run unit tests, integration tests, and security scans in parallel). The jobs share state via artifacts uploaded/downloaded between stages.

**A18.** A trigger runs the pipeline in response to an event (push, PR opened, tag pushed, webhook). A schedule runs the pipeline on a cron expression regardless of events. Schedules are used for nightly builds, dependency updates, and scheduled deploys.

**A19.** (b). Lead time for changes is the elapsed time from first commit to code running in production. Deployment frequency (a) is a separate metric; change failure rate and MTTR measure reliability.

**A20.** Environment promotion ensures the same immutable artifact is progressively deployed through dev → staging → prod, with increasing confidence at each stage. This guarantees what's tested is what ships, and centralises the release decision.

**A21.** (b). Rolling deployments replace instances one at a time (or in small batches), maintaining overall capacity. Kubernetes `Deployment` with `maxSurge`/`maxUnavailable` is the canonical example.

**A22.** Immutable artifacts guarantee byte-for-byte equivalence across environments, eliminating "works on my machine" and "works in staging" failure modes. The only variable is configuration, which is isolated from the artifact. Rollback becomes promotion of the previous immutable artifact.

**A23.** Scripted pipelines (Groovy) offer full programmability — dynamic stage generation, complex control flow, shared libraries with arbitrary logic. Use them for pipelines whose structure depends on runtime conditions or for building pipeline frameworks. Declarative is preferred for 90%+ of cases because it's easier to read, lint, and validate.

**A24.** (b). Trunk-based with short-lived branches minimises divergence and therefore minimises merge conflicts. Long-lived approaches (a, c, d) all accumulate conflicts proportional to branch age and parallelism.

**A25.** A changelog documents user-facing changes per release, grouped by type (Added, Changed, Deprecated, Removed, Fixed, Security). It should be generated from structured commit messages (Conventional Commits) using tools like `release-please`, `semantic-release`, or `git-cliff`. Manual changelogs are error-prone and drift from reality.
