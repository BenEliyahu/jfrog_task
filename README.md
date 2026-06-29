# JFrog Task — Stage 2

**JFrog Instance:** https://beneliyahu.jfrog.io  
**Fork of:** https://github.com/yonarbel/jfrog_task

---

## What This App Does

A Node.js/Express REST API for user management, intentionally built with outdated dependencies to demonstrate end-to-end DevSecOps scanning and remediation.

**Stack:** Node.js 18, Express 4, axios, lodash, moment, jsonwebtoken

---

## Setup & CI/CD

- **JFrog Free Trial:** https://beneliyahu.jfrog.io
- **npm packages** resolved via JFrog Artifactory (`npm-virtual` → `npm-remote` → npmjs.org)
- **Docker image** pushed to `beneliyahu.jfrog.io/docker-local`
- **Build Info** published to Artifactory on every push via JFrog CLI
- **CI:** GitHub Actions — `.github/workflows/publish-build.yml`

---

## a. Vulnerability Scan Results

Scanned using **JFrog Xray** with Contextual Analysis enabled.

Raw scan: **161 total CVEs** across all components.  
After Contextual Analysis: **2 applicable High vulnerabilities** in the npm app.

| CVE | Package | Severity | CVSS | Fix Version | Status |
|---|---|---|---|---|---|
| CVE-2024-45590 | body-parser 1.x | High | 8.7 | 1.20.3 | Not remediated (transitive) |
| CVE-2022-31129 | moment 2.19.3 | High | 7.5 | 2.29.4 | **Remediated** ✅ |

### Before remediation — Build #7

![Before remediation](screenshots/before.png)

161 CVEs total. Both moment (CVE-2022-31129) and body-parser (CVE-2024-45590) flagged as Applicable.

### After remediation — Build #8

![After remediation](screenshots/after.png)

159 CVEs total. moment CVE-2022-31129 removed. body-parser remains (transitive dependency).

### Versions Diff — Build #7 → Build #8

![Versions diff](screenshots/diff.png)

| Metric | Build #7 | Build #8 | Delta |
|---|---|---|---|
| Total CVEs | 161 | 159 | −2 |
| Applicable High | 2 | 1 | −1 |
| Resolved | — | 2 | CVE-2022-31129, CVE-2022-24785 |

---

## b. What is "Applicable"? — Contextual Analysis

A standard vulnerability scanner reports every CVE in every package inside `node_modules`, regardless of whether the application actually exercises the vulnerable code path.

**Contextual Analysis** (JFrog Xray) traces the application's actual call graph to determine whether the vulnerable function is reachable with user-controlled input in this specific context.

### Example from this app

`lodash 4.17.4` contains a known Prototype Pollution CVE in `_.template()`. This app never calls `_.template()` — so Xray marks it **Not Applicable**. It cannot be exploited here.

By contrast, `moment 2.19.3` has a ReDoS vulnerability that triggers when parsing certain date strings. The app calls `moment()` in `users.js` with data that flows from user requests — so Xray marks it **Applicable**.

### Why this changes prioritization

| Without Contextual Analysis | With Contextual Analysis |
|---|---|
| 161 vulnerabilities to triage | 2 vulnerabilities to act on |
| Security team overwhelmed | Clear, focused remediation |
| Risk of fixing the wrong things | Fix what actually threatens the app |

Instead of spending days triaging 161 CVEs, the team focuses on the 2 that can actually be exploited — a **98% reduction in noise**.

---

## c. Remediation

### Vulnerability fixed: CVE-2022-31129 — moment ReDoS

**Attack:** An attacker sends a crafted date string that triggers catastrophic regex backtracking inside moment's parsing logic, hanging the Node.js event loop and causing denial of service.

**Why it's reachable:** `moment()` is called in `users.js` to generate timestamps. The input flows from HTTP request data — the vulnerable code path is directly reachable.

**Fix applied:**

```json
"moment": "2.29.4"
```

Upgraded from 2.19.3 to 2.29.4 — the first version that patches CVE-2022-31129.

### Why moment was fixed instead of body-parser (CVSS 8.7)

body-parser is a **transitive dependency** bundled inside Express. It cannot be upgraded independently without risking a version mismatch with Express internals. A safe fix requires upgrading Express itself, which is a larger change with broader regression risk.

`moment` is a **direct dependency** with an identical API between 2.19.3 and 2.29.4 — zero code changes required, minimal regression surface, and the fix is verifiable with a single scan. This made it the correct first target for immediate remediation.

---

## d. Docker Image & Build Info

Every push to `main` triggers the GitHub Actions workflow which:

1. Configures npm to resolve packages via JFrog Artifactory (`npm-virtual`)
2. Installs dependencies with `jf npm install` — tracked in Build Info
3. Runs `jf audit` — Xray security scan against the build
4. Builds Docker image tagged with build number + `latest`
5. Pushes to `beneliyahu.jfrog.io/docker-local/jfrog-task-app`
6. Collects environment info via `jf rt bce`
7. Publishes Build Info to Artifactory via `jf rt bp`

Build Info links every npm package, Docker layer, environment variable, and commit SHA to a single traceable build record in Artifactory — full provenance from code to artifact.

**Docker image:** `beneliyahu.jfrog.io/docker-local/jfrog-task-app:latest`

---

## GitHub Actions Workflow

`.github/workflows/publish-build.yml`

```yaml
name: Build and Publish to JFrog

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  IMAGE_NAME: jfrog-task-app
  BUILD_NAME: jfrog-task

jobs:
  build-and-publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '18'

      - uses: jfrog/setup-jfrog-cli@v4
        env:
          JF_URL: ${{ secrets.JF_URL }}
          JF_ACCESS_TOKEN: ${{ secrets.JF_ACCESS_TOKEN }}

      - run: jf npm-config --repo-resolve npm-virtual

      - run: jf npm install --build-name=${{ env.BUILD_NAME }} --build-number=${{ github.run_number }}

      - run: jf audit --npm --fail=false

      - run: echo "${{ secrets.JF_ACCESS_TOKEN }}" | docker login beneliyahu.jfrog.io -u beneliyahu --password-stdin

      - run: docker build -t beneliyahu.jfrog.io/docker-local/jfrog-task-app:${{ github.run_number }} -t beneliyahu.jfrog.io/docker-local/jfrog-task-app:latest .

      - run: jf docker push beneliyahu.jfrog.io/docker-local/jfrog-task-app:${{ github.run_number }} --build-name=${{ env.BUILD_NAME }} --build-number=${{ github.run_number }}

      - run: jf rt bce ${{ env.BUILD_NAME }} ${{ github.run_number }}

      - run: jf rt bp ${{ env.BUILD_NAME }} ${{ github.run_number }}
```

### Required Secrets

| Secret | Value |
|---|---|
| `JF_URL` | `https://beneliyahu.jfrog.io` |
| `JF_ACCESS_TOKEN` | JFrog access token with deploy permissions |

---

## JFrog Artifactory Configuration

| Resource | Type | Purpose |
|---|---|---|
| `npm-virtual` | Virtual repository | Single npm registry endpoint for CI |
| `npm-remote` | Remote repository | Proxies https://registry.npmjs.org |
| `docker-local` | Local repository | Stores built Docker images |

The virtual repository aggregates the remote — CI only needs one URL, and JFrog handles caching and proxying transparently.
