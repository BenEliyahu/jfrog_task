# JFrog Task — Stage 2

**JFrog Instance:** https://beneliyahu.jfrog.io  
**Fork of:** https://github.com/yonarbel/jfrog_task

---

## What This App Does

A Node.js/Express REST API for user management, built intentionally with
outdated dependencies to demonstrate DevSecOps scanning and remediation.

**Stack:** Node.js 18, Express 4, axios, lodash, moment, jsonwebtoken

---

## Setup & CI/CD

- **JFrog Free Trial:** https://beneliyahu.jfrog.io
- **npm packages** resolved via JFrog Artifactory (`npm-virtual`)
- **Docker image** pushed to `beneliyahu.jfrog.io/docker-local`
- **Build Info** published to Artifactory on every push via JFrog CLI
- **CI:** GitHub Actions — `.github/workflows/publish-build.yml`

---

## a. Vulnerability Scan Results

Scanned using **JFrog Xray** with Contextual Analysis enabled.

Raw scan: **161 total CVEs** across all components.  
After Contextual Analysis: **2 applicable High vulnerabilities** in the npm app.

| CVE | Package | Severity | Fix Version | Why Applicable |
|---|---|---|---|---|
| CVE-2024-45590 | body-parser 1.x | High 8.7 | 1.20.3 | App processes user HTTP request bodies via `express.json()` — the vulnerable parsing code runs on every request |
| CVE-2022-31129 | moment 2.19.3 | High 7.5 | 2.29.4 | App calls `moment()` in `users.js` to generate timestamps — the vulnerable regex path is reachable |

---

## b. What is "Applicable" / Contextual Analysis?

A standard vulnerability scanner looks at every package in `node_modules`
and reports every known CVE — regardless of whether your code actually
uses the vulnerable function.

**Contextual Analysis** (JFrog Xray) goes further: it traces the actual
call graph of your application to determine whether the vulnerable code
path is reachable with user-controlled input.

**Example from this app:**  
`lodash 4.17.4` contains a known Prototype Pollution CVE in `_.template()`.
But this app never calls `_.template()` — so Xray marks it as
**Not Applicable**. It won't be exploited in this specific context.

By contrast, `moment 2.19.3` has a ReDoS vulnerability that triggers when
parsing certain date strings. The app calls `moment()` in `users.js` with
data that flows from user input — so Xray marks it as **Applicable**.

**Why this changes prioritization:**

| Without Contextual Analysis | With Contextual Analysis |
|---|---|
| 161 vulnerabilities to triage | 2 vulnerabilities to fix |
| Security team overwhelmed | Clear, focused action |
| Risk of fixing the wrong things | Fix what actually matters |

Instead of spending days triaging 161 CVEs, the team focuses exclusively
on the 2 that can actually be exploited — a **98% reduction in noise**.

---

## c. Remediation

**Chose to fix: CVE-2022-31129 (moment ReDoS)**

**Why this one:**  
- `moment` is a **direct dependency** in `package.json` — simple to update
- The fix is non-breaking: API is identical between 2.19.3 and 2.29.4
- The vulnerability (ReDoS) can cause **denial of service** by sending
  a crafted date string that causes catastrophic regex backtracking

**Approach:**  
Updated `package.json`:
```json
"moment": "2.29.4"
