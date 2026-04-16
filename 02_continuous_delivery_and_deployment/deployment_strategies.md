# Deployment Strategies — Interview Questions

**Subject:** CI/CD
**Topic:** Blue-Green, Canary, Rolling, Feature Flags, Dark Launches
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is a blue-green deployment, and what problem does it solve?

**Answer:**

**Blue-green deployment** maintains two identical production environments, only one of which serves traffic at any time.

- **Blue** is live and receiving all traffic.
- **Green** is idle, running the new version.

Deployment process:

1. Deploy the new version to **green** while **blue** keeps serving.
2. Run smoke tests against **green**.
3. Switch the router (DNS, load balancer, service mesh) to point at **green**.
4. **Blue** becomes idle and holds the previous version for rollback.
5. After a bake-in period, recycle **blue** for the next deployment.

**Problem solved:**

1. **Near-zero downtime.** Traffic cuts over atomically.
2. **Fast rollback.** Flip the switch back to blue — one network-config change, not a redeploy.
3. **Smoke testing on identical hardware.** Green is production-grade; issues that only surface in production (scale, config) can be caught before cutover.

**Kubernetes example (two Services, or label swap):**

```yaml
apiVersion: v1
kind: Service
metadata: { name: app }
spec:
  selector:
    app: app
    slot: blue   # switch to 'green' to cut over
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: app-blue }
spec:
  replicas: 10
  selector: { matchLabels: { app: app, slot: blue } }
  template:
    metadata: { labels: { app: app, slot: blue } }
    spec:
      containers:
        - name: app
          image: registry/app:v1.4.2
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: app-green }
spec:
  replicas: 10
  selector: { matchLabels: { app: app, slot: green } }
  template:
    metadata: { labels: { app: app, slot: green } }
    spec:
      containers:
        - name: app
          image: registry/app:v1.5.0
```

Cutover is a single command:

```bash
kubectl patch service app -p '{"spec":{"selector":{"slot":"green"}}}'
```

**Drawbacks:**

1. **Double the infrastructure cost** during deployment (blue and green both running).
2. **Stateful workloads are hard.** Databases can't be duplicated easily; schema migrations need to be compatible with both versions.
3. **Big-bang cutover.** All traffic switches at once. If the new version has a bug that only appears at production load, all users are hit simultaneously — contrast with canary.
4. **In-flight requests.** Need connection draining or session affinity.

**When to use:** stateless services where instant rollback matters more than gradual exposure.

### Q2. What is a canary deployment?

**Answer:**

**Canary deployment** exposes a new version to a small percentage of traffic first, gradually ramping up if metrics stay healthy.

Named after the "canary in a coal mine" — a small subset of users are the early warning for problems.

**Typical progression:**

```
0% → 1% → 5% → 25% → 50% → 100%
```

At each step, monitor SLIs (error rate, latency, business metrics). If any exceed thresholds, automatically roll back.

**Kubernetes example with Argo Rollouts:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: app }
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 1
        args:
          - name: service-name
            value: app
  selector: { matchLabels: { app: app } }
  template:
    metadata: { labels: { app: app } }
    spec:
      containers:
        - name: app
          image: registry/app:v1.5.0
```

**Automatic analysis:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: { name: success-rate }
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.995
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{job="{{args.service-name}}",status!~"5.."}[2m])) /
            sum(rate(http_requests_total{job="{{args.service-name}}"}[2m]))
```

If `success-rate` drops below 99.5%, Argo rolls back automatically.

**Advantages:**

1. **Blast radius limited** — bad deploy affects 1% of traffic, not 100%.
2. **Real production signal** — metrics from real users, real load.
3. **Gradual validation** — you can choose to hold at 5% for a day before proceeding.

**Drawbacks:**

1. **Slower.** Full rollout can take hours.
2. **Requires observability.** Without good metrics, you can't tell the canary is dying.
3. **Session consistency.** A user's requests might land on both versions unless you pin sessions.
4. **More complex tooling.** Requires a traffic splitter (service mesh, ingress controller) and analysis framework.

**When to use:** stateless services with good metrics infrastructure where partial exposure limits risk.

### Q3. What is a rolling deployment, and how does it differ from blue-green?

**Answer:**

A **rolling deployment** incrementally replaces instances of the old version with the new one, a few at a time.

**Kubernetes default:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: app }
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # never lose more than 1 pod
      maxSurge: 2         # allow 2 extra pods during rollout
  template:
    spec:
      containers:
        - name: app
          image: registry/app:v1.5.0
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds: 3
```

Kubernetes creates 2 new pods, waits for them to be Ready, then terminates 2 old pods, and repeats until all 10 are upgraded.

**Rolling vs. blue-green:**

| Aspect | Rolling | Blue-green |
|--------|---------|------------|
| Infrastructure cost | ~1.2× normal | 2× normal |
| Cutover | Gradual (pod by pod) | Atomic (switch) |
| Rollback | Requires redeploying old version | Switch back to blue |
| In-flight requests | Drain one pod at a time | Drain entire blue fleet |
| Both versions running simultaneously? | Yes, during rollout | Yes, during cutover |
| Cutover time | Minutes to hours | Seconds |

**Rolling handles both versions running simultaneously.** Your client code must tolerate this. If v1 and v2 expect different API responses, requests can fail mid-rollout.

**Rollback with rolling deployment:**

```bash
kubectl rollout undo deployment/app
# or
kubectl rollout undo deployment/app --to-revision=3
```

This is actually another rolling deployment, so rollback takes as long as the original rollout.

**When to use rolling:** default for most Kubernetes workloads. Works well for stateless services with backwards-compatible changes.

**When rolling fails:** incompatible schema changes, or when you need either-all-old-or-all-new semantics.

### Q4. What are feature flags, and how do they relate to deployment strategies?

**Answer:**

A **feature flag** (or feature toggle) is a runtime switch that controls whether a piece of code is active, without redeploying.

**Decouples deployment from release:**

- **Deploy** = code is running in production
- **Release** = users can access the feature

With feature flags, these are separate events. You can deploy hundreds of times per day but release features only when they're ready, and to specific audiences.

**Common types:**

| Type | Purpose | Example |
|------|---------|---------|
| **Release flag** | Hide incomplete feature | `if flag("new_checkout")` |
| **Experiment flag** | A/B test | Vary UI between two variants, measure |
| **Ops flag** | Kill switch | Disable expensive feature under load |
| **Permission flag** | Entitlement | Unlock pro features for paid users |

**Code example:**

```python
from featureflags import client

