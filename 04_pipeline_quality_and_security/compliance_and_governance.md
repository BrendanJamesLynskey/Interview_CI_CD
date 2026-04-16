# Compliance and Governance — Interview Questions

**Subject:** CI/CD
**Topic:** Audit Trails, Approval Workflows, SOC 2 Controls, Change Management, Separation of Duties
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is an audit trail in the context of CI/CD, and what should it capture?

**Answer:**

An **audit trail** is an immutable, queryable record of what happened, when, and by whom in the CI/CD system. Auditors, incident responders, and regulators rely on it to answer "who did X and when" long after the fact.

**What it should capture:**

| Event category | Examples |
|----------------|----------|
| **Source changes** | Commit SHA, author, signature, PR number, reviewers |
| **Pipeline runs** | Workflow file, trigger, actor, start/end time, outcome |
| **Approvals** | Environment deploy approvals, PR reviews, exception grants |
| **Credentials** | Secret reads, OIDC token issuance, IAM role assumptions |
| **Artefacts** | Builds, signatures, registry pushes and pulls |
| **Deployments** | Image digest deployed, target environment, timestamp |
| **Infrastructure** | Terraform applies, K8s changes, DB migrations |
| **Access changes** | Repo permission changes, secret rotations, role assignments |

**Sources of audit data:**

- GitHub Audit Log (org/enterprise level, via API)
- Cloud provider audit (CloudTrail, GCP Audit Logs, Azure Activity Log)
- Artifact registry logs (ECR, Artifactory audit)
- Deployment system logs (ArgoCD, Spinnaker)
- Identity provider logs (Okta, Azure AD)

**Aggregation pattern:**

```
[GitHub Audit Log API] --------+
[CloudTrail]                    |
[Artifactory audit]             +--> [SIEM (Splunk, Datadog, Panther)] --> [Dashboards + alerts]
[ArgoCD events]                 |                     |
[Kubernetes audit]  ------------+                     v
                                              [WORM storage (3-7 years)]
```

**Properties the trail must have:**

1. **Immutable.** Once written, cannot be modified or deleted by users.
2. **Append-only.** New entries add; no updates.
3. **Time-synchronised.** Clocks across sources should agree (NTP).
4. **Retained.** SOC 2 expects ~1 year; HIPAA 6 years; PCI-DSS 1 year online + retained.
5. **Queryable.** Auditors need to answer ad-hoc questions ("who approved this deploy?") in minutes.

**Anti-pattern:** relying solely on each system's default logs. "We have logs" is not an audit trail if no one has centralised them, retention is inconsistent, and queries cross-system are impossible.

### Q2. What is SOC 2, and which controls are most relevant to CI/CD?

**Answer:**

**SOC 2** (System and Organization Controls 2) is an auditing framework maintained by the AICPA. It evaluates controls at service organisations against five "Trust Service Criteria":

- **Security** (mandatory) — protection against unauthorised access
- **Availability** — system is accessible as committed
- **Processing Integrity** — system processing is complete, valid, accurate
- **Confidentiality** — information designated confidential is protected
- **Privacy** — personal information handled per commitments

**SOC 2 Type I** = "controls were designed appropriately as of date X."
**SOC 2 Type II** = "controls operated effectively over a period (6-12 months)." Type II is the one customers want.

**Controls most relevant to CI/CD (from the Common Criteria):**

| Control | What it means for CI/CD |
|---------|------------------------|
| **CC6.1** Logical access controls | CI/CD access is restricted; MFA on GitHub, cloud |
| **CC6.2** User registration/de-registration | Leavers' access removed within SLA; access reviews quarterly |
| **CC6.3** Access role-based | Principle of least privilege; per-env roles |
| **CC7.1** Change management | Every prod change goes through documented process |
| **CC7.2** Infrastructure changes | IaC reviewed, tested, logged |
| **CC8.1** System changes authorised | PR approvals, deploy approvals, audit trail |
| **CC3.2** Risk assessment | Supply chain risk, third-party actions |
| **A1.2** Environmental protections | Backups, DR drills |

**What auditors actually ask for in CI/CD interviews:**

- Evidence of PR reviews on last 25 production changes
- Evidence of deploy approvals on last 10 production deploys
- Access review evidence (who has admin? approved by whom, when?)
- Evidence of a rollback drill in the last quarter
- Incident response runbook + evidence of one executed incident
- Secret rotation evidence
- List of users who changed CI/CD configuration in the audit period

**Interview insight:** senior engineers in enterprise-facing companies are expected to speak SOC 2. Naming specific control numbers (CC7.1 change management, CC8.1 change authorisation) and mapping them to GitHub features (branch protection, environments, audit log) shows you've been through an audit, not just read about one.

### Q3. What is separation of duties, and how does it apply to CI/CD pipelines?

**Answer:**

