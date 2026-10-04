# Deploying and Verifying — Interview Questions

**Subject:** CI/CD
**Topic:** Preview vs Production, Environment Variables and Secrets, Migrations, Health Endpoints, Smoke Checks, Runbooks, Failures CI Misses
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is the difference between a preview deployment and a production deployment?

**Answer:**

A **preview deployment** is a throwaway copy of the application built from a branch or pull request. It gets its own URL, its own (usually non-production) settings, and lives only as long as it is useful. Reviewers click through the change before it merges.

A **production deployment** is the one the real domain serves to real users.

On platforms such as Vercel and Netlify the split is automatic by default:

| Event | Result |
|---|---|
| Push to a non-production branch, or open a PR | Preview deployment with a unique URL |
| Push or merge to the production branch (usually `main`) | Production deployment |
| `vercel deploy` (no `--prod`) | Preview |
| `vercel deploy --prod` | Production |

**Why it matters:** previews move the "does it actually work deployed?" question before the merge, where it is cheap to answer. They also exercise the real build and runtime, which local `npm run dev` does not.

**Watch out:** previews often share the production database unless you give them their own. A preview that runs migrations or writes data can damage production.

**Interview insight:** say what a preview *doesn't* prove: it runs with preview settings, preview data and usually no real traffic. Production still needs its own post-deploy checks.

### Q2. What is a health endpoint, and what should it check?

**Answer:**

A **health endpoint** is a URL (commonly `/health`, `/healthz` or `/api/health`) that reports whether the application and the things it depends on are working. It returns **200** when healthy and a **5xx** (usually **503 Service Unavailable**) when not.

What a useful one checks:

1. **The process is up** (it answered at all).
2. **Its critical dependencies answer**: the database accepts a trivial query; a cache or queue responds.
3. **Version agreement**: the database schema has every migration this build expects. A schema that is behind the code is a classic post-deploy failure.

A real reply from a small Next.js + Postgres site:

```json
{"ok":true,"data":{"healthy":true,"db":"up","dbError":null,
 "schema":{"status":"current","applied":1,"expected":1,
           "latestExpected":"0000_superb_demogoblin","latestApplied":"0000_superb_demogoblin"}}}
```

What it must **not** do:

- Leak secrets: no connection strings, host names or raw error messages in a public reply. Return codes (`ECONNREFUSED`, a SQLSTATE) instead.
- Be cached: send `Cache-Control: no-store`.
- Be expensive: load balancers may call it every few seconds.

**Liveness vs readiness (Kubernetes):** a liveness probe asks "should this container be restarted?" and should check only the process; a readiness probe asks "should it receive traffic?" and may check dependencies. Mixing them up restarts healthy pods when the database blips.

### Q3. What is a smoke check (smoke test), and how is it different from an end-to-end test suite?

**Answer:**

A **smoke check** is a fast, shallow test of a deployed system: does each important page answer 200, does the health endpoint say healthy, does one key API return valid data. The name comes from hardware: switch it on and check nothing smokes.

| | Smoke check | End-to-end suite |
|---|---|---|
| Runs against | The live deployment (production or preview) | A test build in CI |
| Duration | Seconds | Minutes |
| Depth | Is it up and serving? | Does every user journey work? |
| When | Straight after every deploy | On every PR |
| Data | Real production data (read-only) | Seeded test data |

A smoke check should **exit non-zero** on any failure so a script or pipeline can stop on it, and it should be **read-only** against production.

**Interview insight:** mention that a smoke check needs real data to mean anything. An empty comment list never runs the code that renders comments, so a check that fetches an empty list can pass while every real list returns 500.

### Q4. What is a schema migration, and what is seeding?

**Answer:**

A **schema migration** is a versioned script that changes the database's structure (create a table, add a column, add an index). Migrations are committed with the code, applied in order, and recorded in a table in the database itself (e.g. Drizzle's `__drizzle_migrations`, Rails' `schema_migrations`, Flyway's `flyway_schema_history`), so the tool knows which have run.

