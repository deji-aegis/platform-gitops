# platform-gitops

Argo CD configuration and application deployment manifests for Project Aegis.

## Structure

- `bootstrap/` — one-time cluster bootstrap (Argo CD, Kyverno, Cilium, etc.)
- `argocd/` — Argo CD Applications, ApplicationSets, and AppProjects
- `applications/` — application deployment manifests (Kustomize overlays)
- `environments/` — per-environment namespaces, quotas, limits

## Ownership

Platform team. All changes require PR review. CI validates manifests before merge.

## Bootstrap Order

1. Apply `bootstrap/root-app.yaml` — this creates the app-of-apps
2. Argo CD syncs everything else from this repo
3. Never `kubectl apply` application manifests manually