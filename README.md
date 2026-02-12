# CI/CD Strategy (Front-End)

This document defines the CI/CD strategy for the PlanOS front-end (React + Vite) with an enterprise-ready, SOC-2-friendly change management posture. The goals are:

- Prevent unreviewed or untested code from reaching protected branches
- Provide traceability (who changed what, why, and when)
- Reduce regressions through automated validation
- Detect security and supply-chain risks early
- Produce reproducible, deployable artifacts with a clear release trail

---

## Goals

### Primary
- **Quality:** Every merge to protected branches compiles, passes tests, and meets lint rules.
- **Security:** Dependencies and build output are continuously assessed for known risks.
- **Reliability:** Builds are reproducible and releases can be rolled back.
- **Auditability:** PR + CI provide a clear record of approvals, checks, and artifacts.

### Non-Goals
- This document does not prescribe back-end deployment strategy.
- This document does not replace engineering judgment; it standardizes the baseline.

---

## Branching and Environment Model

### Branches
- `main`: production-ready code (protected)
- `develop` (optional): integration branch for teams that prefer it (protected)
- `feature/*`: feature work
- `fix/*`: bug fixes
- `release/*` (optional): stabilization prior to production

> Reasoning: A predictable branch model reduces accidental production changes and makes audit trails straightforward.

### Environments
- **Preview (PR)**: ephemeral environment per pull request (optional but recommended)
- **Staging**: integration environment for QA / UAT
- **Production**: customer-facing release

> Reasoning: Preview builds catch integration issues earlier and reduce “works on my machine” risk.

---

## Required Engineering Workflow (Change Management)

### 1) Pull Requests Are Mandatory
**Rule:** No direct pushes to `main` (and `develop` if used). All changes merge via PR.

**Reasoning:**
- Enforces review and discussion
- Creates an auditable change record
- Enables automated checks before merging

### 2) Minimum Review Requirements
**Rule:** PRs require at least **1 approval** (2 for critical modules when the team grows). Reviewers cannot be the author.

**Reasoning:**
- Reduces single-person risk (accidental or malicious)
- Catches correctness, security, and maintainability issues
- Matches common enterprise expectations for SDLC controls

### 3) Required Status Checks Before Merge
**Rule:** The following CI checks must pass before merge:
- Lint
- Typecheck
- Unit tests
- Build
- Dependency / security scan

**Reasoning:**
- Prevents broken builds and obvious regressions from reaching protected branches
- Creates consistent quality gates independent of individual discipline

### 4) Linear History / Squash Merge (Recommended)
**Rule:** Prefer squash merges from feature branches into protected branches.

**Reasoning:**
- Keeps history clean and easier to audit
- Avoids merging partial intermediate commits that were not intended as “release state”

---

## CI Pipeline Overview

CI runs automatically on:
- `pull_request` (validate changes before merge)
- `push` to protected branches (create artifacts and deploy)
- `workflow_dispatch` (manual runs for hotfixes or re-deploys)

### Pipeline Stages

1. **Checkout & Setup**
2. **Install Dependencies (locked)**
3. **Static Analysis**
4. **Unit Tests**
5. **Build**
6. **Security / Supply-Chain Checks**
7. **Artifact Packaging**
8. **Deploy**
9. **Post-Deploy Verification**

---

## 1) Checkout & Setup

### Actions
- Checkout repository
- Set Node version to a pinned major (e.g., Node 20)
- Enable dependency caching

### Reasoning
- Pinned Node version avoids environment drift
- Caching speeds up CI and reduces failure rates due to network timeouts
- Consistent setup improves reproducibility

---

## 2) Dependency Installation (Locked)

### Actions
- Use a lockfile (`package-lock.json` / `pnpm-lock.yaml` / `yarn.lock`)
- Install using a lockfile-respecting command:
  - npm: `npm ci`
  - pnpm: `pnpm i --frozen-lockfile`
  - yarn: `yarn --frozen-lockfile`

### Reasoning
- Prevents “works locally” dependency differences
- Ensures the exact dependency graph is tested and shipped
- Reduces supply-chain risk by controlling versions

---

## 3) Static Analysis (Lint + Formatting + Typecheck)

### Actions
- ESLint: `npm run lint`
- TypeScript: `npm run typecheck` (or `tsc -p tsconfig.json --noEmit`)

### Reasoning
- Catches common defects early (unsafe patterns, unused code, incorrect assumptions)
- Typecheck prevents runtime errors that are hard to test exhaustively
- Ensures consistent codebase quality and readability

**Recommended policy:** CI fails on lint/typecheck errors (no warnings allowed on protected branches).

---

## 4) Unit Tests

### Actions
- Run unit tests with a consistent runner (e.g., Vitest/Jest)
- Collect coverage on protected-branch builds (optional but recommended)

### Reasoning
- Prevents regressions and validates component logic
- Coverage provides directional signal over time (not a perfect metric)
- Enterprise buyers often expect documented automated testing discipline

