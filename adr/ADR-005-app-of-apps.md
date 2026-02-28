# ADR-005: ArgoCD App of Apps Pattern

**Status:** Accepted
**Date:** 2024-01-01

## Context

The cluster runs 18+ applications across multiple namespaces: infrastructure
components (cert-manager, ingress-nginx, aws-lbc, ESO, EFS, CA), monitoring
(Prometheus, Grafana, Loki), security (Kyverno, Falco, policy-controller),
developer tooling (Jenkins, SonarQube, ArgoCD itself), and the application
workload (borod-bank, argo-rollouts, chaos-mesh, kubecost, AIOps).

Managing each as an independent ArgoCD Application that must be manually applied
after every cluster recreate does not scale and is error-prone.

## Decision

Use the App of Apps pattern: a single root ArgoCD Application (`borod-bank-app-of-apps`)
points to `borod-bank-cd/argocd/apps/`, which contains one Application manifest
per component. ArgoCD syncs the root app, which in turn creates and syncs all
child apps. Sync waves (`argocd.argoproj.io/sync-wave`) control ordering:
- Wave -2: Kyverno (policy engine must precede all workloads)
- Wave -1: cert-manager, ingress-nginx, ESO, Loki (infrastructure before apps)
- Wave 0: everything else

## Consequences

**Positive:**
- Bootstrapping a fresh cluster requires exactly one `kubectl apply` (the root app).
  ArgoCD handles the rest — namespace creation, Helm chart installation, secret
  sync, and rollout — in dependency order.
- Adding a new service means adding one YAML file in `argocd/apps/` and opening
  a PR. No manual ArgoCD CLI or UI steps.
- The entire desired cluster state is in Git — drift is detected and corrected
  automatically via `selfHeal: true`.

**Negative:**
- `directory.recurse: true` is required on the root app — missing it silently
  skips subdirectories (e.g. monitoring/).
- CRDs larger than 262 KB cannot be applied via client-side `kubectl apply` —
  must use `ServerSideApply=true` in syncOptions (affects chaos-mesh, Kyverno).
- Sync waves only control creation order, not readiness. If a Wave -1 component
  is slow to become Ready, Wave 0 components may start before their dependencies
  are available — mitigated by ArgoCD health checks and resource hooks where needed.