**Separation of duties** (SoD) is the principle that no single person should be able to perform a sensitive operation end-to-end. It protects against both malicious actors (a compromised individual can't act alone) and honest mistakes (a second pair of eyes catches errors).

**SoD in CI/CD:**

| Duty | Separated from |
|------|----------------|
| Writing code | Approving it (peer review requirement) |
| Approving a PR | Deploying the resulting build (deploy approval) |
| Authoring CI config | Running it with production credentials |
| Managing secrets | Using secrets in workflows |
| Having repo admin | Having cloud admin |
| Requesting production deploys | Approving production deploys |

**Implementation examples:**

**1. PR review enforcement:**

```yaml
# GitHub branch protection
required_pull_request_reviews:
  required_approving_review_count: 2
  require_code_owner_reviews: true
  dismiss_stale_reviews: true
```

A developer cannot approve their own PR.

**2. Deploy approval gating:**

```yaml
# GitHub Environment: production
deployment_branch_policy: { custom_branches: ["main", "release/*"] }
required_reviewers:
  - team: prod-deployers
  - prevent_self_review: true
```

The developer who triggered the deploy cannot approve it.

**3. Secret access separation:**

- Vault admin role (manages secrets) is distinct from Vault consumer role (uses secrets)
- CI's OIDC role has read-only access to specific secrets, not wildcards

**4. Repo vs cloud admin split:**

- GitHub org admins manage repos, users, branch protection
- Cloud account admins manage IAM, resources
- No single user has both; emergency access via break-glass with audit

**Pitfalls:**

**Single point of compromise:**

```yaml
# BAD — one role can do everything
permissions:
  contents: write
  packages: write
  id-token: write
  deployments: write
  secrets: read
  pull-requests: write
```

If this token leaks, the attacker has end-to-end power. Better: per-job scoped tokens.

**False separation:**

- Developer and "reviewer" are the same person on different accounts
- Developer and reviewer are married to each other
- One team owns both code and deploy approvals

Auditors probe these. Sub-team or cross-team reviews are stronger.

**Emergency breaks:**

Real incidents sometimes need one person to act fast. Build break-glass procedures that:

- Log the access as an emergency event
- Notify multiple people in real-time
- Expire access after N hours automatically
- Trigger post-hoc review within 48 hours

**Interview insight:** SoD is the control that distinguishes "we have CI/CD" from "we have auditable CI/CD." Mentioning `prevent_self_review`, CODEOWNERS, and break-glass procedures demonstrates practical understanding.

### Q4. What is change management, and how does a modern CI/CD pipeline fit into it?

**Answer:**

**Change management** is the process by which proposed changes to production systems are reviewed, approved, scheduled, executed, and retrospected. In traditional ITIL environments, it was manual: CAB (Change Advisory Board) meetings, change tickets, maintenance windows. In modern CI/CD, most of this is automated — but the *intent* remains.

**Change categories (ITIL, widely used in audits):**

| Type | Definition | Example | Approval |
|------|-----------|---------|----------|
| **Standard** | Pre-approved change type; low risk | Patching already-tested library, routine cert renewal | Automatic |
| **Normal** | Non-standard but not urgent; needs review | New feature deploy, new service | PR + CAB / async CAB |
| **Emergency** | Production-impacting, needs immediate action | Patching active exploit | Abbreviated; post-hoc review mandatory |

**How CI/CD implements change management:**

**Standard change — fully automated:**

```yaml
# Dependabot patch update → auto-merge → auto-deploy
- if: steps.meta.outputs.update-type == 'version-update:semver-patch'
  run: gh pr merge --auto --squash "$PR_URL"
```

The "change approval" is the pre-approval of the type (e.g., "all patch-level dependency updates to approved libraries are pre-authorised").

**Normal change — PR + deploy approval:**

```yaml
# deploy-prod workflow
on: workflow_dispatch
inputs:
  change_ticket: { required: true, type: string }
jobs:
  validate:
    steps:
      - name: Verify CHG ticket is approved
        run: |
          status=$(jira get ${{ inputs.change_ticket }} --field=status.name)
          [ "$status" = "Approved" ] || exit 1
  deploy:
    needs: validate
    environment: production   # required reviewers
```

The PR review + CHG ticket + deploy approval together constitute the change record.

**Emergency change — break-glass:**

```yaml
on: workflow_dispatch
inputs:
  emergency_reason: { required: true, type: string }
jobs:
  deploy:
    environment: production-emergency   # single approver for speed
    steps:
      - run: deploy.sh
      - name: File post-hoc review
        if: always()
        run: |
          gh issue create --title "Emergency change review: ${{ github.run_id }}" \
            --label change-mgmt,emergency \
            --body "Reason: ${{ inputs.emergency_reason }}. Review within 48 hours."
```

The emergency path is faster but mandates follow-up review.

**Evidence the audit trail must produce:**

For any production change:

- Who proposed it (PR author)
- Who reviewed it (PR approvers)
- Who authorised it (deploy approvers)
- What ticket cross-references it (CHG-123)
- When it was deployed (workflow run ID, timestamp)
- What bytes were deployed (artefact digest)
- What the outcome was (success/rollback)

**Integration with ITSM tools:**

Many orgs bridge CI/CD and ITSM (ServiceNow, Jira Service Management):

```yaml
- name: Open CHG ticket
  run: |
    chg_id=$(servicenow create-change \
      --risk=low --impact=low \
      --requested-by="${{ github.actor }}" \
      --description="Deploy of ${{ github.sha }}")
    echo "CHG=$chg_id" >> $GITHUB_ENV

- run: deploy.sh

- name: Close CHG ticket
  if: success()
  run: servicenow close-change "$CHG" --outcome=successful
```

The change record lives in the system of record (ServiceNow); the deploy record lives in CI; they cross-reference.

**Interview insight:** traditional-enterprise interviewers (finance, healthcare, public sector) want to hear ITIL vocabulary (standard/normal/emergency, CAB, CHG). Modern-tech interviewers want to hear "auto-merge with audit trail." Read the room; the same CI/CD can support both framings.

### Q5. What is the difference between detective, preventative, and corrective controls?

**Answer:**

The three categories of security/compliance controls differ by timing relative to the event.

| Category | When it acts | Examples in CI/CD |
|----------|-------------|---------------------|
| **Preventative** | Before the event | Branch protection, push protection, signed commits required |
| **Detective** | During or shortly after | Secret scanning alerts, drift detection, SIEM rules |
| **Corrective** | After detection | Secret rotation, image rollback, account lockout |

**Preventative controls in CI/CD:**

```yaml
# Branch protection — prevents direct pushes
required_pull_request_reviews:
  required_approving_review_count: 2

# Required signed commits — prevents unsigned
required_signatures: true

# Required status checks — prevents merging on red
required_status_checks: [lint, test, secret-scan]

# Required admission policy — prevents unsigned images from running
# (Kyverno policy rejects pods with unverified images)
```

**Detective controls:**

```yaml
# Secret scanning — detects committed secrets
# GitHub native, alerts in real-time

# Daily flake analyser (testing_in_pipelines Q13)
# Drift detection (testing_in_pipelines Q15)

# SIEM rule: alert on unusual workflow behaviour
# E.g., deploy outside business hours, deploy by non-deployer team member
```

**Corrective controls:**

```yaml
# Automated secret rotation after detection
on:
  secret_scanning_alert: { types: [created] }
jobs:
  rotate:
    steps:
      - name: Rotate leaked AWS key
        run: aws iam delete-access-key --access-key-id ${{ github.event.alert.secret }}
      - run: notify-security "Rotated key ${{ github.event.alert.secret }}"
```

```yaml
# Automated rollback on SLO breach
on:
  repository_dispatch:
    types: [slo-breach]
jobs:
  rollback:
    steps:
      - run: kubectl rollout undo deployment/app
```

**Defence in depth = layering all three:**

- Preventative alone fails: every control has edge cases
- Detective alone fails: you notice but can't stop damage
- Corrective alone fails: you're always reacting, not preventing

Good CI/CD has layered examples of each:

| Threat | Preventative | Detective | Corrective |
|--------|--------------|-----------|------------|
| Secret leak | Push protection | Secret scanning | Auto-rotate |
| Malicious deploy | Deploy approval | Audit log alert | Rollback |
| Vulnerable dep | Dependabot, CI scan | Runtime CVE monitoring | Patch + redeploy |
| Unauthorised access | MFA + SSO | Audit log review | Access revocation |

**Interview insight:** the three-category framework is common vocabulary in security/audit interviews. Framing your answers in terms of "we prevent X with Y, detect with Z, correct with W" signals security literacy.

### Q6. What is the principle of least privilege, and how do you apply it to CI pipelines?

**Answer:**

**Least privilege** means every identity (user, service, workflow) should have the minimum permissions needed to do its job, and nothing more. A compromised or misbehaving identity can do less damage.

**Application in CI pipelines:**

**1. Workflow permissions — default to none:**

```yaml
# Workflow default
permissions: {}

jobs:
  build:
    permissions:
      contents: read          # read source
      packages: write         # push image
      id-token: write         # OIDC
    # not: write-all, actions: write, etc.
```

Compare to the anti-pattern:

```yaml
# BAD — everything the token can do
permissions: write-all
```

**2. Per-job tokens:**

Each job gets its own scoped token, not a shared one:

```yaml
jobs:
  test:
    permissions: { contents: read }
  build:
    permissions: { contents: read, packages: write }
  deploy:
    permissions: { contents: read, id-token: write, deployments: write }
```

**3. Cloud IAM roles per workflow:**

```hcl
# Terraform — per-workflow role
resource "aws_iam_role" "github_ci_build" {
  name = "github-ci-build"
  assume_role_policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Principal = { Federated = "arn:aws:iam::123:oidc-provider/token.actions.githubusercontent.com" }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:sub" = "repo:my-org/app:ref:refs/heads/main"
        }
      }
    }]
  })
}

# Build role: read-only to deploy targets, write to build artefacts
resource "aws_iam_role_policy" "github_ci_build" {
  policy = jsonencode({
    Statement = [
      { Effect = "Allow", Action = "s3:PutObject", Resource = "arn:aws:s3:::build-artefacts/*" },
      { Effect = "Allow", Action = "ecr:PutImage", Resource = "arn:aws:ecr:eu-west-1:123:repository/myapp" }
    ]
  })
}
```

A separate `github-ci-deploy` role has different permissions. Build role can't deploy; deploy role can't build.

**4. Branch- and environment-scoped:**

```
"token.actions.githubusercontent.com:sub" = "repo:my-org/app:environment:production"
```

Only workflow runs targeting the `production` environment can assume this role. PR workflows cannot.

**5. Secret scope:**

- Repository secrets: visible to all workflows in the repo
- Environment secrets: visible only when targeting that environment
- Organisation secrets: with repo access controls

Prefer environment secrets for production credentials. Prefer OIDC over secrets for cloud access.

**6. Minimise third-party action access:**

```yaml
# Actions run with the current job's permissions
- uses: third-party/action@sha
  # inherits permissions — so ensure those are tight

# Pin to SHA to prevent tag-swap attacks
- uses: third-party/action@a1b2c3d4...
```

**What to audit regularly:**

- Workflow files: check for `permissions: write-all`
- IAM roles: look for overly broad `Resource: "*"` or `Action: "*"`
- Secrets: who added each, when, when last used
- Third-party actions: are they still pinned? Still maintained?

**Interview insight:** least privilege is mentioned so often it's cliché. What distinguishes a strong answer is *operational specifics*: per-job permissions, per-workflow IAM roles, branch-scoped OIDC, SHA-pinned actions. The cliché is that everyone says it; the signal is whether you've actually configured it.

---

## Intermediate

### Q7. How do you design approval workflows that balance security with developer velocity?

**Answer:**

The trade-off: heavy approvals create compliance evidence but slow deploys; light approvals speed deploys but weaken audit story. A well-designed approval model uses *risk-based routing* — the amount of approval scales with the risk of the change.

**Risk classification:**

| Risk | Examples | Approvals required |
|------|----------|-------------------|
| Very low | Doc changes, test-only changes | Automated CI, no human deploy approval |
| Low | Patch dependency updates, config value tweaks in non-critical paths | 1 PR reviewer, auto-deploy after CI |
| Medium | Feature adds behind flag, schema-compatible DB migrations | 1 PR reviewer, 1 deploy approver |
| High | Feature releases (flag flip), breaking schema changes, IAM changes | 2 PR reviewers (incl. CODEOWNERS), 2 deploy approvers, PRE review |
| Critical | Secrets, production data handling, safety-critical changes | Multi-team PR review, CAB/architecture review, scheduled deploy window |

**Implementation — per-risk GitHub Environments:**

```yaml
# Low-risk: config-only change
deploy-config:
  environment: production-config    # 1 reviewer, self-approval allowed
  steps: [...]

# Medium-risk: standard deploy
deploy-app:
  environment: production           # 1 reviewer, prevent_self_review
  steps: [...]

# High-risk: schema change
deploy-schema:
  environment: production-schema    # 2 reviewers, incl. a DBA
  steps: [...]
```

GitHub Environments' "required reviewers" + "deployment branch policies" can implement each tier.

**Risk classification lives in code:**

```yaml
# .github/change-policy.yml
low_risk_paths:
  - "docs/**"
  - "**/*.md"
  - "tests/**"
medium_risk_paths:
  - "src/services/**"
high_risk_paths:
  - "src/payments/**"
  - "migrations/**"
  - "terraform/production/**"
```

A CI job reads the policy, checks the PR's changed paths, and sets required reviewers / target environment dynamically.

**Fast-path for low-risk:**

```yaml
# If only docs changed, skip approval entirely
jobs:
  classify:
    outputs: { risk: ${{ steps.classify.outputs.risk }} }
    steps:
      - id: classify
        run: |
          changed=$(git diff --name-only origin/main)
          if echo "$changed" | grep -qvE '\.md$|^docs/'; then
            echo "risk=normal" >> $GITHUB_OUTPUT
          else
            echo "risk=trivial" >> $GITHUB_OUTPUT
          fi

  deploy:
    needs: classify
    environment: ${{ needs.classify.outputs.risk == 'trivial' && 'auto' || 'production' }}
```

**Approval fatigue mitigation:**

If approvers rubber-stamp because volume is high, approvals lose value. Symptoms:

- Median approval time < 5 minutes
- Approvers don't comment
- Same 2-3 people approve everything

Mitigations:

- Batch low-risk changes into fewer approvals
- Pre-approve "standard change types" (Q4)
- Rotate approvers (different CODEOWNERS per sub-team)
- Track "rejected PRs" as a quality signal — if none are ever rejected, approvals are theatre

**Cultural point:**

A good approval model trusts developers for low-risk changes and scales scrutiny with risk. A bad model approves everything identically, either creating friction or devolving to rubber-stamping.

**Interview insight:** the answer "one approval for every deploy" fails this question. Senior engineers articulate the risk gradient and defend specific decisions (why auto-approve docs? why 2 approvers for schema?). This is a system-design answer, not a tooling answer.

### Q8. How do you produce evidence for a SOC 2 auditor asking "show me the last 30 production deploys, with who approved each"?

**Answer:**

The goal: a single query or report that the auditor can verify is complete, accurate, and not hand-selected.

**Query against GitHub Deployments API:**

```bash
#!/bin/bash
gh api graphql -f query='
{
  repository(owner: "my-org", name: "app") {
    deployments(first: 30, orderBy: {field: CREATED_AT, direction: DESC}) {
      nodes {
        commitOid
        createdAt
        environment
        creator { login }
        latestStatus { state logUrl }
      }
    }
  }
}'
```

For each deployment, cross-reference to the approval:

```bash
# For deployment with workflow run ID
gh api repos/my-org/app/actions/runs/$RUN_ID/approvals
# Returns list of approvers with timestamps
```

**Aggregated report:**

```python
# produce_deploy_report.py
import subprocess, json
from datetime import datetime

deploys = fetch_last_n_deploys("my-org/app", "production", n=30)
report = []
for d in deploys:
    approvals = fetch_approvals(d.workflow_run_id)
    report.append({
        "deployed_at": d.created_at,
        "version": d.commit_oid[:8],
        "deployer": d.creator,
        "approvers": [a.user for a in approvals],
        "outcome": d.latest_status,
        "pr": find_pr_for_commit(d.commit_oid),
        "change_ticket": extract_change_ticket(d.description),
    })

print_table(report)
```

**Expected output (what the auditor wants):**

```
DATE                  VERSION   DEPLOYER      APPROVERS              OUTCOME    PR    CHG
2025-04-14 15:32 UTC  a1b2c3d   alice@myco    bob@myco, carol@myco   success    #912  CHG-801
2025-04-14 11:07 UTC  5d4e3f2   bob@myco      alice@myco, david@myco success    #910  CHG-799
...
```

**Properties the report must demonstrate:**

1. **Completeness.** Querying the API is complete by design; no cherry-picking possible. If you produce the report from ad-hoc Slack screenshots, the auditor will push back.

2. **Tamper-evidence.** The underlying GitHub audit log is append-only and immutable to users. Auditors accept this as primary evidence.

3. **Cross-referencing.** Every deploy has a PR and a CHG ticket. The auditor samples N deploys and verifies each chain end-to-end.

4. **No gaps.** If there were 30 deploys in the period, show all 30. If the auditor finds a 31st that doesn't appear, confidence evaporates.

**Preparation for the audit:**

- Automate this report (weekly run, archived to evidence folder)
- Store outputs in a WORM bucket (Object Lock on S3) for retention
- Produce evidence throughout the period, not just before the audit

```yaml
# .github/workflows/weekly-audit-evidence.yml
on:
  schedule: [{ cron: "0 6 * * 1" }]
jobs:
  report:
    steps:
      - run: python produce_deploy_report.py > report-$(date -I).csv
      - run: aws s3 cp report-*.csv s3://audit-evidence/deploys/
```

**Typical auditor follow-ups:**

- "Show me a deploy where the approver was the same as the deployer" → should be zero (prevent_self_review)
- "Show me a deploy without a PR reference" → should be zero (workflow requires it)
- "Show me a deploy after hours; was there a change ticket?" → probably an emergency; show the CHG + post-hoc review
- "Show me when each approver last reviewed" → evidence of active engagement

**Interview insight:** "how do you produce audit evidence" is a question for platform, infrastructure, and security engineers at enterprise-facing companies. The answer that wins is "we generate the evidence continuously and automatically, so it's available on demand rather than scrambled together at audit time." That's a mature operational posture, not a heroic audit-week effort.

### Q9. What is an "immutable audit log," and why does it matter for compliance?

**Answer:**

An **immutable audit log** is one that cannot be edited or deleted after an entry is written, including by the system administrators. This is foundational for compliance — auditors need to trust that the logs they see reflect what actually happened.

**Why ordinary logs fail:**

- A compromised admin can delete traces of their actions
- Application bugs can overwrite log rows
- Log aggregators may de-duplicate or drop entries
- Retention policies may prune evidence before an audit

**Properties of an immutable audit log:**

1. **Append-only.** Writes add new entries; no updates, no deletes.
2. **Tamper-evident.** Modification is detectable (cryptographic hashing, Merkle trees).
3. **Retained.** Entries persist for the regulated duration (SOC 2: 1 year; HIPAA: 6 years).
4. **Access-logged.** Reads are themselves logged.
5. **Separated.** Log storage is outside the administrative boundary of the systems being logged.

**Implementation patterns:**

**Pattern A — Cloud object storage with Object Lock / retention:**

```hcl
# AWS S3 bucket with Object Lock
resource "aws_s3_bucket" "audit" {
  bucket = "myco-audit-logs"
  object_lock_enabled = true
}

resource "aws_s3_bucket_object_lock_configuration" "audit" {
  bucket = aws_s3_bucket.audit.id
  rule {
    default_retention {
      mode = "COMPLIANCE"   # cannot be shortened even by root account
      days = 2555            # 7 years
    }
  }
}
```

`COMPLIANCE` mode means even the AWS root user cannot shorten retention. Genuinely immutable for the retention period.

**Pattern B — Append-only database:**

```sql
-- Grant only INSERT, no UPDATE or DELETE
GRANT INSERT ON audit_log TO ci_writer;
REVOKE UPDATE, DELETE ON audit_log FROM ALL;
```

Works for structured logs; fragile without careful RDBMS config.

**Pattern C — Blockchain / transparency log:**

```
Rekor (Sigstore) — append-only Merkle tree
Certificate Transparency — append-only logs for TLS certs
AWS QLDB — Quantum Ledger Database, cryptographically verifiable history
```

**Pattern D — SIEM with WORM storage:**

```
[CI/CD events] -> [SIEM (Splunk, Datadog, Panther)] -> [WORM tier (S3 Object Lock, Azure immutable blob)]
```

**Chain of custody:**

For audit evidence, you need to demonstrate:

1. The event happened (logged at source)
2. The log was transmitted intact (TLS, signed)
3. The log was stored intact (WORM storage, hashing)
4. The log was retrieved unmodified (storage hash matches captured hash)

**Integration with CI/CD:**

```yaml
# Ship audit events to WORM log
- name: Record deploy to audit
  run: |
    echo "{\
      \"timestamp\":\"$(date -Iseconds)\",\
      \"event\":\"deploy\",\
      \"actor\":\"${{ github.actor }}\",\
      \"commit\":\"${{ github.sha }}\",\
      \"environment\":\"production\",\
      \"workflow_run\":\"${{ github.run_id }}\"\
      }" | \
      aws s3 cp - s3://myco-audit-logs/deploy/${{ github.run_id }}.json \
        --sse AES256 --object-lock-mode COMPLIANCE \
        --object-lock-retain-until-date $(date -d '+7 years' -I)
```

**Why auditors care:**

The common compromise scenario: insider attempts fraud, covers tracks by deleting logs. An immutable log prevents this. Auditors ask "what prevents an admin from deleting logs?" — answering "Object Lock in compliance mode" or "ledger database with cryptographic verification" satisfies the question. Answering "our retention policy" does not.

**Interview insight:** the phrase "Object Lock in compliance mode" is specific and signals you've operated this. Generic "we log everything" doesn't; an auditor would dig deeper.

### Q10. How do you audit access to production systems via CI/CD, and what's the difference between user-level and system-level auditing?

**Answer:**

Access auditing answers "who accessed what, when, why." In CI/CD, "who" is often a workflow (not a human), creating a layered question: who triggered the workflow, which workflow ran, and what did it do.

**User-level auditing:**

| What | Where logged |
|------|-------------|
| User logged into GitHub | GitHub audit log |
| User pushed a commit | GitHub audit log + Git log |
| User clicked "approve" on a deploy | GitHub audit log + environment approval |
| User accessed a secret | Vault audit log, AWS IAM CloudTrail |
| User assumed an IAM role | CloudTrail `AssumeRole` event |
| User SSH'd to a host | Host audit log, CloudTrail for SSM |

**System-level (workflow-level) auditing:**

| What | Where logged |
|------|-------------|
| Workflow started | GitHub Actions run log |
| Workflow assumed an IAM role via OIDC | CloudTrail `AssumeRoleWithWebIdentity` |
| Workflow read a secret | Vault audit log (with OIDC identity) |
| Workflow pushed an image | ECR / GHCR audit |
| Workflow deployed to Kubernetes | K8s audit log + ArgoCD events |

**The chain to reconstruct from a deploy:**

```
[human Alice] → [triggered workflow run #456] → [workflow assumed role ci-deploy]
                                              → [role read secret prod-db-password]
                                              → [role applied terraform to change RDS]
                                              → [terraform modified resource arn:...]
```

Each arrow is a separate audit event in a different system. Reconstructing the chain requires cross-system correlation by request ID.

**Integration pattern — propagate correlation ID:**

```yaml
jobs:
  deploy:
    env:
      TRACE_ID: ${{ github.run_id }}
    steps:
      - run: |
          aws s3 cp s3://bucket/config .
        env:
          AWS_REQUEST_HEADERS: "X-Trace-Id: ${{ env.TRACE_ID }}"
```

CloudTrail captures the trace ID; queries can now filter by `runId=456` and find all downstream actions.

**Log queries a good auditor will ask:**

1. "Show me every secret access in the last 30 days by workflow identity." → Vault audit log filtered by OIDC claims.

2. "Show me every production IAM role assumption by `ci-deploy` role outside of a scheduled deploy window." → CloudTrail + schedule cross-reference.

3. "Show me every user whose GitHub session originated from an unexpected geography." → GitHub audit log filtered by IP geolocation.

4. "For this production change, reconstruct the full chain: PR → approval → workflow run → IAM role → resource change." → cross-system correlation.

**Common gaps:**

- **No SIEM cross-correlation.** Each system has logs, but nobody queries across them. A determined attacker exploits gaps between systems.
- **OIDC identity not propagated.** Role assumption records the role, not which workflow. Mitigation: encode workflow metadata in the role session name.
- **Service account tokens with long TTLs.** A leaked service account token produces logs, but not tied to a specific human cause.
- **Direct production access via bastion.** Emergency break-glass sometimes bypasses the CI/CD audit trail. Must have separate break-glass logging.

**Practical setup:**

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123:role/ci-deploy
    role-session-name: github-${{ github.repository }}-${{ github.run_id }}
    # role-session-name shows up in CloudTrail — ties AWS actions to workflow
```

With this pattern, every CloudTrail event tagged `github-my-org/app-456` links back to workflow run 456, which links to PR #912, which links to Alice.

**Interview insight:** the "who accessed what" question in CI/CD is layered. Senior engineers articulate the layering (human → workflow → role → resource) and explain how correlation IDs thread them together.

### Q11. How do you handle "break-glass" procedures for emergency production access?

**Answer:**

**Break-glass** is the procedure for emergency access when normal controls would prevent fixing an urgent problem. It bypasses some standard controls but preserves audit trail, alerts stakeholders, and mandates post-hoc review.

**When it's used:**

- Production is down and the on-call engineer needs direct access to debug
- A critical security vulnerability needs immediate patching outside of review windows
- An approval system itself is broken and blocking the fix

**Design principles:**

1. **Breakable, not broken.** It should work when needed; ordinary operations shouldn't use it.
2. **Logged and visible.** Every use is recorded; multiple people are notified.
3. **Time-bound.** Access expires automatically.
4. **Post-hoc review.** Every use triggers a mandatory review within a defined SLA (e.g., 48 hours).
5. **Separate from normal paths.** Normal path deny does not block break-glass; break-glass deny does not block normal.

**Implementation patterns:**

**Pattern A — Break-glass IAM role with mandatory MFA:**

```hcl
resource "aws_iam_role" "break_glass_admin" {
  name = "break-glass-production-admin"
  assume_role_policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Principal = { AWS = "arn:aws:iam::123:role/oncall-engineer" }
      Action = "sts:AssumeRole"
      Condition = {
        Bool = { "aws:MultiFactorAuthPresent" = "true" }
        NumericLessThan = { "aws:MultiFactorAuthAge" = "900" }  # < 15 min
      }
    }]
  })
  max_session_duration = 3600   # 1 hour
}
```

Assumption requires fresh MFA; session expires within an hour.

**Pattern B — Break-glass workflow with multi-recipient alert:**

```yaml
# .github/workflows/break-glass-deploy.yml
on:
  workflow_dispatch:
    inputs:
      reason: { required: true, type: string }
      severity: { required: true, type: choice, options: [sev1, sev2] }