def render_homepage(user, request):
    if client.is_enabled("new_homepage", user=user, default=False):
        return new_homepage(user)
    return old_homepage(user)
```

**Service:**

LaunchDarkly, Unleash, Split, Statsig, or a home-rolled Redis-backed toggle service.

**Relation to deployment strategies:**

- **Canary + feature flag** — canary deploys the new code; feature flag controls who sees it. More flexible than canary alone because exposure can target user cohorts, not random % of traffic.
- **Dark launch** (see Q5) — deploy new code path but never take user action on its output.
- **Gradual rollout** — flip the flag from 0% to 100% over days, per-user (not per-request).

**Trade-offs:**

1. **Flag debt.** Old flags accumulate in code. Schedule regular cleanup.
2. **Combinatorial testing.** N flags = 2^N states. Test the live combinations.
3. **Runtime dependency.** Flag service downtime either fails open (serve old behaviour) or fails closed (outage).
4. **Audit.** Who toggled what, when, for whom?

**Interview insight:** feature flags are the bridge between continuous deployment and controlled release. Teams that deploy 100× a day without flags are either lying or reckless. Mention "deploy != release" to ground the answer.

### Q5. What is a dark launch?

**Answer:**

A **dark launch** deploys a new code path to production, runs it alongside the old code path on real traffic, but does not show the results to users or let it affect the user-visible behaviour.

Used to validate new code under production load before switching users to it.

**Example — replacing a search engine:**

```python
def search(query):
    old_results = old_search(query)

    # Dark launch: call new engine, log, but don't use its results
    try:
        new_results = new_search(query, timeout=100)
        log_comparison(old_results, new_results)
        metrics.observe("new_search_latency", new_search.latency)
        metrics.observe("result_diff_count", diff(old_results, new_results))
    except Exception as e:
        metrics.increment("new_search_error")

    return old_results   # users see the old behaviour
```

After running dark for weeks:

- Real-world performance numbers (p50, p99, timeout rate)
- Ranking differences between old and new (count, distribution)
- Edge cases discovered (unicode queries, very long inputs)

When the new engine matches or beats old on all metrics, flip the flag:

```python
def search(query):
    if flag("use_new_search", user):
        return new_search(query)
    return old_search(query)
```

**Variants:**

- **Shadow traffic** — duplicate incoming requests, send to both old and new systems, compare outputs offline.
- **Traffic mirroring** (Istio, Envoy) — the service mesh duplicates requests to a shadow deployment.

**Istio example:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
spec:
  hosts: [search]
  http:
    - route:
        - destination: { host: search-v1, subset: v1 }
          weight: 100
      mirror:
        host: search-v2
        subset: v2
      mirrorPercentage:
        value: 100.0
```

100% of traffic goes to v1 for real; the same 100% is mirrored to v2, whose responses are discarded.

**Typical use cases:**

- Rewriting a critical subsystem (search, pricing, recommendations)
- Moving from monolith to microservices
- Swapping databases (dual-writes + dual-reads)
- Major infrastructure change (new cache layer, new CDN)

**Interview insight:** dark launching is a Facebook coinage from ~2009. It's how you de-risk big changes. Mention it when discussing "how would you rewrite X critical system without downtime?"

### Q6. What is the difference between deployment and release?

**Answer:**

- **Deployment** = the code is running in production infrastructure.
- **Release** = users can actually reach and use the new behaviour.

Separating them decouples technical risk from product risk.

**Example timeline for a new payment method:**

| Date | Event | Who's affected |
|------|-------|----------------|
| Day 1 | Code deployed, feature flag off | Nobody (code inert) |
| Day 2 | Flag on for internal employees | 0.01% (~200 users) |
| Day 5 | Flag on for beta list | 0.1% (~2000 users) |
| Day 10 | Flag on for 1% of traffic | 1% |
| Day 15 | Flag on for 50% | 50% |
| Day 20 | Flag on for 100% | 100% — full release |
| Day 60 | Old payment code removed | Everyone on new |

The **deployment** happened on Day 1. The **release** happened on Day 20 (or Day 5, depending on which audience you count).

**Why separate them:**

1. **Deploy code independently of releasing.** Ship incomplete features behind flags, each merge a tiny increment.
2. **Target releases to audiences.** Internal users → beta users → percentage rollout → everyone.
3. **Kill switch.** A release can be reversed without redeploying.
4. **A/B tests.** Compare two variants of "released" without separate deploys.

**Enabling practices:**

- Feature flags (runtime toggles)
- Trunk-based development (small merges, often)
- Canary deployment (infrastructure-level gradual exposure)
- Robust observability (so you can measure release effects)

**Interview insight:** this is conceptual, not technical, but it tests whether a candidate understands modern release engineering culture. The rote answer ("deploy = put code on servers, release = make it live") is fine; the elaboration (flags, cohorts, decoupled risks) is what senior candidates bring.

---

## Intermediate

### Q7. How do you handle database schema changes in a zero-downtime deployment?

**Answer:**

Rolling, blue-green, and canary deployments all have both versions running simultaneously. The database schema must be compatible with both.

**The expand-contract pattern (a.k.a. parallel change):**

For every schema change, do it in two or more deployments:

1. **Expand** — schema change that is backwards-compatible. Both old and new code work.
2. **Migrate** — code switches over to the new schema.
3. **Contract** — remove the old schema elements.

**Example — renaming a column `email` → `email_address`:**

