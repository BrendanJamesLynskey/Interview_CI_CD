# Artifact Management — Interview Questions

**Subject:** CI/CD
**Topic:** Artifact Repositories (Artifactory, Nexus), Container Registries (ECR, GHCR), Dependency Caching, Provenance, SLSA
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is a build artefact, and why does it need a dedicated repository?

**Answer:**

A **build artefact** is the output of a CI build: a compiled binary, a Python wheel, a Java JAR, a Docker image, a Helm chart, a static site bundle. Anything downstream stages or other systems consume from a build belongs in artefact storage.

**Why a dedicated repository (and not Git, S3, or scp):**

1. **Immutable versioning.** Once published, an artefact at a specific version never changes. Repos enforce this.
2. **Metadata.** Repos store provenance, hashes, signatures, build info, license data alongside the binary.
3. **Promotion.** A repo supports moving an artefact from "staging" to "release" without rebuilding.
4. **Retention.** Repos handle eviction policies (keep last N, keep all releases, delete after T days).
5. **Access control.** Per-repo, per-team, read/write/admin separation.
6. **Auditability.** Who pushed what, who pulled what — required for SOC 2 / SLSA.

**Tooling landscape:**

| Type | Tools |
|------|-------|
| Generic / multi-format | JFrog Artifactory, Sonatype Nexus, GitHub Packages |
| Container images | Docker Hub, ECR, GCR, GHCR, ACR, Harbor (self-hosted) |
| Language-specific | PyPI, npm, Maven Central, RubyGems, NuGet |
| Helm charts | OCI registries (Helm 3+ uses OCI), ChartMuseum (legacy) |