jobs:
  alert:
    runs-on: ubuntu-latest
    steps:
      - name: Announce break-glass in progress
        run: |
          curl -X POST $SLACK_WEBHOOK_SECURITY_CHANNEL \
            -d "{\"text\":\"BREAK-GLASS DEPLOY initiated by ${{ github.actor }}: ${{ inputs.reason }}\"}"

          # Also file incident ticket
          gh issue create --repo my-org/incident-response \
            --title "Break-glass: ${{ inputs.severity }} — ${{ inputs.reason }}" \
            --label break-glass \
            --assignee "@my-org/security-oncall"

  deploy:
    needs: alert
    runs-on: ubuntu-latest
    environment: production-emergency   # one approver, expedited
    steps:
      - run: emergency-deploy.sh ${{ inputs.reason }}

  schedule-review:
    needs: deploy
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: |
          gh issue create --title "[REVIEW DUE] Break-glass on $(date)" \
            --body "Review within 48 hours" --label post-hoc-review
```

**Pattern C — Temporary CODEOWNERS bypass:**

```yaml
# Normal: CODEOWNERS requires security-team review for /security/*
# Break-glass: a specific label "break-glass-urgent" (controlled by security team)
#              allows merge with abbreviated review

- if: contains(github.event.pull_request.labels.*.name, 'break-glass-urgent')
  run: |
    echo "Bypass applied. Review required within 48 hours."
    file_review_ticket
