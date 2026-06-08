# Vision Moderation — Rollback Runbook

---

## When to Roll Back
Roll back immediately if **any** of these are true post-deployment:

- [ ] `AvailabilityBurnFast` alert is firing (page received)
- [ ] `LatencyP99High` alert firing — p99 > 1s for > 3 minutes
- [ ] HTTP 5xx error rate > 1% in any 5-minute window
- [ ] `ModelVersionMismatch` alert firing for > 2 minutes
- [ ] Canary `canary-verify.sh` step exits non-zero in CI

---

## How to Roll Back

### Option A — GitHub Actions (Preferred)
1. **Actions** → select `CI/CD & Monitoring for ML` workflow
2. **Run workflow** → branch: `main`
3. Enter the last known good image SHA (check previous successful run)
4. Click **Run workflow** and monitor the `deploy-production` job

### Option B — CLI Break-Glass (Use if CI is unavailable)
```bash
# Roll back production to previous stable image
./scripts/rollback.sh production

# Confirm all pods are on stable image
kubectl rollout status deployment/vision-moderation -n vision-moderation
kubectl get pods -n vision-moderation \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

---

## What to Verify After Rollback

Check all of the following within **5 minutes** of rollback:

- [ ] **Grafana** → `Vision Moderation / SLO Overview` dashboard — error rate back below 0.5%
- [ ] **Grafana** → `Latency` panel — p99 back below 1s
- [ ] **Alertmanager** → `AvailabilityBurnFast` and `LatencyP99High` resolve (go green)
- [ ] **`ModelVersionMismatch`** alert resolves — all replicas on expected version
- [ ] Run smoke manually: `./scripts/smoke.sh production` — must exit 0

---

## Who to Notify

| When | Who | Channel |
|------|-----|---------|
| Rollback starts | On-call engineer | `#incidents` (Slack) |
| Rollback complete | Engineering lead | `#incidents` + `#vision-moderation` |
| If SLO breached | Product owner | Direct message |
| Post-incident | Whole team | Post-mortem ticket in Linear |

---

## What NOT to Do

- **Do NOT roll forward** before root cause is identified and confirmed
- **Do NOT restart pods manually** without running `rollback.sh` — partial restarts create mixed-version state
- **Do NOT close the incident** until all alerts resolve and smoke tests pass
- **Do NOT silence alerts** to avoid noise — resolve the underlying issue first
- **Do NOT assume** the previous image is safe without checking its CI run history

---

## When to Roll Forward

Re-deploy only when **all** of the following are true:

- [ ] Root cause is identified and documented
- [ ] Fix is merged, reviewed, and CI passes (including Trivy scan)
- [ ] Staging smoke tests pass on the new image
- [ ] Engineering lead or on-call explicitly approves the production deploy
- [ ] Monitoring is actively watched during the new canary rollout