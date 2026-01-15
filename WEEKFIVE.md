# CI/CD Pipeline — Week Five

A structured reference for building a DevSecOps CI pipeline with GitHub Actions (Java, Docker, security scanning).

---

## Table of Contents

1. [Part 1: Pipeline Overview & Requirements](#part-1-pipeline-overview--requirements)
2. [Part 2: Workflow Setup](#part-2-workflow-setup)
3. [Part 3: Pipeline Stages](#part-3-pipeline-stages)
4. [Part 4: Principles & Enhancements](#part-4-principles--enhancements)

---

# Part 1: Pipeline Overview & Requirements

## What This Pipeline Does

A **DevSecOps CI pipeline** that runs on every push (and optionally manually):

1. Checkout code
2. Build Java app (Maven)
3. Lint (e.g. Checkstyle)
4. Run unit tests
5. **SAST** (Static Application Security Testing) — e.g. CodeQL
6. **SCA** (Software Composition Analysis) — e.g. OWASP Dependency-Check
7. Build Docker image
8. **Container scan** (e.g. Trivy)
9. **Runtime check** — run container, hit health endpoint
10. Push image to registry (e.g. Docker Hub) only if all steps pass

Security is **shift-left**: code and dependencies are scanned before the image is published.

---

## Flow

```
Code push → Runner → Checkout → Build → Lint → SAST → SCA → Test →
Build JAR → Docker build → Trivy scan → Runtime test → Login → Push image
```

---

## Prerequisites

| Area         | Requirement                                                       |
| ------------ | ----------------------------------------------------------------- |
| **App**      | Java project (Maven), Dockerfile, app on port 8080 (or your port) |
| **Registry** | Docker Hub (or other) account; **username** and **access token**  |
| **GitHub**   | Repo **Secrets**: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`         |

**Secrets:** Repository → **Settings → Secrets and variables → Actions** → New repository secret.

| Secret               | Value                                              |
| -------------------- | -------------------------------------------------- |
| `DOCKERHUB_USERNAME` | Your Docker Hub username                           |
| `DOCKERHUB_TOKEN`    | Docker Hub access token (from Docker Hub settings) |

---

# Part 2: Workflow Setup

## Triggers

```yaml
on:
  push:
    branches:
      - master
  workflow_dispatch:
```

- Runs on **push to master**.
- **workflow_dispatch** allows manual run from the Actions tab.

---

## Job and permissions

```yaml
jobs:
  ci-pipeline:
    runs-on: ubuntu-latest

permissions:
  contents: read
  security-events: write
```

- **contents: read** — checkout and read repo.
- **security-events: write** — publish SAST/scan results to **Security → Code scanning alerts**.

---

# Part 3: Pipeline Stages

Stages run as **steps** in a single job (or split into jobs if you prefer). Order below is typical.

---

## 1. Checkout

```yaml
- uses: actions/checkout@v4
```

Clones the repo into the runner workspace.

---

## 2. Setup Java

```yaml
- uses: actions/setup-java@v4
  with:
    distribution: "temurin"
    java-version: "17"
    cache: "maven"
```

Installs JDK and can cache Maven dependencies.

---

## 3. Lint (code quality)

```yaml
- run: mvn checkstyle:check
  continue-on-error: true
```

Enforces style; `continue-on-error: true` keeps the pipeline running while still showing failures in logs.

---

## 4. SAST (Static Application Security Testing)

Example: **CodeQL**

```yaml
- uses: github/codeql-action/init@v2
  with:
    languages: java
- run: mvn clean compile
- uses: github/codeql-action/analyze@v2
```

Finds issues in **source code** (e.g. SQL injection, command injection, OWASP Top 10). Results appear under **Security → Code scanning**.

---

## 5. SCA (dependency vulnerabilities)

Example: **OWASP Dependency-Check**

```yaml
- uses: dependency-check/Dependency-Check_Action@main
  with:
    project: "my-app"
    path: "."
    format: "SARIF"
    output: "dependency-check-results.sarif"
```

Scans **Maven dependencies** for known CVEs. Can upload SARIF to GitHub Security (similar to CodeQL).

---

## 6. Run tests

```yaml
- run: mvn test
```

Unit/integration tests; pipeline fails if tests fail.

---

## 7. Build artifact

```yaml
- run: mvn clean package -DskipTests
```

Produces JAR/WAR for Docker build (tests already run in step 6).

---

## 8. Build Docker image

```yaml
- run: docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ github.sha }} .
```

Tag with commit SHA (or branch/tag) for traceability.

---

## 9. Container vulnerability scan (Trivy)

```yaml
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ github.sha }}
    format: "sarif"
    output: "trivy-results.sarif"
    exit-code: "1"