```

**Audit outputs:**

Every break-glass event produces:

- Immutable log entry (who, when, what, stated reason)
- Slack announcement (visible to multiple teams in real-time)
- Incident ticket (for tracking and post-hoc review)
- Scheduled review ticket (with SLA)

**Common abuses and guards:**

| Abuse | Guard |
|-------|-------|
| Break-glass used to skip normal approval routinely | Track usage; alert if > N/month per user |
| Post-hoc review skipped | Review SLA ticket escalates to VP after 48 hours |
| Break-glass credentials shared / reused | Session-bound MFA; no persistent tokens |
| Break-glass role has overly broad permissions | Scope to just emergency operations (e.g., rollback, restart) |

**Audit committee reviews:**

Quarterly or monthly, review all break-glass uses:

- Was it actually an emergency?
- Could it have been avoided with better normal paths?
- Were post-hoc reviews completed on time?
- Trend: is break-glass usage increasing? Why?

**Interview insight:** the sophistication of your break-glass design signals operational maturity. A weak answer: "admins bypass when needed." A strong answer: "MFA-bound, time-bound, auto-ticketed, Slack-broadcast, with 48-hour post-hoc review SLA and quarterly audit committee review." Specific mechanisms > general principles.

### Q12. How do you enforce "no single person can deploy to production unilaterally" in practice?

**Answer:**

This is separation of duties (Q3) applied to the deploy step. The technical implementation is straightforward; the completeness is the challenge.

**Layer 1 — Deploy approval gate:**

```yaml
deploy-prod:
  environment: production
  # Environment configured with:
  #   required_reviewers: team "prod-deployers"
  #   prevent_self_review: true
  steps: [...]