**Seeding** loads starter rows the application needs after migrating: reference data, lesson sections, an admin user for a test database.

```bash
pnpm db:generate   # write a new migration from the schema diff (commit it)
pnpm db:migrate    # apply pending migrations to the database in DATABASE_URL
pnpm db:seed       # insert starter rows (idempotent)
```

Key properties:

- **Idempotent seeding:** running it twice must not duplicate rows (`INSERT … ON CONFLICT DO NOTHING`).
- **Migrations are forward-only in practice.** Down-migrations exist but are rarely safe on production data; fix forward instead.
- **CI should migrate and seed a fresh database** before integration and e2e tests, so a broken migration fails the PR.

### Q5. What is an environment variable in a deployment, and what is a "sensitive" (write-only) variable?

**Answer:**

An **environment variable** is a setting the application reads at run time (`DATABASE_URL`, `AUTH_SECRET`, `NEXT_PUBLIC_SITE_URL`). It keeps configuration and secrets out of git and lets one build run in several environments (development, preview, production) with different values.

A **sensitive** (or **write-only**, **secret**) variable can be written and used by deployments but **cannot be read back** through the dashboard, API or CLI after saving. On Vercel, `vercel env pull` returns such variables empty. (Vercel now names the types "Config" and "Secret".)

Consequences:

- Local development needs its own copy of the value (a `.env.local` that is never committed).
- Rotating the value means writing a new one; you can't copy it from the platform.
- Leaks through the dashboard or a compromised CLI session are reduced.

**Build-time vs run-time:** frameworks inline some variables into the client bundle at build time (`NEXT_PUBLIC_*`, `VITE_*`). Anything inlined is public: never put a secret there.

### Q6. What is a runbook, and why write one for a routine deploy?

**Answer:**

A **runbook** is the written, step-by-step procedure for an operation: deploying, restoring a backup, rotating a key, failing over. It says what to run, in what order, what "good" looks like at each step, and what to do when a step fails.

For a routine deploy it might read:

1. If the schema changed, run the migration against production first.
2. Deploy from a clean export of the commit.
3. Run the smoke check against the live URL; it must exit 0.
4. Read the runtime logs for the first 15 minutes.

**Why bother for routine work:**

- The steps that get skipped are the boring ones (step 1 and step 4 above).
- Anyone, including an automated agent, can do it the same way.
- When something goes wrong the runbook is updated, so the fix outlives the person who found it.

**Interview insight:** the strongest answers connect a runbook step to the incident that created it ("step 1 exists because production ran unmigrated for months").

---

## Intermediate

### Q7. Why should you migrate the database before deploying the code that needs the change? What makes that safe?

**Answer:**

If new code goes live first, every request that touches the new column or table fails until the migration runs. If the migration runs first, the **old code** keeps running against the new schema until the deploy completes. That only works when the migration is **backwards-compatible**, which is what the **expand/contract** pattern guarantees:

1. **Expand:** add new tables or nullable columns, new indexes. Old code ignores them.
2. **Deploy** code that writes both old and new shapes (or reads the new with a fallback).
3. **Backfill** existing rows into the new shape.
4. **Switch** reads to the new shape.
5. **Contract** in a later release: drop the old column once nothing uses it.

```sql
-- Release N (expand): safe while release N-1 still runs
ALTER TABLE comments ADD COLUMN body_html text;          -- nullable
-- Release N+1 code fills body_html on write and backfills old rows
-- Release N+2 (contract), once nothing reads body_md alone
ALTER TABLE comments DROP COLUMN body_md;
```

Unsafe in one step: renaming a column, adding a `NOT NULL` column without a default, changing a type. Each breaks the old code that is still serving.

**Interview insight:** say how you'd *know* the order was followed: a health endpoint that compares applied migrations with the build's migration list and returns 503 when the schema is behind.

### Q8. What is "post-deploy verification", and what would you put in it?

**Answer:**

**Post-deploy verification** is the set of checks run against the live system straight after a deploy, before anyone declares it done. CI tested a copy; production has its own database, runtime, settings and traffic.

A practical sequence:

