![logo_ironhack_blue 7](https://user-images.githubusercontent.com/23629340/40541063-a07a0a8a-601a-11e8-91b5-2f13e4e6b441.png)

# Lab | CI/CD & Monitoring for ML

## Overview

You will wire up the operational machinery for a model service: a GitHub Actions pipeline that gates, builds, and canaries; an alerting spec that wakes the right person at the right time; and a rollback runbook the on-call engineer could execute at 3 a.m.

This is a 90-minute lab. You'll write YAML and Markdown. No code, no training. The work products are real artifacts a team could land in a sprint.

## Learning Goals

By the end of this lab you should be able to:

- Author a multi-stage CI/CD pipeline in GitHub Actions YAML with explicit gates
- Specify multi-window burn-rate alerts tied to SLOs
- Write a rollback runbook concise enough to execute under pressure

## Setup

Fork and clone the repo. No runtime needed. You'll need a YAML editor and a Markdown editor.

For a quick validity check on the workflow YAML you can use:

- [actionlint](https://github.com/rhysd/actionlint) — `actionlint .github/workflows/deploy-model.yml`
- Or paste into the GitHub Actions visual editor in your repo

## The Scenario

Continuing the **Vision Moderation** thread. You inherit:

- A repo with the OpenAPI spec, a multi-stage Dockerfile, and a small set of `./scripts/` helpers (`test-unit.sh`, `test-contract.sh`, `smoke.sh`, `deploy.sh`, `canary-verify.sh`, `rollback.sh`). The scripts are provided as black boxes — you only call them, you don't read or modify them.
- A staging environment and a production environment registered in GitHub Environments.
- Prometheus + Alertmanager in production; alerts route to PagerDuty (page) or Linear (ticket).
- The SLOs you wrote in yesterday's lab.

## Tasks

### Task 1 — CI/CD pipeline

Create `.github/workflows/deploy-model.yml` defining the full pipeline. It must include these stages, in order, each as its own job that depends on the previous:

1. **`lint`** — lint the OpenAPI spec and the Dockerfile (use `hadolint` for the Dockerfile)
2. **`test`** — unit tests + contract tests (call the provided scripts)
3. **`build`** — build the image, tag it `ghcr.io/<repo>:<git-sha>`
4. **`scan`** — Trivy scan; **fail on HIGH or CRITICAL** vulnerabilities
5. **`push`** — push the image to GHCR
6. **`deploy-staging`** — apply manifests to staging; only on push to `main`
7. **`smoke-staging`** — run the smoke script against staging
8. **`deploy-production`** — uses GitHub `environment: production` with required approvers; canary rollout at 10%, then verify, then full rollout

Requirements:

- Use `needs:` to enforce ordering.
- Pass the git SHA as the image tag through every step that needs it.
- Pin every third-party action to a major version tag at minimum (`@v4`), ideally a SHA (`@<full-sha>`).
- Use `secrets.GHCR_TOKEN` (or the built-in `GITHUB_TOKEN`) for registry auth — never hardcode credentials.
- A reusable `concurrency:` block so a new push cancels an in-flight pipeline for the same ref.

Your workflow must validate with `actionlint` (or pass the GitHub editor validation).

### Task 2 — Alert rules

Create `monitoring/alerts.yaml` defining at least these alerts as Prometheus AlertManager rules (or in a similar declarative format if you prefer). Each alert must include name, severity, the SLI expression, the threshold, the window, and a runbook URL placeholder.

**Required alerts:**

1. **`AvailabilityBurnFast`** — multi-window burn rate, fast: 2% of monthly error budget consumed in 1 hour → page
2. **`AvailabilityBurnSlow`** — multi-window burn rate, slow: 10% of monthly error budget consumed in 24 hours → ticket
3. **`LatencyP99High`** — p99 latency > 1 second for 5 minutes → page
4. **`DriftDetected`** — input-feature drift score (PSI) > 0.2 for any major feature, sustained 1 hour → ticket
5. **`ModelVersionMismatch`** — at least one replica is serving an unexpected model_version (i.e. registry says vX, response header says vY) for 5 minutes → page

Each rule must look something like:

```yaml
- alert: AvailabilityBurnFast
  expr: |
    (
      sum(rate(http_requests_total{job="vision-moderation",code=~"5.."}[1h]))
      /
      sum(rate(http_requests_total{job="vision-moderation"}[1h]))
    ) > (14.4 * (1 - 0.995))
  for: 5m
  labels:
    severity: page
  annotations:
    summary: "Availability burn rate is consuming budget too fast"
    runbook: "https://runbooks.example.com/vision-moderation/availability-burn"
```

Use real PromQL-style expressions. Hand-wavy `latency > X` strings fail the bar.

### Task 3 — Rollback runbook

Write `runbooks/rollback.md`. Keep it under one page. It must answer:

- **When to roll back** — explicit, measurable triggers (e.g. error rate, latency, quality proxy)
- **How to roll back** — exact commands or workflow_dispatch invocations
- **What to verify** — which dashboards/alerts must return to green
- **Who to notify** — channels and roles
- **What not to do** — the easy-to-make mistakes (don't roll forward before root cause)
- **When to roll forward** — the conditions under which you re-deploy

The whole document should fit on one screen of a phone. That is the bar.

## Submission

Open a PR to the lab repo with:

```
.github/workflows/deploy-model.yml
monitoring/alerts.yaml
runbooks/rollback.md
```

Paste the PR link as your deliverable. Include in the PR description either the `actionlint` output (passing) or a screenshot of the GitHub workflow validation.

## Quality bar

You will be reviewed on:

- **Does every job have a `needs:` predecessor**, or are they parallel by accident?
- **Does the image scan actually fail the build** on high/critical CVEs?
- **Are the alerts multi-window burn-rate**, or single-threshold?
- **Does each alert link to a runbook URL**, even a placeholder?
- **Is the rollback runbook a *checklist*** or a *narrative*? Checklists win.