```

GitHub enforces: (a) at least one approver from the team; (b) the approver cannot be the workflow triggerer.

**Layer 2 — Code reviewer cannot also be deploy approver:**

This is harder to enforce natively. Approaches:

**Approach A — Separate teams:**

- `backend-reviewers` team: can approve PRs
- `prod-deployers` team: can approve production deploys
- Maintain zero overlap between teams

**Approach B — Custom validator job:**

```yaml
validate-separation:
  runs-on: ubuntu-latest
  steps:
    - name: Verify deployer != code author
      run: |
        pr_author=$(gh pr view ${{ github.event.pull_request.number }} --json author --jq .author.login)
        triggerer="${{ github.actor }}"
        if [ "$pr_author" = "$triggerer" ]; then
          echo "::error::PR author cannot trigger production deploy."
          exit 1
        fi
```

**Layer 3 — Deploy requires reference to an approved change ticket:**

```yaml
- name: Verify CHG ticket is approved by someone other than triggerer
  run: |
    chg="${{ inputs.change_ticket }}"
    approver=$(jira get $chg --field=customfield_change_approver.name)
    if [ "$approver" = "${{ github.actor }}" ]; then
      echo "::error::Change ticket must be approved by someone other than the deployer."
      exit 1
    fi
```

**Layer 4 — Cloud-side IAM enforcement (belt-and-braces):**

The CI deploy role requires an additional human approval before it can apply changes in certain scopes:

```json
// AWS IAM — requires MFA
{
  "Effect": "Allow",
  "Action": "rds:*",
  "Resource": "arn:aws:rds:eu-west-1:123:db:prod-*",
  "Condition": { "Bool": { "aws:MultiFactorAuthPresent": "true" } }
}
```

Or use AWS IAM Identity Center's session-based access with out-of-band approval.

**Common gaps:**

**Gap 1 — Admin override.**

An org admin can bypass environment protection rules. Restrict org admins to a small, accountable set; log admin actions.

**Gap 2 — Self-hosted runners.**

A self-hosted runner can bypass protections by executing directly. Sandbox self-hosted runners; don't give them deploy credentials without the environment gate.

**Gap 3 — Leaked approver token.**

An attacker with an approver's PAT can approve on their behalf. Mitigation: require approval via UI click (interactive session), not PAT.

**Gap 4 — Collusion.**

Two colluding engineers can satisfy all technical controls. SoD is about honest mistakes and single-party compromise, not against determined collusion. Detection via analytics: does the same pair always review each other's changes?

**Gap 5 — Emergency path.**

Break-glass (Q11) may allow single-person deploy. Reserve for true emergencies with post-hoc review.

**Evidence for audit:**

```sql
SELECT
  d.deployment_id, d.deployed_at, d.commit_sha,
  d.triggered_by, a.approver,
  (d.triggered_by = a.approver) AS self_approval
FROM deployments d
LEFT JOIN deployment_approvals a ON d.deployment_id = a.deployment_id
WHERE d.environment = 'production'
  AND d.deployed_at > NOW() - INTERVAL '90 days';
-- self_approval should always be FALSE
```

**Interview insight:** the question is a test of depth. Good answers mention at least three layers of enforcement (GitHub env, custom validator, cloud IAM, CHG ticket) plus explicit handling of break-glass and collusion. Superficial answers stop at "we require approval."

---

## Advanced

### Q13. Design a CI/CD pipeline that satisfies both SOC 2 and PCI-DSS requirements for a fintech service.

**Answer:**

SOC 2 and PCI-DSS have overlapping controls but different emphases. SOC 2 is principles-based; PCI-DSS is prescriptive (specific requirements per control). The pipeline design must satisfy both.

**Requirements overlap:**

| SOC 2 / PCI-DSS shared |
|-------------------------|
| Access control, least privilege, MFA |
| Change management with documented approvals |
| Audit trail with retention |
| Vulnerability scanning and patching |
| Secure coding practices |
| Secrets management |

**PCI-specific requirements:**

| PCI-DSS v4.0 control | Pipeline implication |
|----------------------|---------------------|
| 6.2 — Develop software securely | SAST in CI, security training evidence |
| 6.3 — Protect against known vulnerabilities | SCA with patch SLAs |
| 6.4 — Separate dev/test from production | Separate environments, separate credentials |
| 6.5 — Review changes before production | PR approvals, deploy approvals |
| 7.2 — Least-privilege access to CDE | Scoped IAM, no wildcard access |
| 10 — Logging and monitoring | Tamper-evident logs, time-sync, 1 year retention |
| 11.3 — Penetration testing | Annual test, evidence of remediations |
| 12.8 — Third-party management | Vetting of actions and dependencies |

**Pipeline architecture:**

**Source layer:**

```yaml
# Required protections on main
required_signatures: true            # SOC 2 CC8.1, PCI 10.5
required_pull_request_reviews:
  required_approving_review_count: 2 # PCI 6.5
  require_code_owner_reviews: true
required_status_checks:
  - lint
  - sast                             # PCI 6.2
  - dependency-scan                  # PCI 6.3
  - secret-scan                      # PCI 3.5
  - compliance-check
```

**Build layer — hermetic, signed, attested:**

```yaml
jobs:
  build:
    permissions: { id-token: write, contents: read, packages: write, attestations: write }
    runs-on: ubuntu-latest   # ephemeral runner
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      # PCI 6.3 — known-vulnerability scan pre-build
      - uses: aquasecurity/trivy-action@master
        with: { scan-type: fs, exit-code: 1, severity: CRITICAL,HIGH }

      - uses: docker/build-push-action@v5
        with:
          push: true
          provenance: true     # SLSA
          sbom: true           # PCI 6.3
          cache-from: type=registry,ref=ghcr.io/my-org/app:buildcache
          cache-to: type=registry,ref=ghcr.io/my-org/app:buildcache,mode=max

      # SLSA provenance + cosign signature
      - uses: actions/attest-build-provenance@v2
      - uses: sigstore/cosign-installer@v3
      - run: cosign sign --yes ghcr.io/my-org/app@${{ steps.push.outputs.digest }}
```

**Test layer — evidence-producing:**

```yaml
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: pytest --junit-xml=junit.xml --cov=src --cov-report=xml
      - uses: actions/upload-artifact@v4
        with:
          name: test-evidence
          path: |
            junit.xml
            coverage.xml
      # Retain test evidence for 1 year
      - run: aws s3 cp junit.xml s3://evidence/tests/${{ github.run_id }}.xml
```

**Deploy layer — approvals + change ticket:**

```yaml
  deploy-prod:
    needs: test
    runs-on: ubuntu-latest
    environment:
      name: production       # PCI 6.5: 2 approvers, prevent_self_review
    permissions: { id-token: write }
    steps:
      - name: Verify change ticket
        run: verify_chg_ticket.sh ${{ inputs.change_ticket }}

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123:role/ci-deploy-pci-cde
          role-session-name: github-deploy-${{ github.run_id }}

      # PCI 7.2 — least privilege; PCI 10 — logged
      - name: Verify image signature before deploy
        run: |
          cosign verify ghcr.io/my-org/app@${{ needs.build.outputs.digest }} \
            --certificate-identity-regexp '^https://github.com/my-org/app/.*' \
            --certificate-oidc-issuer 'https://token.actions.githubusercontent.com'

      - run: kubectl apply -f manifests/
      - run: record_deploy_to_audit_log.sh
```

**Evidence production (continuous):**

```yaml
# Weekly evidence generation job
name: Evidence generation
on:
  schedule: [{ cron: "0 6 * * 1" }]
jobs:
  evidence:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy evidence (SOC 2 CC8.1, PCI 6.5)
        run: produce_deploy_report.py > deploys-$(date -I).csv

      - name: Access review evidence (SOC 2 CC6.2, PCI 7.1)
        run: list_repo_collaborators.py > access-$(date -I).csv

      - name: Vulnerability scan evidence (PCI 6.3)
        run: trivy image --format cyclonedx ghcr.io/my-org/app:latest > scan-$(date -I).json

      - name: Push to WORM audit store
        run: aws s3 cp --recursive . s3://audit-evidence/ --sse AES256 \
               --object-lock-mode COMPLIANCE \
               --object-lock-retain-until-date $(date -d '+1 year' -I)