**Step 1 (expand):**
```sql
ALTER TABLE users ADD COLUMN email_address VARCHAR(255);
-- Backfill (via a background job, batched):
UPDATE users SET email_address = email WHERE email_address IS NULL;
-- Triggers to keep in sync during transition:
CREATE TRIGGER sync_email_cols BEFORE UPDATE ON users FOR EACH ROW
    EXECUTE FUNCTION sync_email();
```

Deploy this migration. The old code still reads/writes `email`; the trigger keeps `email_address` in sync.

**Step 2 (migrate):**
Code reads/writes `email_address` (possibly behind a feature flag for safety). Deploy and validate.

**Step 3 (contract):**
```sql
DROP TRIGGER sync_email_cols ON users;
ALTER TABLE users DROP COLUMN email;
```

Deploy this migration. Old code is long gone; the column is no longer needed.

**Destructive changes to avoid mid-rollout:**

- `DROP COLUMN` — old code may still SELECT it
- `RENAME COLUMN` — old code writes to the wrong column
- `ALTER COLUMN NOT NULL` — old code may insert NULLs
- `DROP TABLE` — instant breakage for any straggler code
- `ALTER COLUMN TYPE` (non-trivially) — may break queries that expect old type

**Tool support:**

- **Flyway / Liquibase** — versioned migrations, ordered by filename
- **Alembic (SQLAlchemy)** — Python migrations with auto-generation
- **golang-migrate** — simple, language-agnostic
- **Gh-ost / pt-online-schema-change** — for MySQL, perform long-running changes without locking tables

**Testing migrations:**

```yaml
- name: Test migration up
  run: |
    psql -c "CREATE DATABASE test_db"
    migrate -database "postgres://.../test_db" -path ./migrations up
- name: Test migration rollback
  run: migrate -database "postgres://.../test_db" -path ./migrations down 1
- name: Load production-shaped data
  run: pg_restore --dbname=test_db sanitised_prod_dump.sql
- name: Replay application tests
  run: pytest tests/integration --db-url="postgres://.../test_db"
```

**Online schema changes on large tables:**

`ALTER TABLE` on a 1-TB table in MySQL/Postgres locks the table for minutes. Use:

- **pg_repack** / **pt-online-schema-change** — copy-on-write into a new table, swap
- **Application-level dual-write + backfill** — write to both old and new schemas while migrating

**Interview insight:** this is the single hardest problem in zero-downtime deployment. Candidates who can walk through the expand-contract pattern, name specific tools, and discuss the read-write boundary show real experience.

### Q8. When would you choose canary over blue-green, and vice versa?

**Answer:**

Both limit blast radius, but they do it in different ways.

**Choose canary when:**

1. **You have good observability.** Canary needs automated analysis of SLIs. Without it, you're just eyeballing dashboards.
2. **Bugs are correlated with scale.** A production bug appears under real user load, not in staging. Canary tests with real production traffic.
3. **Gradual exposure matters more than fast rollback.** 1% → 100% over hours catches slow-burn issues.
4. **Stateless services with HTTP-like traffic.** Canary is easy to implement at L7.
5. **Costs matter.** Canary adds a few extra instances; blue-green duplicates the fleet.

**Choose blue-green when:**

1. **Rollback speed is paramount.** A flip-of-a-switch rollback is seconds vs. canary's minutes.
2. **You need atomic cutover.** A schema migration that requires all old code gone before new code runs can't be canary-safe.
3. **Workloads are stateful and both cannot run at once.** E.g., a singleton scheduler or leader-elected service.
4. **You have testing environments that should mirror prod exactly.** Blue-green's idle environment is the ultimate pre-prod.
5. **Regulatory or operational constraints on partial states.** Sometimes "two versions running" is forbidden.

**Hybrid: canary → blue-green cutover.**

Deploy new version to green. Use canary traffic splitting at the ingress to send 1%/5%/25% of traffic to green. When green reaches 100%, decommission blue. You get canary's gradual validation and blue-green's atomic rollback.

**Decision matrix:**

| Need | Strategy |
|------|----------|
| Fast rollback | Blue-green |
| Gradual user exposure | Canary |
| Atomic behavioural change | Blue-green |
| Low infra cost | Rolling or canary |
| Stateful workload | Blue-green (singleton) or rolling with care |
| Experimentation (A/B) | Canary or feature flags |

**Interview insight:** "it depends" is the right answer, but only if you can articulate the axes. Candidates who reflexively pick one show they've only used one in practice.

### Q9. How do you implement canary analysis with automated rollback?

**Answer:**

Canary analysis is the automated evaluation of whether the canary is healthy, and rollback if not. Without it, canary is manual and doesn't scale.

**Components:**

1. **Metrics pipeline** — Prometheus, Datadog, CloudWatch
2. **Analyser** — compares canary metrics to baseline (the stable version)
3. **Gating logic** — advances, holds, or rolls back the canary

**Common metrics:**

- **Error rate** — HTTP 5xx, exceptions, gRPC errors
- **Latency** — P50, P95, P99
- **Saturation** — CPU, memory, queue depth
- **Business metrics** — conversions, sign-ups, add-to-carts
- **Log anomalies** — rate of ERROR-level logs

**Statistical approach:**

Comparing raw numbers is naive — a canary with 1% of traffic has huge variance. Use **relative comparison** (canary vs. baseline) with a significance test:

```python
# Simplified Mann-Whitney or proportion z-test
def compare_error_rates(canary_errs, canary_total, base_errs, base_total, alpha=0.01):
    p_c = canary_errs / canary_total
    p_b = base_errs / base_total
    pooled = (canary_errs + base_errs) / (canary_total + base_total)
    se = sqrt(pooled * (1 - pooled) * (1/canary_total + 1/base_total))
    z = (p_c - p_b) / se
    p_value = 1 - norm.cdf(z)  # one-sided test
    return p_value < alpha      # True = canary significantly worse
```

**Tool examples:**