**Interview insight:** the wrong place to store artefacts is "in the same repo as the code" (bloats Git) or "as a workflow artefact" (CI provider artefacts have short retention and aren't designed for production consumption).

### Q2. What is the difference between Artifactory and Nexus, and when would you choose each?

**Answer:**

Both are universal artefact repositories supporting most package formats. The differences are commercial and operational.

| Aspect | JFrog Artifactory | Sonatype Nexus |
|--------|-------------------|----------------|
| Open source edition | Yes (OSS, limited features) | Yes (Nexus OSS, full features) |
| Enterprise edition | Artifactory Pro / Enterprise | Nexus Repository Pro |
| Supported formats | 30+ (npm, PyPI, Maven, Docker, Helm, Conan, Cargo, Go, ...) | 20+ (similar coverage) |
| HA / replication | Native (Pro+) | Native (Pro) |
| Storage backends | Filesystem, S3, GCS, Azure Blob | Filesystem, S3, Azure Blob |
| UI / search | Polished | Functional |
| Companion tools | Xray (vuln scanning), Distribution | IQ Server (vuln scanning) |

**When to choose Artifactory:**

- You need wide format support out of the box (Conan, Cargo, Helm, Go modules)
- Your org already uses other JFrog products (Xray, Pipelines, Distribution)
- You want polished UX with deep build-info integration

**When to choose Nexus:**

- Java-heavy environment (Sonatype = the Maven Central people)
- Cost-sensitive, the OSS edition covers many use cases
- You want IQ Server's policy engine for licence/vulnerability gating

**When to choose neither:**

- Cloud-native shop with one cloud — use ECR/GCR/ACR for containers, native package services (CodeArtifact, Artifact Registry) for language packages
- Open-source project — use GitHub Packages or the language-specific public registries

### Q3. What is a container registry, and what's the difference between Docker Hub, ECR, and GHCR?

**Answer:**

A **container registry** stores OCI/Docker images and serves them via the Docker Registry HTTP API. All three are container registries; the differences are hosting, pricing, and integration.

| Aspect | Docker Hub | AWS ECR | GitHub GHCR |
|--------|-----------|---------|-------------|
| Hosted by | Docker, Inc. | AWS | GitHub |
| Visibility | Public + private | Private (or public via ECR Public) | Public + private |
| Auth | Docker login / token | AWS IAM (OIDC supported) | GitHub PAT or `GITHUB_TOKEN` |
| Free tier | 1 private repo | 500 MB / month / public ECR | Unlimited public |
| Rate limits | 100/200 pulls per 6h (anonymous/authenticated) | None within AWS | Generous |
| Integration | Universal | Native AWS (IAM, ECS, EKS) | Native GitHub Actions |
| Image scanning | Docker Scout | Inspector / native scan | Trivy via integration |

**Push to GHCR from GitHub Actions:**

```yaml
permissions:
  contents: read
  packages: write

jobs:
  build-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

**Push to ECR with OIDC:**

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123:role/ci-ecr-push
    aws-region: eu-west-1
- uses: aws-actions/amazon-ecr-login@v2
- run: |
    docker build -t 123.dkr.ecr.eu-west-1.amazonaws.com/app:${{ github.sha }} .
    docker push 123.dkr.ecr.eu-west-1.amazonaws.com/app:${{ github.sha }}
```

**Choice criteria:**

- **GHCR** for OSS projects and GitHub-centric workflows
- **ECR** if you deploy to ECS/EKS — IAM auth and zero egress beats anything else
- **Docker Hub** for public distribution where consumers expect to find you there
- **Self-hosted Harbor / Artifactory** for regulated environments needing on-prem

### Q4. What is dependency caching, and what layers should a pipeline cache?

**Answer:**

**Dependency caching** stores downloaded packages and intermediate build outputs across CI runs to avoid repeating expensive work. A pipeline that takes 10 minutes uncached often takes 1-2 minutes warm.

**Layered caching model (fastest to slowest):**

| Layer | Lifetime | Examples |
|-------|----------|----------|
| **L0 — Runner local** | One job | `node_modules` between steps in the same job |
| **L1 — CI provider cache** | Days, per branch | GitHub Actions cache, GitLab cache |
| **L2 — Internal mirror / proxy** | Weeks | Artifactory remote repos, npm-mirror, devpi |
| **L3 — Public registry** | Forever | npmjs.com, pypi.org, Docker Hub |

**Per-language cache targets:**

```yaml
# Python
- uses: actions/cache@v4
  with:
    path: |
      ~/.cache/pip
      ~/.cache/uv
    key: ${{ runner.os }}-py-${{ hashFiles('**/requirements*.txt', '**/poetry.lock', '**/uv.lock') }}

# Node.js
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}

# Go
- uses: actions/cache@v4
  with:
    path: |
      ~/.cache/go-build
      ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}

# Maven
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
```

**Build output caching (the higher-value layer):**

Tools like Bazel, sccache, ccache, Turborepo, and Gradle Build Cache key outputs by content hash of inputs. If nothing changed, fetch from cache:

```bash
bazel build //... --remote_cache=grpcs://cache.example.com
sccache --start-server && CC="sccache cc" make
```

**Pitfall — cache poisoning:**

Caches can store bad state (wrong arch, corrupted file). Always include `runner.os`, language version, and architecture in the key. Provide a manual "bust cache" mechanism (rotate a salt in a config file).

### Q5. What is artefact promotion, and why is it preferred over rebuilding?

**Answer:**

**Promotion** is moving an existing artefact through environments (dev → staging → production) by changing labels or copying to a new repo path, *without* rebuilding from source.

**Why promote instead of rebuild:**

1. **Build determinism is hard.** A second build often produces a slightly different binary (timestamps, build paths, embedded versions). Promoting the exact tested artefact means production runs what staging tested.
2. **Faster releases.** No rebuild = no compile + test cycle.
3. **Audit clarity.** "This SHA-256 hash was tested on date X and deployed on date Y" is one entity through its lifecycle.
4. **Supply chain integrity.** A promoted artefact retains its signature and provenance. A rebuild produces a new artefact requiring re-signing and re-attestation.

**Promotion patterns:**

**Pattern A — Repo-per-environment:**

```
artifactory/builds-dev/myapp/2.4.1
artifactory/builds-staging/myapp/2.4.1
artifactory/builds-prod/myapp/2.4.1
```

```bash
# Copy on promotion
jfrog rt copy "builds-dev/myapp/2.4.1" "builds-staging/myapp/" --flat=false
```

**Pattern B — Tag-based promotion:**

Single repo; container/manifest tags indicate state:

```bash
docker tag registry/app:2.4.1-rc1 registry/app:2.4.1-staging
docker push registry/app:2.4.1-staging
```

**Pattern C — Properties / labels (Artifactory):**

Same artefact, metadata changes:

```bash
jfrog rt sp "builds/myapp/2.4.1/*" "deployment-status=production;promoted-by=alice"
```

**Anti-pattern: rebuild-per-environment.**

```yaml
# Don't do this
deploy-to-staging:
  steps: [build, test, deploy]

deploy-to-prod:
  steps: [build, test, deploy]    # different binary than staging tested!
```

The "build once, promote many" mantra is the foundation of trustworthy CD.

### Q6. What is provenance in the context of CI/CD, and why does it matter?

**Answer:**

**Provenance** is verifiable metadata describing how an artefact was built: source repo, source commit, builder identity, build inputs, build steps, and timestamps. It answers the question "where did this binary come from?"

**Why it matters:**

After incidents like SolarWinds (2020), Codecov (2021), and `xz-utils` (2024), regulators and customers want cryptographic evidence that the binary they run came from the source they reviewed. Provenance is that evidence.

**SLSA Provenance specification (in-toto attestation):**

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [
    {
      "name": "myapp",
      "digest": { "sha256": "a1b2c3..." }
    }
  ],
  "predicateType": "https://slsa.dev/provenance/v1",
  "predicate": {
    "buildDefinition": {
      "buildType": "https://github.com/actions/runner",
      "externalParameters": {
        "workflow": ".github/workflows/release.yml",
        "ref": "refs/tags/v2.4.1",
        "repository": "github.com/my-org/myapp"
      }
    },
    "runDetails": {
      "builder": { "id": "https://github.com/actions/runner@v2.317.0" },
      "metadata": {
        "invocationId": "https://github.com/my-org/myapp/actions/runs/123",
        "startedOn": "2025-04-15T10:00:00Z"
      }
    }
  }
}
```

**Generating provenance in GitHub Actions:**

```yaml
permissions:
  id-token: write
  contents: read
  attestations: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make build
      - uses: actions/attest-build-provenance@v2
        with:
          subject-path: dist/myapp
