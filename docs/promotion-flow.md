# Promotion Flow: Dev → Staging → Prod

## Overview

Promotion is **PR-driven**, not automated. A human reviews every environment
transition. This is deliberate — it preserves the audit trail and prevents
accidental production changes.

## The Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. App CI builds and signs image                                │
│    └── platform-ci pushes to ECR with digest                    │
│    └── cosign signs the digest                                  │
│    └── CI opens PR against platform-gitops for dev              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Dev PR auto-merges (or platform reviews)                     │
│    └── Argo CD syncs to wattzgoat-dev                           │
│    └── Smoke tests run against dev                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼  (manual promotion)
┌─────────────────────────────────────────────────────────────────┐
│ 3. Promote to staging                                           │
│    └── Open PR: copy dev digest → staging/values.yaml           │
│    └── Platform review + merge                                  │
│    └── Argo CD syncs to wattzgoat-staging                       │
│    └── Integration + DAST tests run                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼  (manual promotion)
┌─────────────────────────────────────────────────────────────────┐
│ 4. Promote to prod                                              │
│    └── Open PR: copy staging digest → prod/values.yaml          │
│    └── Two platform reviewers required                          │
│    └── Merge triggers Argo CD sync to wattzgoat-prod            │
│    └── Post-sync health check gates the deployment              │
└─────────────────────────────────────────────────────────────────┘
```

## Promotion Commands

### Dev → Staging

```bash
# Extract the digest currently deployed in dev
DIGEST=$(yq '.image.digest' \
  applications/wattzgoat/dev/values.yaml)

# Create a promotion branch
git checkout -b promote/wattzgoat-staging-$(date +%Y%m%d)

# Update staging values
yq -i ".image.digest = \"$DIGEST\"" \
  applications/wattzgoat/staging/values.yaml

git commit -am "promote(wattzgoat): $DIGEST to staging"
git push origin HEAD

# Open PR
gh pr create \
  --title "Promote wattzgoat to staging" \
  --body "Promoting digest \`$DIGEST\` from dev to staging."
```

### Staging → Prod

Same pattern, but:
- **Two required reviewers** (enforced by CODEOWNERS + branch protection)
- **Change window** — no promotions on Fridays or during freeze periods
- **Rollback plan** — must be documented in the PR description

## What Gets Promoted

**Only the image digest.** Never:
- Environment-specific values (replicas, resources)
- Namespace names
- Secrets

Environment differences live in each env's `values.yaml` and `patches.yaml`.
The digest is the single source of truth for "what code is running."

## Rules

| Rule | Enforced By |
|------|-------------|
| No direct push to `main` | Branch protection |
| Two reviewers for prod PRs | CODEOWNERS |
| Image must be signed | Kyverno `verify-signatures` |
| Image must have SLSA provenance | Kyverno `verify-slsa-provenance` |
| Digest must match source env | CI validation script |
| No promotion on Fridays | Process (not enforced) |

## Anti-Patterns

- ❌ Promoting `:latest` — always promote by digest
- ❌ Copying env values between environments — only the digest moves
- ❌ Skipping staging — dev → prod directly is forbidden
- ❌ Force-merging around CI — if CI fails, fix the underlying issue