**Spinnaker Kayenta** — canary analysis as a service. Takes canary + baseline metrics, returns pass/fail with a score.

**Flagger (Kubernetes):**

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata: { name: app }
spec:
  targetRef: { apiVersion: apps/v1, kind: Deployment, name: app }
  service: { port: 80 }
  analysis:
    interval: 1m
    threshold: 5         # rollback after 5 failed checks
    maxWeight: 50
    stepWeight: 5
    metrics:
      - name: request-success-rate
        thresholdRange: { min: 99 }
        interval: 1m
      - name: request-duration
        thresholdRange: { max: 500 }
        interval: 30s
    webhooks:
      - name: load-test
        url: http://flagger-loadtester/
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 http://app-canary/"
```

Flagger advances `stepWeight%` every `interval`, evaluates metrics, rolls back if `threshold` consecutive checks fail.

**Argo Rollouts AnalysisTemplate** — see Q2 example.

**Failure modes:**

1. **Flaky metric causes false rollback.** Cooldown and multiple consecutive failures help.
2. **Canary too small to be statistically meaningful.** Can't detect a 0.1% error rate increase with 100 requests. Hold at higher percentages longer.
3. **Cold cache skew.** The canary starts with empty cache; latency looks worse. Warm it before canary analysis starts.
4. **Selection bias.** If canary gets only certain requests (e.g., sticky-session users), metrics differ for non-canary reasons.

**Interview insight:** if the interviewer asks "how do you tell if the canary is bad?", weak candidates say "check the dashboards." Strong candidates name a specific tool (Kayenta, Flagger, Argo), discuss statistical significance, and mention the baseline-comparison technique.

### Q10. How do you manage feature flags at scale without them becoming technical debt?

**Answer:**

Flag debt is real. Etsy at one point had 2000+ active flags, many of whom owners had forgotten about. Etsy wrote a seminal post-mortem about this.

**Governance to prevent flag debt:**

1. **Classify flags at creation.** Each flag has a `type` (release, ops, experiment, permission) and a `planned_removal_date`.
2. **Ownership.** Every flag has an owner. When they leave, flags go into a "needs-adoption" queue.
3. **Automated expiration.** A CI job scans the codebase for flags; compares to the flag service; flags that are 100%-on for > N days should be removed.
4. **Pull requests to clean up.** The flag service auto-files a PR when a flag hits its removal date.
5. **Measure flag lifetime.** Dashboard: median flag lifetime, flags over their removal date, flags with zero traffic.

**Example: automated stale-flag detection:**

```python
# ci/audit_flags.py
import subprocess, json, sys
from datetime import datetime, timedelta

flags = json.loads(subprocess.check_output(["curl", "-s", "https://flags.internal/api/flags"]))
active_in_code = subprocess.check_output(
    ["git", "grep", "-h", "flag(", "--", "*.py", "*.js"]
).decode()

warnings = []
for flag in flags:
    if flag["key"] not in active_in_code:
        warnings.append(f"Flag '{flag['key']}' exists in service but not in code — delete it")
    age = datetime.now() - datetime.fromisoformat(flag["created_at"])
    if flag["rollout"] == 100 and age > timedelta(days=60):
        warnings.append(f"Flag '{flag['key']}' is 100%-on for {age.days} days — remove from code")
    if flag["rollout"] == 0 and age > timedelta(days=30):
        warnings.append(f"Flag '{flag['key']}' is 0%-on for {age.days} days — dead flag?")

if warnings:
    print("\n".join(warnings))
    sys.exit(1)
```

**Flag hygiene rules:**

1. **One flag per PR.** Don't stack five flags in one change.
2. **Use short names and namespace.** `billing_v2_checkout`, not `feature1`.
3. **Default-off.** New flags start off; opt-in for early access.
4. **Tests cover both states.** Tests marked with the flag state; flip the default, watch tests fail.
5. **Document the plan.** "Rollout from 0 to 100% over 2 weeks, remove flag by Q2."

**Runtime dependency concerns:**

- **Flag service as SPOF.** If your flag provider is down, what happens? Usually fail to last-known-good; prefer this over fail-closed.
- **Client-side caching.** SDKs should cache flags locally, refresh every few seconds, and serve stale if the backend is unreachable.
- **High-volume flag evaluations.** A flag checked 10k times per request kills performance. Cache the decision at request scope.

**Example flag cleanup PR:**

```python
# Before
if flag.is_enabled("new_checkout_ui", user=user):
    return render_new_checkout(user)
return render_old_checkout(user)

# After (flag cleanup)
return render_new_checkout(user)
```

**Interview insight:** flag debt is one of those problems that reveals engineering maturity. Candidates who can talk about automated cleanup, owners, and expiration dates show they've managed a real feature flag system, not just used one.

### Q11. Design a deployment strategy for a stateful service like PostgreSQL or Redis.

**Answer:**

Stateful services don't support blue-green or canary in the simple way stateless services do, because you can't duplicate the data.

**Patterns:**

**1. Primary-replica failover (simplest).**

Run an always-on replica. To deploy:

1. Upgrade the replica.
2. Promote the replica to primary (failover).
3. The old primary, now a replica, gets upgraded.
4. Failover back if desired, or leave it.

Downtime: the failover window (seconds to minutes).

**2. Rolling upgrade of clusters.**

For natively clustered stores (Kafka, Cassandra, Elasticsearch, Redis Cluster):

```bash
# Kubernetes StatefulSet rolling update
kubectl set image statefulset/redis-cluster redis=redis:7.2.5
# Rolls pod by pod, waiting for readiness
```

Each pod is drained, upgraded, re-joined. The cluster tolerates one node at a time being down.

**Constraints:**
- Cluster must support mixed-version operation during rollout (e.g., Kafka's minor versions).
- `podManagementPolicy: OrderedReady` for ordered updates; `Parallel` for speed.
- `maxUnavailable: 1` in the StatefulSet's `updateStrategy`.

**3. Blue-green with data migration.**

Spin up a new cluster (green), replicate data from blue to green, cut traffic over. This is how major version upgrades are done (Postgres 14 → 16).

Tools:
- **pg_basebackup + logical replication** for Postgres
- **AWS DMS** for migrations between engines/versions
- **Debezium + Kafka** for CDC-based migrations

**4. In-place upgrade with downtime window.**

The blunt approach: schedule a maintenance window, take the service down, upgrade binaries, restart. Appropriate for:
- Infrequent, well-announced upgrades
- Single-instance services where replicas are unjustified
- Low-stakes internal systems

**Key constraints per service:**

- **Schema/format compatibility.** New binary must read old files. Postgres guarantees this within major versions; major upgrades need `pg_upgrade` or dump/restore.
- **Protocol compatibility.** Clients of the old version must still work during the rollout.
- **Leader election during upgrade.** Etcd, Consul, Raft-based systems need to elect a new leader when the current one goes down. Expect brief write unavailability.

**Kubernetes PostgreSQL operator example:**

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: { name: mydb }
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:16.4
  primaryUpdateStrategy: unsupervised   # or supervised for manual gates
  primaryUpdateMethod: switchover        # or restart
```