```

**PCI-specific additions:**

- **CDE (Cardholder Data Environment) segmentation.** Separate AWS account / Kubernetes cluster for PCI workloads; IAM roles and OIDC trust scoped to CDE-only.
- **Annual penetration test.** Schedule, remediate, evidence.
- **Quarterly vulnerability scans by an ASV (Approved Scanning Vendor).** Evidence stored with other audit artefacts.
- **No prod data in non-prod.** Synthetic test data only.

**SOC 2-specific additions:**

- **Vendor management.** Third-party action vetting, dependency risk review.
- **Incident response evidence.** One documented incident with runbook execution during the audit period.
- **Risk assessment.** Annual documented risk review of the CI/CD system.

**Cross-cutting:**

- All production changes flow through this pipeline; no side channels (no direct `kubectl apply` from laptops)
- Break-glass procedure with post-hoc review (Q11)
- Audit trail with 1-year retention (PCI minimum; SOC 2 may accept shorter)

**Interview insight:** for senior roles in regulated industries, expect this question. Naming specific PCI-DSS requirements by number (6.3, 7.2, 10.5) and mapping each to a pipeline feature is strong signal. Generic "we follow best practices" is weak.

### Q14. How do you structure a change advisory board (CAB) that actually functions in a high-velocity CI/CD environment?

**Answer:**

The traditional CAB — a weekly meeting of senior managers reviewing every production change — is incompatible with high-velocity engineering. Modern CAB approaches preserve the intent (oversight, risk assessment) while removing the friction.

**Traditional CAB problems:**

- Weekly meeting → blocks urgent changes until next meeting
- All attendees review all changes → nobody has context on all of them
- Approval is rubber-stamp or arbitrary pushback
- Engineers game the system (bundle changes to avoid extra meetings)

**Modern CAB patterns:**

**Pattern 1 — Asynchronous CAB by risk tier.**

Risk-classify changes (Q7). Only medium/high risk go through CAB at all.

- **Low risk:** auto-approved, no CAB
- **Medium risk:** async CAB review via a ticket; review SLA 1 business day
- **High risk:** sync review, architecture/security/SRE representatives

**Pattern 2 — Embedded review, not centralised.**

Instead of a central CAB, each team has a "change champion" who reviews their team's changes. The central CAB meets monthly to review trends, not individual changes.

**Pattern 3 — Pre-approved standard changes (Q4).**

Build a catalogue of pre-approved change types. The approval is up-front for the type, not per-instance:

```yaml
# standard-changes.yml
- type: "Dependabot patch update on approved library"
  conditions:
    - ecosystem: [pip, npm, github-actions]
    - semver_level: patch
    - all_tests_pass: true
  pre_approved: true

- type: "Configuration value change in non-prod"
  conditions:
    - path: "configs/dev/**"
  pre_approved: true
```

Changes matching pre-approved types skip CAB. Changes that don't fall into a defined type go through normal review.

**Pattern 4 — CAB as continuous monitoring, not gate.**

The CAB's role becomes reviewing *outcomes* rather than gating *proposals*:

- Weekly review: last week's deploys, incidents, rollbacks
- Monthly: trends — change volume, failure rate, MTTR, SoD violations
- Quarterly: structural issues — are certain teams consistently producing risky changes?

This preserves oversight without blocking throughput.

**Membership:**

- Engineering lead (speaks for the change's technical context)
- SRE or platform rep (speaks for operational readiness)
- Security rep (for security-adjacent changes)
- Product owner (for customer-impacting changes)

Not every change needs every member. Review is contextual.

**Pipeline integration:**

```yaml
jobs:
  classify:
    steps:
      - id: class
        run: ./classify-change.sh    # outputs: low | medium | high

  # Low risk: auto-merge, auto-deploy
  auto-merge:
    needs: classify
    if: needs.classify.outputs.risk == 'low'
    # merge and deploy without CAB

  # Medium risk: async CAB
  async-cab:
    needs: classify
    if: needs.classify.outputs.risk == 'medium'
    steps:
      - name: File async CAB ticket
        run: |
          gh issue create --repo my-org/cab --title "CAB: PR ${{ github.event.pull_request.number }}" \
            --body "Risk: medium. Review within 1 business day." \
            --label async-cab
      - name: Wait for CAB approval
        # Workflow paused until label 'cab-approved' is added
        run: poll_for_approval.sh

  # High risk: sync CAB, scheduled deploy window
  sync-cab:
    needs: classify
    if: needs.classify.outputs.risk == 'high'
    steps:
      - run: schedule_sync_review.sh
```

**Metrics to track:**

- Time from PR open to CAB approval (by risk tier)
- % of changes auto-approved (too low = friction; too high = no oversight)
- Change-failure rate (deploys that triggered rollback) — higher-risk tiers should have lower rates
- Emergency changes per month (high = process breakdown somewhere)

**Common anti-patterns:**

- **Every change to CAB.** Choking point, no oversight because review is superficial.
- **No CAB.** No systemic view; security/SRE find out about risks post-incident.
- **CAB that reviews but can't block.** Theatre.
- **CAB that doesn't share its decisions.** Team morale suffers; nobody learns.

**Interview insight:** for enterprise-oriented senior roles, the CAB question probes whether you can bridge "compliance wants oversight" with "engineering wants velocity." The answer is "risk-based, async by default, sync only when necessary, monitor outcomes not gate proposals." That's the modern synthesis.

### Q15. A new regulation (EU Cyber Resilience Act, 2027 enforcement) requires signed releases and SBOMs for any software with digital elements. Walk through the pipeline changes required.

**Answer:**

The EU Cyber Resilience Act (CRA) has specific requirements for manufacturers of products with digital elements. The operational translation for CI/CD:

**Relevant CRA obligations:**

- Provide an SBOM for every release
- Handle and report security vulnerabilities
- Sign releases cryptographically
- Conformity assessment before sale
- 5-year post-launch support obligations

**Pipeline changes by obligation:**

**1. Signed releases (CRA Annex I, s2).**

Already covered by Sigstore + cosign (`supply_chain_security.md` Q5). Action: ensure every release artefact is signed. Gate deploys on signature verification.

```yaml
- uses: sigstore/cosign-installer@v3
- run: cosign sign --yes ghcr.io/my-org/app@${{ steps.build.outputs.digest }}
```

**2. SBOMs for every release (CRA Annex I, s2).**

Already covered (`supply_chain_security.md` Q2, `artifact_management.md` Q11). CRA mandates SBOMs be available to authorities on request — storage and retrieval matter.

```yaml
- uses: anchore/sbom-action@v0
  with: { format: cyclonedx-json, output-file: sbom.cdx.json }

- uses: actions/attest-sbom@v2
  with:
    subject-name: ghcr.io/my-org/app
    subject-digest: ${{ steps.build.outputs.digest }}
    sbom-path: sbom.cdx.json
```

**3. Vulnerability handling and reporting (CRA Art 13).**

CRA requires notifying ENISA within 24 hours of becoming aware of an actively exploited vulnerability. The pipeline implication:

- **Continuous vulnerability scanning** of deployed versions (not just at build time)
- **Automated alerting** when a CVE affects a deployed product
- **Workflow for reporting** to authorities within the time window

```yaml
# Daily scan of production SBOMs
on:
  schedule: [{ cron: "0 2 * * *" }]
jobs:
  scan:
    steps:
      - name: Scan current production SBOM
        run: grype sbom:s3://sboms/prod-current.json --fail-on high --output json > scan.json
      - name: Alert security team on findings
        if: failure()
        run: |
          python scripts/notify_and_evaluate_reportability.py scan.json
          # If exploitable and CRA-reportable, start the 24-hour clock
```

**4. Long-term support (CRA Art 10, 5-year minimum).**

Must be able to build and ship security patches for products sold up to 5 years ago. Implications:

- Long-term retention of source, build environment, and artefacts
- Build reproducibility (can we build the same thing 5 years later?)
- Documented build environment (container images archived)

```yaml
# At release time, archive the build environment
- name: Archive build environment
  run: |
    docker save ghcr.io/my-org/buildenv:v2.4.1 | gzip > buildenv-v2.4.1.tar.gz
    aws s3 cp buildenv-v2.4.1.tar.gz s3://longterm-build-archive/ \
      --storage-class GLACIER