| When | Check | Fails if |
|---|---|---|
| T+0 | Health endpoint | not 200, or schema not current |
| T+0 | Smoke check: key pages and APIs answer 200 | any non-200, non-JSON or empty-when-it-shouldn't-be |
| T+0 | One read of real data through the code path that renders it | render error |
| T+5–15 min | Runtime logs / grouped errors | any new error group |
| T+15–30 min | Error rate and latency vs baseline | regression past a threshold |

Automate the first three as a script with a non-zero exit code; wire it into the deploy job or the runbook. Tie the later checks to an automated rollback where the platform supports it.

### Q9. What is a Git-integrated deploy versus a CLI deploy, and what is a "clean export"?

**Answer:**

With a **Git-integrated** deploy the platform watches the repository and builds every push: preview for branches and PRs, production for the production branch. The deployed code is exactly a commit.

With a **CLI deploy** (`vercel deploy --prod`, `netlify deploy --prod`, `fly deploy`) you upload a directory from your machine or a CI job. The platform builds **whatever is in that directory**, including:

- uncommitted edits,
- an `.env.local` with local secrets (if not excluded),
- a stale `node_modules`, build cache or generated files.

A **clean export** removes that risk: deploy a fresh copy of exactly one commit.

```bash
rm -rf /tmp/deploy && mkdir /tmp/deploy
git archive HEAD | tar -x -C /tmp/deploy   # tracked files only, as committed
cp -r .vercel /tmp/deploy/                 # project link, no secrets
(cd /tmp/deploy && vercel deploy --prod --yes)
```

**When CLI deploys are reasonable:** the Git integration isn't connected yet, the platform's GitHub app lacks access, or the deploy must follow a manual step (a production migration). Record the deployed commit either way.

### Q10. What is an OAuth callback URL, and why do domain moves break sign-in?

**Answer:**

In the OAuth authorization-code flow the identity provider (GitHub, Google…) sends the user back to the application's **callback URL** (also called redirect URI) with a one-time code: e.g. `https://app.example.com/api/auth/callback/github`.

The provider only redirects to URLs registered for that OAuth app. So when the site moves to a new domain:

1. Users sign in, the provider refuses to redirect (or redirects to the old domain), and sign-in fails.
2. Fix: register the new callback URL on the provider, and update the app's own setting of its public URL (`AUTH_URL`, `NEXTAUTH_URL`, `NEXT_PUBLIC_SITE_URL`).

Related: the **old domain** should answer **308 Permanent Redirect** to the new one, preserving the path. A 308 (like 307) keeps the HTTP method and body; a 301 may turn a POST into a GET in some clients.

**Watch out:** preview deployments have their own URLs, which the provider usually doesn't know. Test OAuth on a fixed staging domain, or stub the provider in e2e tests.

### Q11. A team's site "worked locally and in CI" but every request to one route returns 500 in production. How do you investigate?

**Answer:**

This is the **works locally, fails on the platform** class: production's runtime differs from CI's.

1. **Read the production error, not a reproduction.** Runtime logs or the platform's grouped errors give the stack and the count (e.g. `ERR_REQUIRE_ESM`, 14 occurrences, one route).
2. **List the differences** between CI and production for that route: Node version; how the platform loads modules (bundled vs external packages, its own function loader); file system (read-only, no `/tmp` persistence); environment variables; region and database.
3. **Reproduce the difference locally.** For the ES-module case: modern Node (20.19+, 22.12+) allows `require()` of an ES module, but the platform's loader did not. `node --no-experimental-require-module` makes local Node refuse it too, which reproduced the 500.
4. **Fix, then guard.** Replace the dependency with one the bundler can package, and run CI's e2e server under the same flag so that the class of bug fails a PR.
5. **Extend the smoke check** so the route is exercised after every deploy, with real data.

**Interview insight:** the guard in step 4 matters more than the fix. The question an interviewer wants answered is "how do you stop the next one?"

### Q12. What does "render before write" mean, and what failure does it prevent?

**Answer:**

**Render (or validate) before write** means doing every step that can fail *before* storing anything.