`switchover` promotes a replica to primary first, then upgrades the old primary. `restart` just restarts in place (has a downtime window).

**Interview insight:** don't treat databases as containerised services. The upgrade path for Postgres is different from the upgrade path for Redis is different from Kafka. Mentioning one concrete operator or upgrade tool for a specific system shows depth.

### Q12. How do you test that a rollback will actually work?

**Answer:**

Deployment rollbacks are a muscle you exercise, not a capability you assume. Many teams have been burnt by rollbacks that failed when finally needed.

**Why rollbacks fail:**

1. **Forward-only schema changes.** The new code wrote new columns; the old code doesn't know about them and misbehaves (or the migration added NOT NULL to a new column).
2. **Missing artefacts.** The old container image was garbage-collected from the registry.
3. **Irreversible side effects.** New code sent messages to a queue that the old code can't consume.
4. **Cache poisoning.** New code wrote incompatible entries to a shared cache.
5. **Config drift.** The feature flags or config that went out with the new deploy don't match what the old code expects.

**Test strategies:**

**1. Rollback drills.**

Scheduled practice: intentionally deploy a new version, then roll back, in a staging environment (or production during low-traffic windows). Check:

- Can you get back to the previous version with one command?
- Do tests pass after the rollback?
- How long did the rollback take?

**2. Canary-style rollbacks.**

During canary rollouts, automated rollback triggers when SLIs breach. This is testing rollback continuously — every failed canary is a rollback drill.

**3. N-1 compatibility tests.**

In CI, run the full test suite not just against the current version but against a scenario where half the fleet is N and half is N-1. This catches protocol / schema incompatibilities.

```yaml
jobs:
  forward-compat:
    steps:
      - run: docker run registry/app:$CURRENT_VERSION & docker run registry/app:$PREV_VERSION
      - run: ./e2e/full-suite
```

**4. Schema rollback tests.**

For every migration:

```yaml
- run: migrate up
- run: run_integration_tests --version=new
- run: migrate down
- run: run_integration_tests --version=old   # does old app work after rollback?
```

Most migration failures are one-way. Either make every migration reversible (expand-contract), or acknowledge that some migrations are irreversible and plan differently (forward-fix, not rollback).

**5. Artefact retention policies.**

Pin policies so container images, JARs, debians aren't garbage-collected for 90 days minimum. Test retention by trying to pull last month's version.

```bash
# Docker / OCI registry retention
# e.g., JFrog Artifactory / GHCR retention rules
# Keep all tags matching v*.*.*, keep last 100 untagged digests
```

**6. Backout runbooks.**

Every deployable service has a `ROLLBACK.md` containing:

- Exact command(s) to roll back
- Any data/cache considerations
- Any out-of-band notifications (status page, Slack)
- Known-safe previous version

**Antipatterns:**

- "We've never needed to roll back" — usually means "we've rolled forward through outages because we couldn't roll back"
- "Rollback is the same as deploy" — maybe, but have you tested the reverse direction?

**Interview insight:** this question separates "I've deployed" from "I've operated." The answer should involve practice, specific failure modes (schema, artefacts, side effects), and automation.

---

## Advanced

### Q13. How do you deploy safely when two services depend on each other and both need to change?

**Answer:**

Service A calls service B; both need a coordinated change. Deploying them together is risky; deploying them separately requires care.

**Naming the problem: a protocol change between A and B.**

**Anti-pattern — deploy simultaneously:**

```
T0: Deploy A and B at the same time
T1: During the rollout window (maybe 5 min), some A@old instances talk to B@new, and vice versa.
T2: Protocol mismatch → errors.
```

**Correct approach — expand-contract on the protocol, mirror Q7's DB advice:**

**Phase 1: Expand B to support both old and new.**

B speaks old protocol (what A currently uses) and new protocol. Deploy B.

**Phase 2: Switch A to new protocol.**

A now calls B using the new protocol. Deploy A. B still supports both, so rollout windows with mixed A versions are safe.

**Phase 3: Contract B to drop old protocol.**

Verify no A is using the old protocol (observability). Remove old-protocol code from B. Deploy B.

**Example — changing a gRPC endpoint signature:**

```proto
// v1 (before)
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
}
message GetUserRequest { string id = 1; }

// v2 expand: add new field, accept both
message GetUserRequest {
  string id = 1;
  bool include_profile = 2;  // new optional field
}

// v2 contract: old field removed once all clients upgraded
```

**Protocol versioning conventions:**

1. **Add fields, never remove.** Clients ignore unknown fields.
2. **Never renumber or reassign field IDs.** In protobuf, this breaks wire compatibility.
3. **New required behaviour = new method.** `GetUserV2` with breaking changes, deprecate `GetUser` once everyone migrates.
4. **Version in URL or header, not implicit.** `/v2/users`, not "we'll just change what v1 means."