**Recommended policy:**
- PR: run tests
- Protected branch: run tests + coverage upload

---

## 5) Build

### Actions
- Production build: `npm run build` (Vite)
- Validate that the build completes without warnings that indicate broken config

### Reasoning
- Ensures the repo is deployable on every merge
- Catches bundling issues, env var issues, tree-shaking problems, and invalid imports
- Prevents shipping changes that compile only in dev mode

---

## 6) Security & Supply-Chain Checks

### Actions (minimum)
- Dependency audit (choose one approach):
  - `npm audit --production` (baseline)
  - or a dedicated scanner (Snyk, Dependabot alerts, etc.)
- Secret scanning (recommended)
- License policy checks if required (recommended)

### Reasoning
- Most real-world compromises occur through dependencies or leaked secrets
- Automated scanning provides early warning and creates a documented security posture
- Supports SOC 2-aligned secure development practices

**Recommended policy:**
- Fail CI on **critical** vulnerabilities with known exploits
- Allow temporary exceptions only with documented justification and ticket reference

---

## 7) Artifact Packaging & Provenance

### Actions
- Attach build metadata to artifacts:
  - git commit SHA
  - branch
  - build timestamp
  - CI run ID
- Publish build artifacts (e.g., `dist/`) for staging/production deployments
- (Optional) Generate an SBOM (software bill of materials)

### Reasoning
- Enables rollback and forensic traceability
- Helps prove “what code is running” in a given environment
- SBOM improves enterprise trust and helps with vulnerability response

---

## 8) Deploy Strategy

### Preview Deployments (PR) — Recommended
**Trigger:** `pull_request`  
**Deploy:** ephemeral preview environment (Netlify/Vercel/S3+CloudFront/etc.)

**Reasoning:**
- Lets reviewers validate UX, routing, and integration in a real environment
- Reduces risk by testing changes in a production-like runtime before merging

### Staging Deployments
**Trigger:** merge to `develop` or `main` with a staging flag  
**Deploy:** staging environment

**Reasoning:**
- Enables QA/UAT and integration checks
- Allows controlled testing of environment variables and API connectivity

### Production Deployments
**Trigger:** version tag (recommended) or merge to `main`  
**Deploy:** production environment with change record

**Reasoning:**
- Tag-based releases are easier to audit and roll back
- Separates “code merged” from “code released,” which enterprises prefer

---

## 9) Post-Deploy Verification

### Actions
- Health check / smoke test:
  - Load homepage
  - Validate critical routes render
  - Confirm API base URL is reachable (where applicable)
- Optional synthetic monitoring or lighthouse budget checks

### Reasoning
- Catch misconfigured environment variables or CDN issues immediately
- Reduces time-to-detect for broken deployments
- Creates confidence that deployment succeeded beyond “build finished”

---

## Versioning & Release Process

### Recommended: Semantic Versioning (SemVer)
- `MAJOR.MINOR.PATCH`
- Tag releases: `v1.2.3`

**Reasoning:**
- Predictable versioning supports enterprise change management
- Enables clean rollbacks and structured release notes

### Release Notes
- Each production release includes:
  - Summary of changes
  - Risk notes (breaking changes, migrations)
  - Links to PRs / tickets

**Reasoning:**
- Reduces operational surprises for customers
- Provides auditable change documentation

---

## Required GitHub Repository Settings (Enforcement)

### Branch Protection Rules for `main`
- Require PR before merging
- Require approvals (>= 1)
- Require status checks:
  - lint
  - typecheck
  - test
  - build
  - security scan
- Require conversation resolution
- Restrict who can push
- Disallow force pushes

**Reasoning:**
- Prevents bypassing CI gates
- Enforces consistent SDLC controls
- Supports enterprise and SOC 2-aligned change management evidence

---

## CI/CD Evidence & Audit Readiness (SOC 2 Friendly)

Maintain the following evidence in GitHub:
- PR history with approvals
- CI runs and logs
- Branch protection settings
- Release tags and artifacts
- Vulnerability scanning results
- Incident / exception tickets (for any bypasses)

**Reasoning:**
- Auditors want proof that controls are consistently followed, not just “we say we do it.”

---

## Operational Policies (Recommended)

### Hotfix Policy
- Hotfix branches: `hotfix/*`
- PR required, expedited review allowed
- Post-incident review required if a shortcut was taken

**Reasoning:**
- Enables rapid response without permanently weakening controls

### Exception Policy
If a security scan or check must be bypassed:
- Requires a ticket
- Requires explicit approval by a designated approver
- Requires a follow-up remediation plan

**Reasoning:**
- Makes exceptions visible and accountable
- Prevents “temporary” exceptions from becoming permanent risk

---

## Summary

This CI/CD strategy ensures:
- Every change is reviewed and traceable
- Automated checks prevent obvious defects from shipping
- Security and supply-chain risks are continuously evaluated
- Deployments produce signed, identifiable artifacts for rollback and audit trails
- The process scales from small team to enterprise expectations without rework