The failure it prevents, from a real comment endpoint:

1. `POST /comments` inserts the comment.
2. It then renders the Markdown to HTML for the reply. The renderer crashes: 500.
3. The user sees an error and presses Post again. Steps 1–2 repeat.
4. Result: three identical stored comments and three errors.

Reordered:

```ts
const html = renderComment(body);          // may throw: nothing stored yet
const saved = await db.insert(comments).values({ body, html }).returning();
return json({ ok: true, data: saved });
```

General forms of the rule: validate input before side effects; compute derived values before the transaction; make retried writes **idempotent** (an idempotency key from the client); and have the client keep the user's draft on any non-OK reply rather than assuming success.

---

## Advanced

### Q13. A production database ran unmigrated for months and nobody noticed. How could that happen, and what would you put in place?

**Answer:**

How it happens: a **silent fallback**. To let a fresh clone run without a database, the data layer caught database errors and returned empty data. In production, the database had never been migrated, so every query failed and every page showed empty lists, with no error in the logs and nothing red in CI (CI used its own, migrated database).

Controls, from cheapest:

1. **A health endpoint that refuses to fall back.** It queries the migrations table and returns 503 for a schema that is `behind` or `none`.
2. **A deploy runbook with migrate-first as step 1** and the smoke check (which includes health) as step 3.
3. **Count fallbacks.** Every time the fallback fires, increment a counter visible on an admin page or metric. A healthy check next to a non-zero fallback count means the database was failing recently.
4. **Fail loudly in production.** Allow the fallback only in development; in production, log at error level or let the request fail.
5. **Alert on the absence of expected data** (zero comments in a week on a busy site) as a last line.

**Interview insight:** generalise: every fallback, retry or default is a place where a failure can hide. Each needs a signal that it happened.

### Q14. How would you add a post-deploy smoke check to a pipeline so that a failed check actually stops something?

**Answer:**

1. **Write it as a script with an exit code**, read-only, against a URL argument:

```bash
pnpm smoke https://app.example.com            # exits 1 on any failure
pnpm smoke https://app.example.com --comments-section 07-sampling
```

2. **Run it in the deploy job, after the deploy step**, against the URL the deploy produced (preview or production).
3. **Make failure do something:**
   - Preview: fail the check on the PR (and make it a required check).
   - Production with progressive delivery: halt the canary or roll back automatically.
   - Production without it: page the on-call and run the rollback step of the runbook.
4. **Keep it fast** (< 1 minute) so nobody is tempted to skip it, and **deterministic** against real data: point data-dependent checks at records that are known to exist, and fail if the list is unexpectedly empty.
5. **Version it with the app**, so new routes get smoke coverage in the same PR.

### Q15. Lighthouse CI fails a PR on performance, but the change was a copy edit. What do you do?

**Answer:**

Lighthouse scores vary from run to run: CPU contention on shared runners, network jitter, third-party scripts. Before trusting a failure:

1. **Look at the reports, not the score.** LHCI uploads per-run reports; compare the failing metric (LCP, TBT, CLS) between this run and `main`.
2. **Check the gate's design:** it should take the **median of several runs** (`numberOfRuns: 3` or 5), on a production build (`next build && next start`), with thresholds that leave headroom (e.g. ≥ 0.9 when `main` scores 0.97+).
3. **Re-run once** if the variance explains it, and note it. Repeated flakes mean the gate needs fixing (more runs, a quieter runner, assertions on specific metrics with budgets instead of the composite score).
4. **If it's real**, it often is: a copy edit can change the LCP element (a longer hero heading), add a web font or shift layout.

Remember what the gate measures: Lighthouse is a **lab** test. Core Web Vitals (LCP ≤ 2.5 s, INP ≤ 200 ms, CLS ≤ 0.1 at the 75th percentile) are measured from real users, and a lab run cannot measure INP at all (Total Blocking Time stands in). Treat LHCI as a regression alarm, and field data as the truth.

### Q16. A library repo is installed by five downstream repos from git. How do you keep all of them green, and what are the trade-offs of pinning?