**Cross-service feature flags:**

If both services agree on a flag name, they can coordinate:

```python
# In Service A
def get_user(uid):
    if flag("user_service_v2_protocol", uid):
        return service_b.get_user_v2(uid)
    return service_b.get_user_v1(uid)

# Service B offers both for the transition
def handle_get_user_v1(req): ...
def handle_get_user_v2(req): ...
```

When the flag is 100% on for a week, remove v1.

**When atomic coordination is really needed:**

Rare, but sometimes (schema migration requiring both writer and reader to change at once). Tools:

- **Coordinated deployment** — orchestrator pauses traffic, deploys both, resumes traffic. This is effectively downtime.
- **Read-only mode** — A is in read-only mode while the switch happens. Depends on your business.
- **Monorepo** — if A and B are in the same repo, an atomic merge at least ensures the intent is atomic, even if deployment isn't.

**Interview insight:** the expand-contract pattern applies to protocols, not just databases. Strong candidates generalise.

### Q14. Design a progressive delivery pipeline for a service with strict SLOs.

**Answer:**

**Progressive delivery** = deployment + release + experimentation, automated end-to-end with SLO enforcement.

**Example spec:**

- Service has 99.95% availability SLO, P99 latency < 300 ms
- 30-day error budget = 21.6 minutes of downtime
- Deploy multiple times per day

**Pipeline design:**

```
1. Merge to main
   ↓
2. CI: build, test, scan, sign, publish image
   ↓
3. CD: deploy to staging (full stack)
   ↓
4. Run smoke tests + selected E2E tests against staging
   ↓
5. Deploy to prod as canary (1%)
   ↓
6. Flagger / Argo Rollouts analyses for 10 min
   - Error rate < 0.05%?
   - P99 < 300 ms?
   - No elevated rate of ERROR logs?
   ↓
7. If healthy: 5% → analyse → 25% → analyse → 100%
   If unhealthy: auto-rollback + page on-call
   ↓
8. Feature flags control end-user exposure independently
   ↓
9. After bake-in, remove old version
```

**Flagger config for this scenario:**

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata: { name: orders }
spec:
  targetRef: { kind: Deployment, name: orders }
  progressDeadlineSeconds: 1800
  service: { port: 80 }
  analysis:
    interval: 1m
    iterations: 10
    threshold: 3          # tolerate 3 consecutive failures
    maxWeight: 100
    stepWeight: 5
    metrics:
      - name: success-rate
        thresholdRange: { min: 99.95 }
        interval: 1m
      - name: latency-p99
        thresholdRange: { max: 300 }
        interval: 1m
      - name: error-log-rate
        templateRef: { name: error-log-rate }
        thresholdRange: { max: 10 }
    webhooks:
      - name: smoke-test
        type: pre-rollout
        url: http://smoke-runner/run
        timeout: 60s
      - name: acceptance-test
        type: rollout
        url: http://acceptance-runner/run
        timeout: 300s
```

**SLO enforcement (error budget as gate):**

```python
# ci/check_error_budget.py
# Gate production deploys when error budget is < 10% remaining.
import requests, sys

r = requests.get("https://sli.internal/api/orders/slo_state")
budget_remaining = r.json()["budget_remaining_pct"]

if budget_remaining < 10:
    print(f"ERROR BUDGET LOW ({budget_remaining}%) — blocking deploy")
    sys.exit(1)
```

Add this as a step in the deploy workflow. When the service is burning budget faster than expected, deploys pause automatically, forcing engineers to stabilise before shipping new features.

**Observability requirements:**

- Per-version metrics (tag by `image_tag` / `git_sha`)
- Baseline comparison (last-stable-version)
- Fast metric pipeline (seconds, not minutes)
- Alerting on rollback events

**Chaos engineering integration:**

Between step 4 and 5, run chaos experiments against staging (Gremlin, Litmus):

```yaml
- name: Chaos latency injection
  run: litmusctl run pod-network-latency --namespace=staging --duration=5m
- name: Verify SLO holds during chaos
  run: validate-slo --duration=5m
```

If the service can't hold its SLO under chaos, don't promote to prod.

**Interview insight:** progressive delivery is the 2024+ vocabulary. Argo Rollouts, Flagger, and Spinnaker are the common tools. Mention SLO-gated deploys and error budgets to show SRE fluency.

### Q15. How do you handle deployments that involve multiple regions (multi-region CD)?

**Answer:**

Multi-region deployment adds two complications: (a) sequential vs. parallel rollout across regions, and (b) regional failover during deployment.

**Strategy 1: Wave-based rollout.**

Deploy to regions in sequence, with bake time between:

```
Wave 1: eu-west-1  (20 min bake)
Wave 2: us-east-1  (20 min bake)
Wave 3: ap-southeast-2 (20 min bake)
Wave 4: everywhere else
```

If any wave fails, stop. Regions that haven't deployed are still on the old version — no further risk.

**Pipeline example:**

```yaml
jobs:
  wave-1:
    runs-on: ubuntu-latest
    environment: prod-eu-west-1
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh --region=eu-west-1
      - run: ./bake.sh --duration=20m --region=eu-west-1

  wave-2:
    needs: wave-1
    runs-on: ubuntu-latest
    environment: prod-us-east-1
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh --region=us-east-1
      - run: ./bake.sh --duration=20m --region=us-east-1

  wave-3:
    needs: wave-2
    runs-on: ubuntu-latest
    environment: prod-ap-southeast-2
    steps:
      - run: ./deploy.sh --region=ap-southeast-2