```

**Verification by consumers:**

```bash
gh attestation verify dist/myapp --repo my-org/myapp
```

This proves the binary was built by the named GitHub Actions workflow on the named commit, signed by the GitHub OIDC identity, and stored in Sigstore's transparency log.

---

## Intermediate

### Q7. How do you set up Artifactory as a proxy for public package registries, and what benefits does that provide?

**Answer:**

**Configuration:** create a "remote repository" in Artifactory pointing at the public registry. Builds connect to Artifactory; Artifactory caches misses.

```
artifactory/api/pypi/pypi-remote/simple/  → proxies pypi.org
artifactory/api/npm/npm-remote/           → proxies registry.npmjs.org
artifactory/api/docker/dockerhub-remote/  → proxies docker.io
```

**Pip configuration:**

```ini
# /etc/pip.conf
[global]
index-url = https://artifactory.example.com/api/pypi/pypi-remote/simple/
```

**Benefits:**

1. **Availability.** Outage at npmjs.org doesn't break your builds — Artifactory serves cached packages.
2. **Speed.** First fetch from public registry; subsequent fetches from local Artifactory at LAN speed.
3. **Audit.** All third-party packages your org consumes are visible in one place.
4. **Vulnerability scanning.** Artifactory + Xray scans cached packages and blocks downloads of known-bad versions.
5. **Supply chain control.** You can quarantine compromised packages (e.g., delete `event-stream@3.3.6` from your cache after the 2018 incident).
6. **Cost control.** No egress charges for repeated pulls of the same dependency.
7. **Air-gap readiness.** If you ever need to operate disconnected, you've already cached everything you need.

**Virtual repository pattern:**

Combine multiple remotes plus your own internal repos behind a single URL:

```
pypi-virtual:
  ├── pypi-internal      (your own packages)
  ├── pypi-staging       (release candidates)
  └── pypi-remote        (proxy of pypi.org)