**Answer:**

Two ways to depend on a git-hosted library:

| | Track a branch | Pin a commit |
|---|---|---|
| Spec | `pkg @ git+https://…/lib` | `pkg @ git+https://…/lib@<sha>` |
| Upstream change reaches downstream | On the next install | When someone bumps the pin |
| A breaking upstream push | Turns downstream red overnight, with no downstream commit | Breaks nothing until the bump PR, where CI catches it |
| Reproducibility | Low: the same downstream commit can pass today and fail tomorrow | High |

Keeping them green:

1. **Run downstream CI after upstream changes.** Manually (`gh run rerun <id>`; a re-run uses the same downstream commit but re-resolves an unpinned install) or automatically (`repository_dispatch` from the upstream's workflow, or a scheduled nightly run).
2. **Beware stale environments.** pip skips a `pkg @ git+…` that is already installed, so a persistent agent's reused venv keeps testing the old upstream. Use fresh environments, or `pip install --force-reinstall --no-deps "pkg @ git+…"`.
3. **If pinning, automate the bumps** (Dependabot/Renovate) so the pin doesn't rot, and let the bump PR's CI be the integration test.
4. **Give the library its own contract tests** for what the downstreams rely on.

### Q17. Your team says "CI is green". What exactly should that mean before a merge or a release?

**Answer:**

**Green at HEAD**: the newest commit on the branch (its HEAD) has passed **every required check**, on that commit, not an earlier one.

Ways "green" lies:

- **An older commit was green**; a later push hasn't finished, or a check was cancelled.
- **A check was renamed.** Required checks match by name, so a renamed job leaves the rule waiting for a check that never reports, while the new check passes but isn't required.
- **The PR is green but `main` moved.** Without "require branches to be up to date" (strict), two individually green PRs can merge into a red `main`. A merge queue fixes this at scale.
- **A dependency moved** (Q16): green yesterday, red on re-run today.
- **Skipped is not passed.** Path filters and `if:` conditions can skip a required job; know whether your platform counts a skipped check as success.
- **Production differs** (Q11): green CI says nothing about the runtime it didn't run.

Signals that keep it honest: a **CI status badge** on the README for `main`, required checks enforced by **branch protection or a ruleset**, and a post-deploy smoke check for the part CI can't see.

### Q18. Compare GitHub branch protection rules with repository rulesets. When would you choose each?

**Answer:**

Both enforce the same kinds of rules on branches: require a PR, required reviews, **required status checks**, block **force pushes** and deletions, require linear history or signed commits.

| | Branch protection rules | Rulesets |
|---|---|---|
| Scope | One rule per branch pattern | Several rulesets can target the same branch; rules **aggregate** and the most restrictive wins |
| Visibility | Admins (others see only the effect) | Anyone with read access can see active rulesets |
| Turning off | Delete the rule | Set enforcement to disabled (or evaluate mode on some plans) without deleting |
| Who may skip | "Do not allow bypassing" / include administrators | An explicit **bypass list** (roles, teams, apps) |
| Org-wide | No | Yes, on paid organisation plans (check the docs) |
| Tags and pushes | Branches only | Branches, tags, and push rules (file paths, sizes) on some plans |

Both can apply at once; all applicable rules are enforced.

**Choose rulesets** for new setups: they layer, they are auditable by every developer, and the bypass list is explicit. An empty bypass list means even the owner's direct push is refused (`GH013 … Changes must be made through a pull request`), which is the point. **Keep branch protection** where existing automation reads or writes it through the API, until it is migrated.

Plan availability differs, especially for private repositories; check [About rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets).

**Interview insight:** mention the operational gotcha: required checks match the **check name** (on GitHub Actions, the job's `name:` plus any matrix values). Renaming a job without updating the rule blocks every PR.

---

**See also:** the [Introduction to CI/CD](https://brendanjameslynskey.github.io/Introduction_to_CI_CD/) deck explains each of these terms on a slide (Protecting main, CI Across Repositories, Deploying Safely, After the Deploy, the case study, and the Key terms glossary).
