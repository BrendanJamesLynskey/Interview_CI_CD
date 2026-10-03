# CI/CD — Interview Preparation

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Subject: CI/CD](https://img.shields.io/badge/Subject-CI%2FCD-blue)](https://en.wikipedia.org/wiki/CI/CD)

## Overview

This repository provides interview preparation material for software engineering roles that require practical expertise in continuous integration, continuous delivery, and continuous deployment. The content covers pipeline design, deployment strategies, release management, tooling, testing in CI, and modern supply chain security.

The material targets Senior Software Engineers interviewing for backend, platform, DevOps, and SRE-adjacent positions at companies that ship software frequently and safely. Code examples use realistic GitHub Actions YAML, Jenkinsfiles, GitLab CI, and shell/Python snippets that mirror real production pipelines.

## Table of Contents

- [01 Continuous Integration](#01-continuous-integration)
- [02 Continuous Delivery and Deployment](#02-continuous-delivery-and-deployment)
- [03 Pipeline Tooling](#03-pipeline-tooling)
- [04 Pipeline Quality and Security](#04-pipeline-quality-and-security)
- [05 Quizzes](#05-quizzes)
- [How to Use](#how-to-use)
- [Related Repositories](#related-repositories)
- [Contributing](#contributing)
- [License](#license)

### 01 Continuous Integration

The foundations of CI: running builds and tests automatically on every change, integrating work frequently, and keeping `main` releasable.

- [`build_pipelines.md`](01_continuous_integration/build_pipelines.md) — Pipeline stages, triggers, build agents, caching, parallelism
- [`source_control_integration.md`](01_continuous_integration/source_control_integration.md) — Git workflows (GitFlow, trunk-based), branch protection, webhooks, monorepo strategies

### 02 Continuous Delivery and Deployment

Getting validated changes into production safely, with strategies to limit blast radius and recover quickly when something goes wrong.

- [`deployment_strategies.md`](02_continuous_delivery_and_deployment/deployment_strategies.md) — Blue-green, canary, rolling, feature flags, dark launches
- [`release_management.md`](02_continuous_delivery_and_deployment/release_management.md) — Semantic versioning, changelogs, release gates, rollback strategies

### 03 Pipeline Tooling

The concrete tools that implement CI/CD in practice, and the ecosystem of registries and caches they depend on.

- [`jenkins_and_github_actions.md`](03_pipeline_tooling/jenkins_and_github_actions.md) — Jenkins pipelines, GitHub Actions, GitLab CI, CircleCI
- [`artifact_management.md`](03_pipeline_tooling/artifact_management.md) — Artifact repositories, container registries, dependency caching, provenance

### 04 Pipeline Quality and Security

Ensuring that the CI/CD system itself produces trustworthy output — fast, correct, and tamper-resistant.

- [`testing_in_pipelines.md`](04_pipeline_quality_and_security/testing_in_pipelines.md) — Test pyramid in CI, flaky tests, test parallelisation, test selection
- [`supply_chain_security.md`](04_pipeline_quality_and_security/supply_chain_security.md) — SBOMs, SLSA, signed commits, secret scanning, dependency scanning
- [`compliance_and_governance.md`](04_pipeline_quality_and_security/compliance_and_governance.md) — Audit trails, approval workflows, SOC2, change management

### 05 Quizzes

Self-assessment quizzes covering each major topic area.

- [`quiz_ci_and_delivery.md`](05_quizzes/quiz_ci_and_delivery.md) — 25 questions from sections 01 and 02
- [`quiz_tooling_and_quality.md`](05_quizzes/quiz_tooling_and_quality.md) — 25 questions from sections 03 and 04

## How to Use

This repository is structured as a progressive CI/CD interview course:

1. **Start with continuous integration.** Understand what happens between `git push` and a green build — stages, caching, agents, and how source control integrates with the pipeline. Most CI/CD interview questions start here.

2. **Move to delivery and deployment.** Deployment strategies (blue-green, canary, rolling) and release management appear in nearly every senior-level pipeline interview. Know the trade-offs and failure modes.

3. **Study pipeline tooling.** Be able to write a passable GitHub Actions workflow or Jenkinsfile on a whiteboard. Understand artifact management and the caching hierarchy that keeps pipelines fast.

4. **Learn pipeline quality and security.** Senior interviews increasingly focus on flaky tests, supply chain attacks (SolarWinds, Codecov, xz), SLSA levels, and compliance. These separate staff-level candidates from mid-level ones.

5. **Use the quizzes** to identify weak areas and return to the relevant section.

## Related Repositories

- **[Interview_DevOps](https://github.com/BrendanJamesLynskey/Interview_DevOps)** — DevOps practices, infrastructure as code, observability
- **[Interview_Software_Testing](https://github.com/BrendanJamesLynskey/Interview_Software_Testing)** — Test strategy, automation, and quality engineering
- **[Interview_Security_Engineering](https://github.com/BrendanJamesLynskey/Interview_Security_Engineering)** — Application and infrastructure security
- **[Interview_System_Design](https://github.com/BrendanJamesLynskey/Interview_System_Design)** — System design interview preparation

## Related Repositories

- **[Introduction to Jenkins](https://brendanjameslynskey.github.io/Introduction_to_Jenkins/)** — a newcomer's tour: controller and agents, plugins, declarative and scripted pipelines, credentials, test reports, shared libraries, security, and Jenkins vs GitHub Actions vs GitLab CI, with real screenshots ([repo](https://github.com/BrendanJamesLynskey/Introduction_to_Jenkins))
- **[Jenkins for Hardware and Simulation Teams](https://brendanjameslynskey.github.io/SimEng_07_Jenkins_for_Simulation_Teams/)** — declarative and scripted pipelines, matrix builds, shared libraries, JUnit and coverage, nightly regressions, performance gates and credentials, from real Jenkins runs of five simulator repositories ([Simulation Engineering Toolkit](https://github.com/BrendanJamesLynskey/SimEng_Hub_Toolkit))
- **[Performance Analysis of Simulators and Systems](https://brendanjameslynskey.github.io/SimEng_11_Performance_Analysis/)** — designing CI regression gates from measured timing noise ([Simulation Engineering Toolkit](https://github.com/BrendanJamesLynskey/SimEng_Hub_Toolkit))

## Contributing

Contributions are welcome. Please ensure:

1. Content is technically accurate and reflects current industry practice
2. YAML examples are valid and realistic (test with `actionlint`, `yamllint`, or the Jenkins linter where applicable)
3. Trade-offs are presented honestly — every CI/CD choice has costs
4. Quiz questions reflect realistic Senior SWE interview scenarios

For significant additions, please open an issue first to discuss scope and approach.

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.