```

Resolution order is configurable. Internal packages shadow public ones — defence against dependency confusion attacks.

### Q8. What is a dependency confusion attack, and how do you prevent it?

**Answer:**

**Dependency confusion** (Alex Birsan, 2021) exploits the fact that many package managers prefer the highest-version package across all configured registries. If your private package `internal-tools` is at v1.4 in your internal registry, and an attacker publishes `internal-tools@99.0.0` to the public PyPI/npm, builds may pull the malicious public version.

**Affected ecosystems:** npm, PyPI, RubyGems, NuGet, Maven (less directly), Cargo (less directly).

**Prevention strategies:**

**1. Scoped namespaces (npm):**

```json
// .npmrc
@my-org:registry=https://artifactory.example.com/api/npm/npm-internal/
registry=https://registry.npmjs.org/
```

All `@my-org/*` packages resolve to the internal registry only. Public can't impersonate.

**2. Index URL precedence (pip):**

```ini
[global]
index-url = https://artifactory.example.com/api/pypi/pypi-virtual/simple/
# No --extra-index-url — the virtual repo handles fallback
```

Avoid `--extra-index-url`; pip queries all index URLs in parallel and picks the highest version.

**3. Reserve the package name publicly:**

Even if you never publish to PyPI/npm, register the name with a "do not use" placeholder. Attackers can't squat on it.

**4. Pin exact versions and verify hashes:**

```
# requirements.txt with hashes (pip-compile --generate-hashes)
internal-tools==1.4.0 \
    --hash=sha256:a1b2c3d4...
```

Even if a malicious package has a higher version, hash verification fails.

**5. Use a virtual / proxy repository as the only configured registry:**

Build configs should only know about your internal Artifactory virtual repo. The proxy handles fetching from public registries with internal-package precedence.

**Audit step in CI:**

```yaml
- name: Detect external resolution
  run: |
    pip install --dry-run --report install-report.json -r requirements.txt
    jq '.install[].metadata | select(.name | startswith("my-org-"))' install-report.json | \
      jq -e '.download_info.url | startswith("https://artifactory.example.com")'
```

Fail the build if any internal package was resolved from outside the internal registry.

### Q9. How do you implement Docker layer caching effectively in CI?

**Answer:**

A naive `docker build` in CI rebuilds every layer from scratch — slow and wasteful. Effective layer caching requires a remote cache and a Dockerfile structured for cache hits.

**Dockerfile structured for caching:**

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Layer 1: rarely changes
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Layer 2: changes often
COPY src/ ./src/

CMD ["python", "-m", "src.app"]
```

**Why the order matters:** Docker invalidates a layer and all subsequent layers when its inputs change. Putting `requirements.txt` before `src/` means a code-only change skips the slow `pip install`.

**BuildKit registry cache:**

```yaml
- uses: docker/setup-buildx-action@v3
- uses: docker/login-action@v3
  with: { registry: ghcr.io, username: ${{ github.actor }}, password: ${{ secrets.GITHUB_TOKEN }} }
- uses: docker/build-push-action@v5
  with:
    push: true
    tags: ghcr.io/my-org/app:${{ github.sha }}
    cache-from: type=registry,ref=ghcr.io/my-org/app:buildcache
    cache-to: type=registry,ref=ghcr.io/my-org/app:buildcache,mode=max
```

`mode=max` exports cache for all stages of a multi-stage build — useful when later stages reuse earlier work.

**GitHub Actions cache backend:**

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

Uses the GitHub Actions cache service. No extra registry storage needed; subject to the 10 GB / 7-day limits.

**Multi-platform builds with cache:**

```yaml
- uses: docker/build-push-action@v5
  with:
    platforms: linux/amd64,linux/arm64
    cache-from: type=registry,ref=ghcr.io/my-org/app:buildcache
    cache-to: type=registry,ref=ghcr.io/my-org/app:buildcache,mode=max
```

BuildKit caches per-platform automatically. The cache image stores both architectures.

**Pitfall:** `apt-get install` without `--no-install-recommends` and without cleanup pulls in many packages and a large lock file, busting cache on minor version changes. Always:

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*
```

### Q10. What is SLSA, and what do its levels mean in practice?

**Answer:**

**SLSA** (Supply-chain Levels for Software Artifacts, "salsa") is a framework for ranking the integrity of a software build process. Maintained by the OpenSSF, it has four levels — each one harder to defeat than the last.

**SLSA v1.0 build levels:**

| Level | Requirement | What it prevents |
|-------|-------------|------------------|
| **L1** | Documented build process; provenance generated (may be unsigned) | Confusion about which build produced an artefact |
| **L2** | Hosted build platform; signed provenance | Modification of provenance after the build |
| **L3** | Hardened, isolated build platform; non-falsifiable provenance | Tampering by build users; cross-build contamination |
| **L4** | (Removed in v1.0) — was hermetic, two-party review | (Now covered by Source levels) |

**What this looks like in practice:**

**L1 — Many internal pipelines.**

```yaml
- run: ./build.sh
- run: echo "Build by $GITHUB_ACTOR on $(date)" > provenance.txt
```

Provenance exists. It is not authenticated. Anyone with repo access can fake it.

**L2 — GitHub Actions with `attest-build-provenance`:**

```yaml
- uses: actions/attest-build-provenance@v2
  with: { subject-path: dist/myapp }
```

Provenance is generated by the GitHub-hosted runner and signed by the GitHub OIDC identity. The user cannot forge the OIDC signature.

**L3 — SLSA GitHub Generator (reusable workflow):**

```yaml
jobs:
  build:
    permissions: { id-token: write, contents: read, actions: read }
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.0.0
    with:
      base64-subjects: ${{ needs.hash.outputs.subjects }}
```

The build runs in a separate, locked-down workflow that the calling user cannot modify. Provenance is generated outside the user's reach. This is the highest level practically achievable on GitHub Actions today.

**L4 — Reproducible, hermetic builds with two-party review.** Rare outside of Google-internal Bazel + Borg pipelines.

**Why SLSA matters in interviews:**

Senior platform / security roles are increasingly expected to articulate "we are SLSA L2 today, working toward L3 by Q3." Knowing what each level requires (and what attacks each prevents) signals you've operated a real supply chain programme.

### Q11. What is an SBOM, and how do CycloneDX and SPDX differ?

**Answer:**

**SBOM** (Software Bill of Materials) is a machine-readable inventory of components in a piece of software — direct dependencies, transitive dependencies, versions, licences, hashes. After Executive Order 14028 (US, 2021), SBOMs are required for federal software procurement and increasingly demanded by enterprise customers.

**Two dominant formats:**

**CycloneDX** (OWASP, 2017):

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.6",
  "components": [
    {
      "type": "library",
      "name": "requests",
      "version": "2.31.0",
      "purl": "pkg:pypi/requests@2.31.0",
      "hashes": [{ "alg": "SHA-256", "content": "a1b2c3..." }],
      "licenses": [{ "license": { "id": "Apache-2.0" } }]
    }
  ]
}
```

**SPDX** (Linux Foundation, 2010):

```json
{
  "spdxVersion": "SPDX-2.3",
  "packages": [
    {
      "name": "requests",
      "versionInfo": "2.31.0",
      "licenseConcluded": "Apache-2.0",
      "checksums": [{ "algorithm": "SHA256", "checksumValue": "a1b2c3..." }]
    }
  ]
}
```

| Aspect | CycloneDX | SPDX |
|--------|-----------|------|
| Origin | OWASP, security focus | Linux Foundation, licence focus |
| Primary use case | Vulnerability management | Licence compliance |
| Support for VEX | Native (vulnerability exploitability exchange) | Via separate spec |
| Adoption | Strong in security tooling (Snyk, Trivy, Anchore) | Strong in licence tooling (FOSSology) |
| Format | JSON, XML, ProtoBuf | JSON, YAML, RDF, tag-value |

**Generating an SBOM in CI:**

```yaml
- name: Generate SBOM (CycloneDX)
  uses: anchore/sbom-action@v0
  with:
    format: cyclonedx-json
    output-file: sbom.cdx.json

- name: Generate SBOM (SPDX) for the same build
  uses: anchore/sbom-action@v0
  with:
    format: spdx-json
    output-file: sbom.spdx.json
```

**Attaching to a release artefact:**

```yaml
- uses: actions/attest-sbom@v2
  with:
    subject-path: dist/myapp
    sbom-path: sbom.cdx.json
```

The SBOM is signed and recorded in Sigstore alongside the artefact. Consumers can fetch and verify it with `gh attestation verify`.

**Practical recommendation:** generate both formats. Different consumers prefer different formats; the cost of producing both is negligible.

### Q12. How do you set retention and cleanup policies for a container registry?

**Answer:**

Without policy, registries fill up with millions of images: every PR, every commit, every nightly. Storage costs balloon and the UI becomes unusable.

**Categories of images and their retention needs:**

| Category | Example tag | Retention |
|----------|-------------|-----------|
| Production releases | `v2.4.1` | Forever (or 7+ years for compliance) |
| Pre-release / RC | `v2.4.1-rc.1` | 90 days after the release ships |
| Main branch builds | `main-a1b2c3d` | 30 days |
| PR builds | `pr-1234-a1b2c3d` | 7 days, or until PR closes |
| Nightly | `nightly-2025-04-15` | 14 days |
| Scratch / test | `scratch-*` | 24 hours |

**ECR lifecycle policy example:**

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep release tags forever",
      "selection": {
        "tagStatus": "tagged",
        "tagPatternList": ["v*.*.*"],
        "countType": "imageCountMoreThan",
        "countNumber": 1000000
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 10,
      "description": "Expire PR builds after 7 days",
      "selection": {
        "tagStatus": "tagged",
        "tagPatternList": ["pr-*"],
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 20,
      "description": "Expire untagged after 1 day",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 1
      },
      "action": { "type": "expire" }
    }
  ]
}
```

**Apply via Terraform:**

```hcl
resource "aws_ecr_lifecycle_policy" "app" {
  repository = aws_ecr_repository.app.name
  policy     = file("${path.module}/lifecycle-policy.json")
}
```

**GHCR lifecycle (via API or `actions/delete-package-versions`):**

```yaml
- uses: actions/delete-package-versions@v5
  with:
    package-name: my-app
    package-type: container
    min-versions-to-keep: 50
    delete-only-untagged-versions: false
```

**Subtle requirement: keep what's actually deployed.**

Before deleting "old" images, query Kubernetes for currently deployed image SHAs. If a six-month-old pod is still running because nobody rolled it, deleting its image breaks rescheduling. Deletion policy must intersect with deployment reality.

```bash
kubectl get pods -A -o json | \
  jq -r '.items[].spec.containers[].image' | sort -u > deployed-images.txt
```

**Interview insight:** the "how do you clean up your registry" question separates candidates who've operated production from those who haven't. The candidate who mentions "before deletion, intersect with deployed-image inventory" has been bitten before.

---

## Advanced

### Q13. Design an artefact promotion pipeline for a regulated environment with three environments (dev, staging, prod), provenance generation, and signed releases.

**Answer:**

Goal: build once in dev; promote unchanged through staging and prod; produce verifiable provenance and signatures at every step.

**Architecture:**

```
Source --build--> [Artifact + Provenance + SBOM] --signed--> dev/
                                                         |
                                                  --copy + sign--> staging/
                                                         |
                                                  --copy + sign--> prod/
```

The artefact bytes never change. Each promotion adds a new attestation (signed by the promoter's identity and the environment).

**Build phase — produce artefact + provenance + SBOM, sign with cosign:**

```yaml
permissions:
  id-token: write
  contents: read
  attestations: write
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with: { registry: ghcr.io, username: ${{ github.actor }}, password: ${{ secrets.GITHUB_TOKEN }} }

      - id: push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/my-org/app:dev-${{ github.sha }}
          provenance: true
          sbom: true

      # SLSA provenance
      - uses: actions/attest-build-provenance@v2
        with:
          subject-name: ghcr.io/my-org/app
          subject-digest: ${{ steps.push.outputs.digest }}
          push-to-registry: true

      # SBOM
      - uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/my-org/app@${{ steps.push.outputs.digest }}
          format: cyclonedx-json
          output-file: sbom.cdx.json
      - uses: actions/attest-sbom@v2
        with:
          subject-name: ghcr.io/my-org/app
          subject-digest: ${{ steps.push.outputs.digest }}
          sbom-path: sbom.cdx.json

      # Cosign keyless signature
      - uses: sigstore/cosign-installer@v3
      - run: cosign sign --yes ghcr.io/my-org/app@${{ steps.push.outputs.digest }}
```

**Promotion phase — copy by digest, add environment-specific attestation:**

```yaml
promote-to-staging:
  needs: build
  runs-on: ubuntu-latest
  environment: staging       # required reviewers
  permissions: { id-token: write, packages: write, attestations: write }
  steps:
    - uses: sigstore/cosign-installer@v3
    - name: Verify build provenance
      run: |
        cosign verify-attestation \
          --type slsaprovenance \
          --certificate-identity-regexp '^https://github.com/my-org/app/.*' \
          --certificate-oidc-issuer https://token.actions.githubusercontent.com \
          ghcr.io/my-org/app@${{ needs.build.outputs.digest }}

    - name: Re-tag for staging
      run: |
        crane copy \
          ghcr.io/my-org/app@${{ needs.build.outputs.digest }} \
          ghcr.io/my-org/app:staging-${{ github.sha }}

    - name: Attest staging promotion
      uses: actions/attest@v2
      with:
        subject-name: ghcr.io/my-org/app
        subject-digest: ${{ needs.build.outputs.digest }}
        predicate-type: https://my-org.example/promotion/v1
        predicate: |
          {
            "from": "dev",
            "to": "staging",
            "promoter": "${{ github.actor }}",
            "timestamp": "${{ github.event.created_at }}",
            "tests-passed": ["unit", "integration", "e2e"]
          }
```

**Production deploy verifies the chain:**

```yaml
deploy-prod:
  needs: promote-to-prod
  steps:
    - name: Verify production-readiness chain
      run: |
        # Original SLSA provenance must be present
        cosign verify-attestation --type slsaprovenance ghcr.io/my-org/app@$DIGEST
        # SBOM must be present
        cosign verify-attestation --type cyclonedx ghcr.io/my-org/app@$DIGEST
        # Promotion to staging must be attested
        cosign verify-attestation --type https://my-org.example/promotion/v1 \
          --certificate-identity 'staging-promoter@my-org' ghcr.io/my-org/app@$DIGEST
        # Promotion to prod must be attested
        cosign verify-attestation --type https://my-org.example/promotion/v1 \
          --certificate-identity 'prod-promoter@my-org' ghcr.io/my-org/app@$DIGEST
    - run: kubectl set image deployment/app app=ghcr.io/my-org/app@$DIGEST
```

**Properties this design provides:**

- The bytes deployed to prod are bit-identical to what was tested in dev/staging
- The complete deployment chain is cryptographically signed
- A compromised CI runner cannot inject an unsigned image into prod (admission controller enforces signature)
- Every promotion has a named human approver in the audit log
- An auditor can replay the chain from prod backward to source commit

### Q14. What is a Sigstore "transparency log," and how does cosign use it for keyless signing?

**Answer:**

Traditional code signing requires an org to manage long-lived signing keys — rotate them, store them in HSMs, recover from theft. **Keyless signing** with Sigstore eliminates the long-lived key by binding signatures to short-lived workload identities.

**How it works:**

1. The CI workload requests a short-lived OIDC token from GitHub Actions: `sub=repo:my-org/app:ref:refs/tags/v2.4.1`.
2. Cosign sends the token to **Fulcio** (Sigstore's CA). Fulcio verifies the token and issues a short-lived (10 min) X.509 code-signing cert containing the OIDC identity.
3. Cosign signs the artefact's digest with the ephemeral private key.
4. Cosign records the signature, certificate, and artefact digest in **Rekor** (the transparency log).
5. The ephemeral key is discarded.

**Verification:**

```bash
cosign verify ghcr.io/my-org/app@sha256:a1b2c3... \
  --certificate-identity 'https://github.com/my-org/app/.github/workflows/release.yml@refs/tags/v2.4.1' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com'
```

The verifier:

- Looks up the signature in Rekor (provides tamper-evident timestamp)
- Checks the cert was issued by Fulcio
- Verifies the cert's identity claims match expectations
- Verifies the signature against the artefact digest

**Why a transparency log:**

Even if Fulcio were compromised and issued a fraudulent cert, every issuance is publicly logged in Rekor. The legitimate org can detect "wait, I never signed that artefact" and the fraud is provable.

This mirrors **Certificate Transparency** for TLS certs. CT was invented after the DigiNotar breach (2011); Rekor brings the same property to code signing.

**What keyless signing eliminates:**

- HSM management
- Key rotation runbooks
- Key recovery procedures
- "Who has access to the signing key" discussions
- Long-lived secrets in CI

**What it requires:**

- Trust in the OIDC issuer (GitHub Actions, GitLab CI, etc.)
- Trust in Fulcio not to issue rogue certificates
- Internet egress from build runners (or self-hosted Sigstore — `sigstore-stack`)

**Interview insight:** mention that keyless signing has shifted the security boundary. Compromising your release process now requires compromising both your CI provider's OIDC issuer *and* leaving a public trail in Rekor. That's a much higher bar than "steal the key."

### Q15. You discover a critical vulnerability in a transitive dependency that's deployed to production. Walk through the response, and how your artefact management infrastructure helps.

**Answer:**

This is an incident response question. The artefact infrastructure should reduce time-to-detection, time-to-fix, and time-to-recovery.

**Step 1 — Determine blast radius (minutes).**

Query SBOMs for every deployed artefact. Which services include the vulnerable package and version?

```bash
# Anchore Grype against deployed images, or query an SBOM database
syft packages dir:./artifact-store --output json | \
  jq '.artifacts[] | select(.name=="lodash" and .version | startswith("4.17."))'
```

If you maintained SBOMs and stored them centrally, this query takes seconds. Without SBOMs, it takes hours of grep-ing build logs.

**Step 2 — Identify "known good" alternatives (minutes).**

Check the upstream's advisory: what versions are patched? Is there a backport? An exploit-detection signature?

**Step 3 — Block further deployments (minutes).**

Push an Artifactory Xray policy or admission controller rule that fails any image containing the vulnerable package version:

```yaml
# Kyverno policy
spec:
  rules:
    - name: block-vulnerable-lodash
      match: { resources: { kinds: [Pod] } }
      validate:
        message: "Image contains lodash < 4.17.21 (CVE-2021-23337)"
        pattern:
          spec:
            containers:
              - image: "!*lodash:4.17.20*"
```

This prevents the situation from worsening while you build a fix.

**Step 4 — Rebuild and promote (hours).**

Bump the dependency, build, run the standard pipeline, generate fresh SBOM and provenance, promote through environments. The "build once, promote many" infrastructure (Q5, Q13) ensures the patched artefact is the same bytes everywhere.

**Step 5 — Decommission the bad artefact (after deploy).**

Tag the vulnerable image as `quarantined`. Don't delete it immediately — you may need it for forensics. Deletion happens after a defined retention period.

**Step 6 — Record and audit (post-incident).**

- File a CVE response in your security incident tracker referencing the CHG ticket
- Update the SBOM database with the patched version
- Auditor question: "When was the vulnerable version last running in prod?" Answer comes from your deployment history + image scan log

**How the infrastructure paid off:**

| Capability | Time saved | Without it |
|------------|-----------|------------|
| Centralised SBOMs | Hours → minutes for blast radius | grep build logs across teams |
| Image scanning at promote | Caught at promotion, not deploy | Caught after prod incident |
| Admission control | Blocks new deploys instantly | Manual coordination across teams |
| Build once / promote many | One rebuild, propagate | Each team rebuilds independently |
| Provenance + signatures | Verify the patched image is the patched image | Trust without verification |

**Interview insight:** the punchline of any "respond to CVE-X" question is "we'd already invested in supply chain visibility, so the response was a query, not an investigation." Senior interviews want to hear that you've connected the artefact infrastructure to incident response, not just to deployment.

### Q16. Compare three approaches to dependency mirroring for an air-gapped production environment.

**Answer:**

When the production network has no internet egress, you need dependencies on the inside. Three architectures address this:

**Option A — Periodic full mirror.**

Tools: `bandersnatch` (PyPI), `verdaccio` (npm), `aptly` (Debian), `reposync` (RPM).

```bash
# Sync all of PyPI to a local mirror
bandersnatch mirror --json
# Result: ~25 TB of Python packages
```

Internal builds point at the mirror.

**Pros:**

- Self-contained; no on-demand egress needed
- Air-gap-compatible by transferring snapshots
- Predictable storage cost

**Cons:**

- Massive storage (PyPI is now ~25 TB; npm full-mirror impractical)
- Network bandwidth on each sync
- Mirror lag (you don't get day-zero packages instantly)
- You mirror packages you'll never use

**Option B — On-demand caching proxy (Artifactory remote repo).**

Internal builds point at Artifactory; Artifactory fetches from public registry on cache miss, caches the response.

**Pros:**

- Storage proportional to actual usage (often 100 GB, not 25 TB)
- New packages available on first request
- Audit log of what was actually consumed

**Cons:**

- Requires controlled egress from the proxy host (often allowed via outbound proxy + allow-list)
- First-use latency hits the public registry
- Doesn't work in true air-gap (no egress at all)

**Option C — Curated allow-list mirror.**

Maintain an explicit list of approved packages and versions. Tooling syncs only those.

```yaml
# allowed-packages.yml
- pypi:requests:2.31.0
- pypi:flask:3.0.0
- npm:react:18.2.0
```

```bash
pip download -r allowed-packages.txt --dest ./mirror/
```

**Pros:**

- Smallest storage footprint
- Strong supply chain control (only reviewed packages enter)
- Air-gap-friendly

**Cons:**

- High operational overhead (every dependency change is a ticket)
- Friction slows development
- Risk of "ghost packages" — required by something but missing from list

**Recommended hybrid for high-security air-gap:**

1. **Outer ring** — caching proxy with allow-list, internet egress permitted
2. **Inner ring (production)** — periodic snapshot from outer ring, no egress
3. **Promotion** — packages move from outer to inner only after Xray scan + license review

```
[Internet] -- allow-list proxy --> [Artifactory cache] -- snapshot --> [Air-gap mirror]
```

**Interview insight:** the right answer depends on the threat model and operational constraints. Naming both `bandersnatch` and `Artifactory remote repo` and discussing why an air-gapped DOD project might choose differently from a fintech in AWS shows you've operated in multiple environments.

### Q17. Design a multi-region container registry strategy for a service deployed in 6 AWS regions, with disaster recovery and minimal pull latency.

**Answer:**

Goals: (a) sub-100ms pull latency in every region, (b) tolerate single-region registry outage, (c) one source of truth for image versions, (d) reasonable cost.

**Strategy: regional ECR + cross-region replication.**

ECR supports cross-region replication (CRR) within an AWS account. Push to one "primary" region; ECR async-replicates to the others.

```hcl
resource "aws_ecr_replication_configuration" "this" {
  replication_configuration {
    rule {
      destination { region = "eu-west-1" registry_id = data.aws_caller_identity.current.account_id }
      destination { region = "ap-southeast-1" registry_id = data.aws_caller_identity.current.account_id }
      destination { region = "ap-northeast-1" registry_id = data.aws_caller_identity.current.account_id }
      destination { region = "us-east-1" registry_id = data.aws_caller_identity.current.account_id }
      destination { region = "sa-east-1" registry_id = data.aws_caller_identity.current.account_id }

      repository_filter {
        filter      = "prod-*"
        filter_type = "PREFIX_MATCH"
      }
    }
  }
}
```

Pods in each region pull from the local ECR endpoint (e.g., `123.dkr.ecr.eu-west-1.amazonaws.com/app`). No cross-region pulls in the hot path.

**Replication characteristics:**

- Async; typical lag is seconds to minutes
- "Push to primary" is the source of truth
- Replicated images carry the same digest in every region
- Cost: per-region storage + cross-region transfer (one-time per image)

**DR posture:**

- **Single region down (registry side):** pods in that region cannot pull new images. Existing pods continue running. Failover traffic to other regions.
- **Primary region (us-west-2) down for push:** CI cannot release new images. Failover plan: promote a secondary region to primary, repoint CI. Test this drill quarterly.
- **Single bad image:** delete from primary; replication propagates the deletion (or use Lifecycle Policy).

**Caching layer for pull-storm protection:**

For very high pull rates (massive autoscaling event), use ECR pull-through cache or a regional Spegel/registry cache so node-local pulls don't hammer ECR.

**Image signing across regions:**

Cosign signatures live as OCI artefacts in the registry. ECR replication carries them automatically. Verifier in any region sees the same signature.

**Cost optimisation:**

- Only replicate prod-tagged images (`prod-*` filter), not every PR build
- Aggressive lifecycle policy on dev/staging tags (Q12)
- Use compression (zstd) for image layers

**Alternative: Harbor with replication.**

For multi-cloud or on-prem + cloud, use Harbor (CNCF). Native replication policies between Harbor instances support pull-based, push-based, scheduled. More operational overhead than ECR but cloud-agnostic.

**Interview insight:** mention that "single registry across 6 regions" tends to fail under load and during regional events. Distributed-with-replication is the answer — and the rebuttal to interviewers who push back with "but isn't that complex?" is "the alternative is your prod regions becoming unable to scale because the registry hiccupped 2,000 km away."

### Q18. How does the SLSA framework relate to GitHub Actions, OIDC, Sigstore, and cosign — draw the architecture and explain the trust chain end to end.

**Answer:**

These pieces compose into a coherent supply chain. Walk through them as one system.

**The architecture (top to bottom):**

```
[Source repo] ----commit----> [GitHub Actions workflow]
                                       |
                                       | (id-token: write)
                                       v
                            [GitHub OIDC issuer] -- mints --> [JWT: sub=repo:owner/repo:ref:...]
                                                                       |
                                                                       v
                                                              [Fulcio CA] -- issues --> [Short-lived X.509]
                                                                                                |
[Build runs; produces artefact] -------- cosign sign with ephemeral key --------------------------+
                                                                                                |
                                                                                                v
                                                                                          [Rekor log]
                                                                                                |
[Artefact] + [Signature] + [Cert] + [Provenance] -----> pushed to registry ---> [GHCR / ECR]
                                                                                                |
                                                                                                |
[Admission controller in cluster] <----- pulls + verifies ---------------------------------------+
```

**Step-by-step trust chain:**

**1. Source.** A commit is pushed to GitHub. (SLSA Source levels begin here — ideally signed commits, branch protection, two-party review.)

**2. Build trigger.** A tag push triggers a release workflow.

**3. OIDC token mint.** The workflow requests an ID token. GitHub's OIDC issuer signs a JWT with claims:

```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "sub": "repo:my-org/app:ref:refs/tags/v2.4.1",
  "workflow": ".github/workflows/release.yml",
  "run_id": "12345",
  "exp": <now+5min>
}
```

This claim binds the build identity to the source repo, the tag, and the workflow file.

**4. Cert request to Fulcio.** Cosign sends the token. Fulcio verifies the token against GitHub's published JWKS, then issues a short-lived (10 min) X.509 signing cert containing the OIDC identity in `subjectAltName`.

**5. Signing.** Cosign generates an ephemeral key, signs the artefact's digest, attaches the cert. The private key is discarded.

**6. Rekor log entry.** Cosign uploads the signature, cert, and artefact digest to Rekor. Rekor returns a signed inclusion proof. This is the **non-falsifiable timestamp** — even if Fulcio later rotates keys, the entry remains verifiable.

**7. Provenance attestation.** A separate step (`actions/attest-build-provenance`) creates an in-toto SLSA Provenance attestation, signs it the same way (Fulcio cert + Rekor entry), and pushes to the registry.

**8. SBOM attestation.** Same pattern with the SBOM as the predicate.

**9. Push to registry.** The artefact and all its attestations land in GHCR/ECR.

**10. Deployment-time verification.** The cluster's admission controller (Kyverno, Connaisseur, sigstore policy-controller) intercepts every pod creation:

```yaml
# Kyverno policy snippet
verifyImages:
  - imageReferences: ["ghcr.io/my-org/app:*"]
    attestors:
      - entries:
        - keyless:
            subject: "https://github.com/my-org/app/.github/workflows/release.yml@refs/tags/v*.*.*"
            issuer: "https://token.actions.githubusercontent.com"
```

The controller fetches the cert from the registry, verifies the chain to Fulcio, fetches the Rekor inclusion proof, validates the SAN identity matches the policy. If anything fails, the pod is rejected.

**SLSA levels achieved:**

- **L1:** Documented build, provenance generated → trivially yes
- **L2:** Hosted build (GitHub-hosted runner), signed provenance → yes
- **L3:** Provenance generated by an isolated builder the user cannot tamper with → yes (with `slsa-github-generator` reusable workflow)

**What attacks this prevents:**

| Attack | Prevented by |
|--------|--------------|
| Attacker pushes malicious image | Admission controller rejects unsigned image |
| Attacker compromises CI runner and signs malicious image | Fulcio cert binds to specific workflow + tag; admission policy mismatches |
| Attacker compromises Fulcio | Rogue cert appears in Rekor; legitimate org detects and revokes |
| Provenance is forged | Provenance is signed by the same OIDC chain |
| Old vulnerable image is redeployed | Lifecycle policies + admission policies on minimum versions |

**What remains as residual risk:**

- Compromise of GitHub Actions OIDC issuer (Anthropic of supply chain)
- Compromise of an action's source code (mitigation: SHA pinning)
- Compromise of the source commit (mitigation: signed commits, two-party review)

**Interview insight:** if you can sketch this diagram on a whiteboard and explain each arrow, you have demonstrated mastery of modern supply chain security. The components are independent open-source projects that compose into a coherent system — this is the modern equivalent of being able to explain TLS handshakes a decade ago.