```

- **exit-code: 1** — fail pipeline on critical/high (configurable).
- Scans OS packages and app dependencies inside the image.

**Upload to GitHub Security:**

```yaml
- uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: "trivy-results.sarif"
```

---

## 10. Runtime validation

```yaml
- run: |
    docker run -d -p 8080:8080 --name app ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ github.sha }}
    sleep 10
    curl -f http://localhost:8080/health || exit 1
    docker stop app
```

Checks that the container starts and the app responds (e.g. health endpoint).

---

## 11. Registry login

```yaml
- uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

Use equivalent for ECR, GCR, etc., if not Docker Hub.

---

## 12. Push image

```yaml
- run: docker push ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ github.sha }}
```

Only runs if all previous steps passed — image is promoted only after tests and security checks.

---

## Stages summary

| #   | Stage        | Purpose                                         |
| --- | ------------ | ----------------------------------------------- |
| 1   | Checkout     | Get source                                      |
| 2   | Setup Java   | JDK + Maven cache                               |
| 3   | Lint         | Code style                                      |
| 4   | SAST         | Source code security (e.g. CodeQL)              |
| 5   | SCA          | Dependency vulnerabilities (e.g. OWASP)         |
| 6   | Test         | Unit/integration tests                          |
| 7   | Build        | JAR/WAR                                         |
| 8   | Docker build | Image                                           |
| 9   | Trivy        | Image vulnerability scan; optional SARIF upload |
| 10  | Runtime test | Container runs and responds                     |
| 11  | Login        | Registry auth                                   |
| 12  | Push         | Publish image                                   |

---

# Part 4: Principles & Enhancements

## DevOps & security principles

- **CI automation** — every push runs the same pipeline.
- **Shift-left security** — SAST, SCA, and container scan before promotion.
- **DevSecOps** — security steps are part of the pipeline, not an afterthought.
- **Supply chain** — dependencies and container layers are scanned.
- **Immutable artifacts** — tagged images (e.g. by SHA); no “latest” only.
- **Quality gates** — pipeline fails on test or security failure.
- **Secrets** — credentials only in GitHub Secrets (or equivalent); never in code.

---

## What makes it “enterprise-ready”

- Multiple security layers: **code + dependencies + container**.
- **Fail on critical/high** vulnerabilities (Trivy/CodeQL config).
- **Runtime check** before push.
- **Controlled promotion** — only successful runs push.
- **Secrets** for registry and sensitive data.
- **Security tab** integration (SARIF uploads).

---

## Possible enhancements

| Area          | Idea                                           |
| ------------- | ---------------------------------------------- |
| Quality       | SonarQube for coverage and quality gates       |
| Supply chain  | SBOM generation (e.g. Syft, Trivy)             |
| Registry      | Use AWS ECR, GCR, etc.                         |
| Deploy        | CD to EKS, ECS, or K8s                         |
| Notifications | Slack/Teams on failure or release              |
| Versioning    | Semantic versioning or tag strategy            |
| Infra         | Separate pipeline for Terraform/CloudFormation |

---

## Summary

This pipeline is a **DevSecOps CI gateway**: build, test, SAST, SCA, container scan, and runtime check before pushing the image. It ensures only validated, scanned artifacts are published and integrates findings into GitHub Security.

---

_End of Week Five notes._