```

**Choosing the first wave:**

- Smallest traffic region (limits blast)
- Internal-only region if one exists
- Region where the on-call team is awake (timezone-aware)

**Strategy 2: Canary region.**

Dedicated "canary region" that takes 5-10% of global traffic. Deploy there first, bake, then deploy everywhere.

**Strategy 3: Global canary.**

Rarely used — deploy 1% of instances in every region simultaneously. Argues that "bugs are independent of region" so partial global exposure is safer than 100% regional.

**Failover considerations:**

What if the deployment pipeline itself goes through the region being deployed? Don't deploy to the region hosting your CD system *from that region*. Use an out-of-band control plane.

**Traffic management:**

During region-at-a-time rollout, a GeoDNS or Global Load Balancer directs traffic. If a region fails, traffic reroutes:

```yaml
# AWS Route 53 health-checked failover
- Primary: eu-west-1 ALB
- Secondary: us-east-1 ALB
- Health check: /healthz every 10s, fail after 3 misses
```

**Data residency:**

Some data must stay in-region (GDPR, HIPAA). Deployments must respect this:

- Per-region database clusters
- Cross-region replication only for allowed data types
- Schema migrations run per-region

**Config per region:**

```yaml
# helm values
global:
  image: registry/app:v1.5.0
regions:
  eu-west-1:
    dbHost: pg-eu-west-1.internal
    regionCode: EU
  us-east-1:
    dbHost: pg-us-east-1.internal
    regionCode: US
```

**Pitfalls:**

- **Inconsistent versions across regions for days.** Complicates debugging — which version is a user hitting?
- **Clock skew assumptions.** Code deployed to region A calls region B, but they're on different versions. See Q13.
- **Global state.** Feature flags, config, and secrets are often global. Changing them affects all regions simultaneously. Gate changes carefully.

**Interview insight:** multi-region CD is staff-level territory. Mention wave-based rollouts, error-budget gating, and data residency to show fluency. Concrete tools: Spinnaker (regional deployments native), Argo CD with ApplicationSet generators.

### Q16. What is GitOps, and how does it change the deployment model?

**Answer:**

**GitOps** is a deployment model where Git is the single source of truth for the desired state of a system, and an in-cluster agent reconciles actual state to match.

**Coined by Weaveworks, 2017.**

**Principles:**

1. **Declarative state in Git.** Kubernetes manifests, Helm charts, Terraform.
2. **Version controlled.** Every change is a commit, PR, and approval.
3. **Automated reconciliation.** An agent pulls from Git and applies the state.
4. **Continuous drift detection.** The agent continuously compares actual vs. desired and fixes drift.

**Tools:**

- **Argo CD** — Kubernetes-native, strong UI, huge adoption
- **Flux** — Kubernetes-native, lighter, CNCF graduated
- **Atlantis** — Terraform PRs with plan/apply via Git comments
- **Jenkins X** — GitOps-oriented Jenkins distribution

**Workflow:**

```
Developer → git push feature branch → PR → review → merge to main
                                                        ↓
                                              Argo CD (polling or webhook)
                                                        ↓
                                              Apply manifests to cluster
                                                        ↓
                                              Cluster state matches Git
```

**Argo CD Application example:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: orders, namespace: argocd }
spec:
  project: default
  source:
    repoURL: https://github.com/org/config
    targetRevision: main
    path: apps/orders/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: orders
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff: { duration: 5s, factor: 2, maxDuration: 2m }
```

**Push vs. pull model:**

- **Push (traditional CI/CD):** Pipeline has credentials for the target environment; `kubectl apply` happens from the pipeline.
- **Pull (GitOps):** The agent in the cluster has credentials only for itself; it pulls desired state from Git.

Pull is more secure — your CI system never has prod cluster credentials.

**Repository structure — two common patterns:**

**App-config in same repo:**
```
/my-service/
  src/
  .github/workflows/ci.yml
  deploy/
    base/
      deployment.yaml
    overlays/
      staging/kustomization.yaml
      prod/kustomization.yaml
```

**Separate config repo (more common):**
```
# my-service (source)
/src
/.github/workflows/ci.yml

# my-service-config (separate repo)
/overlays/staging/
/overlays/prod/
```

CI writes the new image tag to the config repo:

```yaml
- name: Bump image in config repo
  run: |
    gh repo clone org/config
    cd config
    yq -i '.image.tag = "${{ github.sha }}"' apps/orders/overlays/staging/values.yaml
    git commit -am "bump orders to ${{ github.sha }}"
    git push
# Argo CD sees the commit and deploys
```

**Trade-offs:**

| Aspect | Traditional CD | GitOps |
|--------|---------------|--------|
| Audit trail | Depends on pipeline logs | Git log is the trail |
| Drift detection | Manual | Automatic |
| Credential surface | CI has prod credentials | Agent has prod credentials |
| Rollback | Redeploy via pipeline | `git revert` |
| Debuggability | Pipeline logs | Agent logs + Git |
| Multi-environment | Explicit pipeline steps | Separate Git paths / branches |

**Pitfalls:**

1. **Config repo sprawl.** One repo per service times N environments becomes unmanageable. Use ApplicationSet (Argo) or Kustomize generators.
2. **Slow reconciliation.** Default 3-minute polling is too slow for emergency changes. Use webhooks.
3. **Secrets.** Secrets in Git need encryption (SOPS, Sealed Secrets — see `source_control_integration.md` Q10).
4. **Progressive delivery gaps.** Argo CD alone doesn't do canary; pair with Argo Rollouts or Flagger.

**Interview insight:** GitOps is now standard in Kubernetes shops. Name Argo CD or Flux. Differentiate pull model from push. Mention Sealed Secrets or SOPS for the secrets question interviewers often ask next.

### Q17. How do you design deployment for a system with strong consistency requirements (e.g., financial transactions)?

**Answer:**

In payments, trading, or regulated systems, partial failures during deployment can cause financial loss or compliance violations. The deployment must preserve invariants.

**Key invariants to preserve:**

1. **No dropped transactions.** Every transaction acknowledged to the user is durable.
2. **No duplicates.** A single transaction isn't double-processed.
3. **No ordering violations.** Related transactions apply in the correct order.
4. **Auditability.** Every decision (approved, rejected) is traceable.

**Strategies:**

**1. Read-only mode during cutover.**

For truly atomic changes, briefly put the system in read-only mode:

```
T0: Announce planned maintenance (regulatory + customer comms)
T1: Set system to read-only mode
T2: Drain in-flight writes, confirm all acked writes are durable
T3: Deploy new version
T4: Verify data integrity (checksums, canary reads)
T5: Resume writes
```

Downtime: typically minutes. Required for some schema changes or algorithm changes.

**2. Idempotency + dual-writes.**

Every transaction has an idempotency key. During a rollout:

- Old and new versions both accept writes
- Both compute the same result for the same idempotency key
- Writes are durable before acknowledgement
- Duplicate writes (same key) are deduplicated at the storage layer

This is how Stripe, Square, and similar payment systems handle deploy windows.

```python
def charge(idempotency_key: str, amount: int, user: str) -> Transaction:
    existing = transactions.find_by_key(idempotency_key)
    if existing:
        return existing   # safe to retry
    with transaction():
        result = process_payment(user, amount)
        transactions.insert(idempotency_key, result)
        return result
```

**3. Event sourcing + replay.**

The system records events to a durable log (Kafka, DynamoDB streams). The current state is a projection. A new version can:

- Replay events from the log, building its state
- Take over once its projection matches the old version's state
- Old version is then safe to decommission

**4. Shadow mode (dark launch) for algorithmic changes.**

For risk models, fraud detection, credit scoring: the new algorithm runs in parallel, its decisions logged but not used. After weeks of comparison, switch.

```python
def approve_transaction(tx):
    old_decision = old_model.predict(tx)
    new_decision = new_model.predict(tx)
    log_comparison(tx.id, old_decision, new_decision)
    metrics.increment("old_approved" if old_decision else "old_denied")
    metrics.increment("new_approved" if new_decision else "new_denied")
    return old_decision   # authoritative
```

**5. Strict compatibility windows.**

Every release is tagged as compatible with specific data format versions. The deployer enforces that no two incompatible versions run simultaneously.

**6. Regulatory considerations:**

- **Change windows.** Some regulators mandate deploys only during announced windows.
- **Four-eyes approval.** Two engineers approve every production change.
- **Dual-region writes.** Every transaction committed to two regions before acknowledgement.
- **Non-repudiation.** Every decision signed with a versioned key.

**Deployment pipeline example:**

```yaml
jobs:
  pre-deploy-checks:
    steps:
      - name: Verify reconciliation completed
        run: ./scripts/check_reconciliation.sh
      - name: Verify ledger integrity
        run: ./scripts/ledger_checksum.sh
      - name: Pause scheduled jobs
        run: ./scripts/pause_batches.sh

  deploy:
    needs: pre-deploy-checks
    environment:
      name: prod
      url: https://pay.example.com
    steps:
      - name: Require two approvals
        if: github.ref == 'refs/heads/main'
        # Handled via environment protection rules
      - name: Deploy
        run: ./deploy.sh --mode=canary

  post-deploy-verification:
    needs: deploy
    steps:
      - name: Run reconciliation
        run: ./scripts/reconcile_transactions.sh --since-deploy=$DEPLOY_TIME
      - name: Verify no dropped transactions
        run: ./scripts/verify_txn_count.sh --expected-delta=$EXPECTED
```

**Interview insight:** for financial or regulated systems, the answer is never "just do blue-green." Mention idempotency keys, dual-writes, reconciliation, and explicit change windows. Candidates who have worked on payments, trading, or clinical systems bring this nuance.

### Q18. Compare continuous deployment to release train models.

**Answer:**

Two philosophies of how often changes reach production:

**Continuous Deployment:**

- Every merge to main is deployed to production automatically
- Deployments per day: 10-1000+
- Changes are small and independent
- Feature flags control user exposure

**Release Trains:**

- Changes accumulate on a release branch during a fixed window (e.g., 2 weeks)
- At end of window, a release is cut, stabilised, and deployed
- Deployments per year: 26 (fortnightly) to 4 (quarterly)
- Changes are bundled; release notes enumerate features
- Strong QA gate before release

**When release trains work:**

- **Shrink-wrapped software.** Ship a version to customers who install it (databases, IDEs, mobile apps pre-continuous-delivery).
- **Regulated industries.** Medical devices, automotive — every release is a regulatory event.
- **Coordinating across teams.** The train sets a predictable rhythm; teams know they must "catch the train" for their feature to go out.
- **Release notes matter.** Customers (B2B SaaS) expect grouped, documented releases.

**When continuous deployment works:**

- **Web SaaS.** Users don't need to install anything; there's no "release" to them.
- **High-throughput teams.** Hundreds of engineers can't wait on a central release manager.
- **Infrastructure investment exists.** Feature flags, canary deployments, rollback automation are prerequisites.

**Hybrid models:**

**1. Continuous deployment with feature flags for releases.**

Deploy continuously; release via flag flips. The "release" becomes a marketing-and-product event distinct from engineering.

**2. Train per team, not per organisation.**

Each team deploys continuously within their service, but major product releases bundle and announce.

**3. Trunk continuous + LTS trains.**

`main` deploys continuously to SaaS. Periodically, an LTS branch is cut for on-prem customers who want slower updates.

**Metrics comparison (DORA):**

| Metric | CD median (high performers) | Release trains (low performers) |
|--------|-------|-----------------|
| Deployment frequency | Multiple per day | 1 per month |
| Lead time for changes | < 1 day | 1-6 months |
| Mean time to recover | < 1 hour | > 1 day |
| Change failure rate | 0-15% | 46-60% |

Release trains don't inherently have worse metrics, but organisations that *can't* deploy more often than monthly tend to correlate with the low-performer bucket.

**Common misconception:**

"We use release trains because our customers can't handle constant change." Often the real constraint is pipeline confidence, not customer preference. Feature flags address the customer concern separately.

**Interview insight:** the right model depends on the domain. Mobile apps and embedded systems legitimately favour trains; SaaS rarely does. Connecting the answer to DORA metrics and feature flags shows senior-level thinking.