```

**5. Conformity assessment (CRA Art 24).**

For high-criticality products, independent assessment before placing on the market. The pipeline doesn't do the assessment, but it produces the evidence:

- Complete SBOM
- Vulnerability scan reports
- Security test results
- Signed provenance
- Build reproducibility evidence
- Documentation of security development practices

```yaml
# On release tag, package conformity evidence
on:
  push: { tags: ["v*.*.*"] }
jobs:
  conformity-package:
    steps:
      - run: |
          mkdir evidence/
          cp sbom.cdx.json evidence/
          cp scan.json evidence/
          cp junit.xml evidence/
          cp provenance.json evidence/
          tar cf conformity-evidence-${GITHUB_REF_NAME}.tar evidence/
          aws s3 cp conformity-evidence-${GITHUB_REF_NAME}.tar s3://cra-evidence/ \
            --object-lock-mode COMPLIANCE \
            --object-lock-retain-until-date $(date -d '+10 years' -I)
```

**6. Machine-readable technical documentation (CRA Annex V).**

CRA requires product documentation in a format authorities can automate against. Pipeline generates:

- Machine-readable product specification (JSON / XML)
- Cryptographic hash of the product with signature
- Reference to SBOM

**Rollout plan (from today to 2027):**

| Year | Milestone |
|------|-----------|
| 2025 | Inventory all products; confirm SBOM generation on every release |
| 2025-2026 | Implement continuous vulnerability monitoring of deployed SBOMs |
| 2026 | Establish ENISA reporting workflow; tabletop-test the 24-hour timeline |
| 2026 | Build reproducibility baseline; long-term archive policies |
| 2026-2027 | Conformity assessment preparation; documentation pipelines |
| 2027 | Full enforcement; audit readiness |

**What doesn't change:**

Most of what CRA demands is what modern supply chain security already provides — cosign signing, SBOM generation, SLSA provenance. The gap for most orgs is (a) long-term retention and (b) continuous vulnerability monitoring vs one-time scan-at-build.

**Interview insight:** for 2025-2027, CRA readiness is a real topic in EU-facing companies. Naming CRA articles, discussing the 24-hour ENISA reporting window, and articulating the rollout plan signals current engagement with the regulatory landscape.

### Q16. How do you balance logging detail for audit purposes against the risk of leaking sensitive data in logs?

**Answer:**

More logging produces more evidence for audit but more risk of leakage. The balance requires per-field classification and structural logging.

**Data classification:**

| Class | Examples | Logging treatment |
|-------|----------|-------------------|
| **Public** | Service name, region, version | Full fidelity |
| **Internal** | Workflow run ID, commit SHA, deployment environment | Full fidelity |
| **Sensitive** | User IDs (when tied to PII), internal IPs | Logged with retention limits |
| **Secret / PII** | Passwords, tokens, credit card numbers, email addresses | Never logged; redacted or tokenised |

**Structured logging with field classification:**

```python
import structlog

log = structlog.get_logger()
log.info(
    "deploy_started",
    environment="production",       # internal — log
    actor=github_actor,              # internal — log
    commit=github_sha,               # internal — log
    change_ticket=chg_id,            # internal — log
    customer_id=mask(customer_id),   # sensitive — hash/mask
    # Never log: api_key, db_password, customer.email
)
```

Structured logs can be filtered at ingest — drop fields matching regex patterns known to contain secrets.

**Redaction at multiple layers:**

**Layer 1 — Application code:**

Never put secrets in log format strings. Use dedicated methods that accept known-safe fields.

**Layer 2 — Log aggregation:**

```yaml
# Vector / Logstash config
transforms:
  redact_secrets:
    type: remap
    source: |
      .api_key = redact(.api_key)
      .password = redact(.password)
      # Regex-based catch-all for high-entropy strings
      .message = redact_secrets(.message, patterns: ["AKIA[A-Z0-9]{16}", "ghp_[A-Za-z0-9]{36}"])
```

**Layer 3 — CI provider native masking:**

GitHub Actions masks values registered as secrets in logs. Limitations:

- Masking is exact-string. Base64-encoding or transforming defeats it.
- Secrets not registered (custom internal tokens) aren't masked.

**Layer 4 — Storage-level protection:**

- Encryption at rest for log storage
- RBAC on log queries (not everyone with "log reader" role can see everything)
- Field-level access control (SOC analysts see redacted; incident responders see full)

**Case study — accidentally logging a token:**

```python
# BAD: logs the full token
log.info(f"Calling API with token: {api_token}")

# BAD: logs via request kwargs that include auth header
log.info(f"Request: {request.__dict__}")

# BETTER:
log.info("calling_api", token_id=api_token[:4] + "..." + api_token[-4:])

# BEST: don't log auth material at all
log.info("calling_api", endpoint="/v1/users")
```

**Secrets-in-logs detection (as a detective control):**

```yaml
# Continuous scan of log storage for leaked secrets
on:
  schedule: [{ cron: "0 */6 * * *" }]
jobs:
  scan:
    steps:
      - run: |
          aws s3 ls s3://logs/ --recursive | xargs -I{} aws s3 cp {} - | \
            trufflehog --only-verified --json | \
            jq -r '.SourceMetadata.Data.Filesystem.file'
```

Finds secrets that slipped past other controls. Rotate + investigate source.

**What to log for audit:**

The following must be logged with high fidelity (SOC 2/PCI retention: 1+ years):

| Event | Why |
|-------|-----|
| Login, logout | Access control evidence |
| Role assumption | Privilege escalation tracking |
| Data access | Who touched what (PCI) |
| Admin actions (user add, permission change) | Change-to-privilege evidence |
| Production changes (deploy, migration) | Change management evidence |
| Secret access (read by workflow) | Key usage audit |
| Break-glass usage | Emergency-access audit |

**What NOT to log:**

- Request/response bodies that might contain PII (log metadata only)
- Database query results (unless redacted)
- Environment variables dumped as-is (will include secrets)
- Stack traces for auth flows (may leak token in local variables)

**Retention differences:**

Operational logs (debug, info, warn): 30-90 days.
Audit logs: 1-7 years per regulation.
PII-containing logs: minimised in duration (GDPR principle of data minimisation).

**Interview insight:** the question tests whether you can balance two opposing demands. Surface-level answers say "we log everything"; senior answers articulate the classification schema, the layered redaction, and the difference between operational and audit retention. Naming specific protections (structured logging, field-level RBAC, storage encryption, periodic secret scanning) makes it concrete.

### Q17. Your CI/CD system must pass an ISO 27001 audit. Walk through the evidence package you'd prepare.

**Answer:**

ISO 27001 is a principles-based information security standard. Unlike SOC 2's attestation model, ISO 27001 is a certification — independent auditors verify a formal ISMS (Information Security Management System) exists and functions. Annex A lists 93 controls (in the 2022 revision).

**Relevant Annex A controls for CI/CD:**

| Control | What it means |
|---------|---------------|
| A.5.15 Access control | Who has access to CI/CD, how is it granted |
| A.5.20 Supplier agreements | Third-party tools (GitHub, registries, SaaS) have DPAs |
| A.5.23 Cloud services | Cloud CI/CD has appropriate agreements |
| A.6.7 Remote working | Engineers work from anywhere; laptops and credentials policy |
| A.8.2 Privileged access rights | Admin / deploy roles separate, reviewed |
| A.8.3 Information access restriction | Least privilege in IAM and secrets |
| A.8.4 Access to source code | Repo permissions, reviews |
| A.8.9 Configuration management | Infrastructure and pipeline configs version controlled |
| A.8.15 Logging | Audit logs retained, reviewed |
| A.8.16 Monitoring activities | Anomaly detection on CI/CD |
| A.8.25 Secure development lifecycle | SDLC with security built in |
| A.8.28 Secure coding | Standards, training, SAST |
| A.8.29 Security testing | Vulnerability scans, pen tests |
| A.8.30 Outsourced development | Third-party code reviewed |
| A.8.31 Separation of dev/test/prod | Environment isolation |
| A.8.32 Change management | Changes reviewed, tested, approved |

**Evidence package structure:**

**Section 1 — ISMS scope and policy (high-level documents):**

- ISMS scope statement (what's covered — "all production CI/CD systems for product X")
- Information security policy (board-approved)
- Risk assessment methodology
- Risk treatment plan

**Section 2 — Asset inventory:**

- CI/CD system inventory (GitHub org, cloud accounts, registries, SIEM)
- Data classification (what data flows through CI/CD, of what sensitivity)
- Third-party service inventory with DPA status

**Section 3 — Access control evidence:**

- List of users with admin access (org-wide, per-system)
- Quarterly access review records — evidence of review, approver, date
- Leavers process evidence — removal log for past year
- MFA enforcement evidence — percentage coverage, exceptions documented

**Section 4 — Change management evidence:**

- Documented change management procedure
- Sample of 30 production deploys from the audit period:
  - PR record (reviewers, approvals)
  - CI run record (tests passed)
  - Deploy approval record
  - Change ticket
  - Outcome (success / rollback)
- Emergency change records (break-glass usage, post-hoc reviews)

**Section 5 — Logging and monitoring:**

- Audit log retention configuration (demonstrate 1+ year retention)
- Sample audit log queries showing coverage
- Monitoring rule definitions
- Alert response evidence (how many alerts, how many responded to, MTTR)

**Section 6 — Vulnerability management:**

- Dependency scanning configuration
- Vulnerability SLA (e.g., Critical: 7 days, High: 30 days, Medium: 90 days)
- Evidence of SLA adherence (sample of CVEs with fix dates)
- Annual penetration test report
- Remediation evidence for pen test findings

**Section 7 — Secure development:**

- SDLC documentation
- Coding standards / secure coding guide
- Training records (who completed security training, when)
- SAST configuration + sample output
- Code review practices (CODEOWNERS, required reviewers)

**Section 8 — Supply chain:**

- Third-party action allow-list
- Dependency vetting process
- SBOM samples for releases
- Provenance generation evidence (SLSA)
- Signed commit enforcement evidence

**Section 9 — Incident response:**

- Incident response plan
- Tabletop exercise records (dates, scenarios, lessons learned)
- Incident records from audit period (anonymised if needed)
- Mean time to detect, respond, resolve — by severity

**Section 10 — Separation of environments:**

- Architecture diagram showing dev/test/prod separation
- IAM evidence showing credential scope
- Network / account isolation evidence

**Section 11 — Supplier management:**

- DPAs with GitHub, cloud providers, SaaS tools
- Supplier security review records
- SOC 2 reports from suppliers

**Audit timeline:**

- **Month -6:** Gap analysis against Annex A; remediate gaps
- **Month -3:** Internal audit (use the certification body's checklist)
- **Month -1:** Evidence collection; freeze pipeline config
- **Month 0:** External audit (Stage 1: documentation review; Stage 2: operational review)
- **Ongoing:** Surveillance audits annually; full recertification every 3 years

**Common findings (things to prepare extra evidence for):**

- "You have branch protection — show me every override in the past year." Evidence: GitHub audit log filtered for `protected_branch.policy_override` events.
- "You rotate secrets — show me." Evidence: per-secret rotation log with timestamps.
- "You do access reviews — show me the last one for team X." Evidence: spreadsheet with reviewer signature, dated.
- "Your CI runs SAST — show me the configuration and a failed run response." Evidence: config file in Git, Jira ticket for fix.
- "How do you know your logs are complete?" Evidence: ingest monitoring, periodic end-to-end test events.

**Interview insight:** ISO 27001 is common in European and multinational companies. The evidence package is extensive; articulating it by section demonstrates you understand the scope. Name specific Annex A controls (A.8.32 change management, A.5.15 access control) to signal familiarity.

### Q18. Design a "compliance as code" framework for a CI/CD platform team serving 500 engineers. How do you keep compliance enforcement from becoming a bottleneck?

**Answer:**

The goal: compliance controls are defined as code, automatically enforced at pipeline time, and reviewed as part of normal code review — not as a parallel bureaucratic process.

**Architecture:**

```
[Policies in Git (policy-as-code)] --loaded by--> [Policy engine (OPA, Kyverno, Sentinel)]
                                                         |
                                                         v
                                 [Evaluated at: PR time, build time, deploy time, runtime]
                                                         |
                                                         v
                                 [Violations become PR comments, build failures, alerts]
