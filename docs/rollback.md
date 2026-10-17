# Rollback

## TL;DR

```bash
# Find the last known-good digest
git log --oneline applications/wattzgoat/prod/values.yaml

# Revert to it
git revert <commit-sha>
git push origin main

# Argo CD auto-syncs within 3 minutes
```

## Three Rollback Scenarios

### Scenario 1: Bad Image (most common)

**Symptom:** New version deployed, errors spike, health checks failing.

**Fix:** Revert the digest.

```bash
# On main branch
git checkout main
git pull

# Find the commit that changed the digest
git log -p applications/wattzgoat/prod/values.yaml | head -50

# Revert it
git revert <bad-commit-sha>
git push origin main
```

Argo CD detects the change, syncs, and Kubernetes rolls back the Deployment
using its normal rolling update strategy.

**Time to recover:** ~3–5 minutes.

---

### Scenario 2: Bad Config Change

**Symptom:** Pods start, but behave incorrectly (wrong env vars, wrong limits).

**Fix:** Revert the specific file.

```bash
git revert <commit-sha> -- applications/wattzgoat/prod/patches.yaml
git push origin main
```

If the change was in a Kyverno policy, revert in `platform-policies` instead.

---

### Scenario 3: Emergency Rollback (Argo CD down)

**Symptom:** GitOps is broken, no sync happening, production is bleeding.

**Fix:** Manual `kubectl` rollback — then reconcile Git.

```bash
# Roll back the deployment immediately
kubectl rollout undo deployment/wattzgoat -n wattzgoat-prod

# Or roll back to a specific revision
kubectl rollout history deployment/wattzgoat -n wattzgoat-prod
kubectl rollout undo deployment/wattzgoat -n wattzgoat-prod --to-revision=3

# Verify
kubectl rollout status deployment/wattzgoat -n wattzgoat-prod
```

**Critical:** After the emergency rollback, you **must** update Git to match
the cluster state, or Argo CD will immediately re-deploy the bad version.

```bash
# Update values.yaml to the rolled-back digest
yq -i '.image.digest = "<old-digest>"' \
  applications/wattzgoat/prod/values.yaml

git commit -am "hotfix(wattzgoat): emergency rollback to <old-digest>"
git push origin main
```

---

## Rollback Decision Tree

```
Is production down or degraded?
│
├── Yes → Is Argo CD syncing normally?
│         │
│         ├── Yes → Revert the Git commit (Scenario 1 or 2)
│         │
│         └── No  → Emergency kubectl rollback (Scenario 3)
│                   Then reconcile Git
│
└── No → Investigate. Do not roll back preemptively.
```

## What Rollback Does NOT Fix

- **Data migrations** — if the new version ran a schema migration, rolling
  back the app may leave it incompatible with the DB. Always test migrations
  in staging first.
- **Secrets rotation** — if a secret changed, rollback doesn't restore it.
- **Kyverno policies** — a bad policy can block rollback. Revert the policy
  first in `platform-policies`.

## Post-Rollback Checklist

- [ ] Cluster state matches Git
- [ ] Incident ticket opened
- [ ] Root cause identified (not just "it was broken")
- [ ] Fix prepared in dev, not hot-patched in prod
- [ ] Post-mortem scheduled within 48 hours
- [ ] Error budget impact documented