```

**Layer 1 — Pipeline policies (OPA / Conftest).**

Check that each team's workflow files meet org-wide standards:

```rego
# policies/pipeline.rego
package pipeline

# Actions must be pinned to a SHA, not a tag
deny[msg] {
    step := input.jobs[_].steps[_]
    step.uses
    not regex.match(`@[a-f0-9]{40}$`, step.uses)
    msg := sprintf("Action '%s' must be pinned to a SHA", [step.uses])
}

# Production deploys require environment with reviewers
deny[msg] {
    job := input.jobs[_]
    contains_lower(job.name, "deploy")
    contains_lower(job.environment, "prod")
    not job.environment.name
    msg := sprintf("Production deploy job '%s' must use a GitHub Environment", [job.name])
}

# No write-all permissions
deny[msg] {
    input.permissions == "write-all"
    msg := "Workflow permissions must not be 'write-all'"
}
```

Enforced at PR time:

```yaml
- uses: open-policy-agent/conftest@v0.52.0
- run: conftest test .github/workflows/*.yml --policy policies/
```

**Layer 2 — Infrastructure policies (Terraform + Sentinel / OPA):**

```rego
# policies/iam.rego
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_iam_policy"
    statement := resource.change.after.policy_json.Statement[_]
    statement.Action == "*"
    statement.Resource == "*"
    msg := "IAM policy must not grant '*:*'"
}
```

Runs on `terraform plan` output in CI.

**Layer 3 — Runtime admission policies (Kyverno / Gatekeeper):**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: require-signed-images }
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-images
      match: { resources: { kinds: [Pod] } }
      verifyImages:
        - imageReferences: ["ghcr.io/my-org/*"]
          attestors:
            - entries:
              - keyless:
                  subject: "https://github.com/my-org/*/.github/workflows/*"
                  issuer: "https://token.actions.githubusercontent.com"
```

Enforces "only signed images may run" at Kubernetes admission time.

**Layer 4 — Audit policies (scheduled scans):**

```yaml
on:
  schedule: [{ cron: "0 3 * * *" }]
jobs:
  compliance-scan:
    steps:
      - name: Check all repos have signed-commits required
        run: |
          for repo in $(gh api /orgs/my-org/repos --paginate --jq '.[].name'); do
            config=$(gh api /repos/my-org/$repo/branches/main/protection --jq .required_signatures.enabled)
            if [ "$config" != "true" ]; then
              echo "Repo $repo: signed commits NOT required"
            fi
          done
      - name: Generate compliance report
        run: ./generate-report.sh
```

**Preventing the platform team becoming a bottleneck:**

**1. Policies are PRs, not tickets.**

A team that needs an exception proposes a PR to the policy repo. The platform team reviews like any PR. Exceptions are documented and time-bound.

```rego
# policies/exceptions.rego
exemptions = {
    "my-org/legacy-app": {
        "reason": "Cannot pin actions; migrating off Jenkins",
        "expires": "2025-09-30",
        "policies": ["pipeline.pin-actions"],
    },
}
```

**2. Self-service tooling for the 80%.**

```bash
# Developers run locally to validate
$ make compliance-check
Checking workflows...     ✓
Checking terraform...      ✓
Checking admission rules... ✓
```

Most teams pass without platform involvement.

**3. Graduated enforcement.**

New policies roll out in three phases:

- **Week 1-2: Advisory.** Violations produce a PR comment; don't block.
- **Week 3-4: Warning.** CI job warns; doesn't block.
- **Week 5+: Blocking.** CI job fails; merge blocked.

Teams have time to fix; platform team has time to help.

**4. Policy deprecation.**

When a policy is obsolete, remove it. Don't accumulate 400 policies half of which nobody remembers the rationale for. Annually review and cull.

**5. Metrics.**

- % of repos passing all policies (target: 98%+)
- Time from policy introduction to 90% adoption
- Exception count (high = over-strict policies)
- Developer satisfaction survey ("compliance as code helps / blocks me")

**6. Centralised dashboard.**

Each team sees their compliance posture at a glance:

```
Team: backend-platform
Repos: 12                              Coverage: 11/12 (92%)
Violations: 3                          Trend: improving
Action: fix signed-commits on repo legacy-auth (see exception)
```

**7. Onboarding tooling.**

New repos are created from a template that already passes all policies. New services are compliant from day one.

**Anti-patterns to avoid:**

| Anti-pattern | Symptom |
|--------------|---------|
| Platform team writes policies, teams don't know | High exception rate, resentment |
| Blocking enforcement from day one | Outage of normal work, rollback |
| No exception process | Teams work around the system |
| Policy repo not version-controlled | Nobody knows why a policy exists |
| No deprecation | Stale policies conflict with modern practice |

**Team structure:**

For a 500-engineer org, a compliance-as-code platform team is typically 3-5 engineers: 2 developing policies + tooling, 1-2 running the compliance dashboard + exception process, 1 lead connecting with security / audit / legal.

**Interview insight:** this is a staff/principal-level system design question. The strong answer combines (a) technical architecture (OPA, Kyverno, Terraform policies), (b) rollout mechanics (advisory → warning → blocking), and (c) organisational design (exception PRs, team structure, metrics). The weak answer lists tools without the governance layer — or describes governance without enforcement code. The synthesis is the test.
