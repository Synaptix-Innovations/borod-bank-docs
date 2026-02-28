# ADR-002: Argo Rollouts Canary over Standard RollingUpdate

**Status:** Accepted
**Date:** 2024-01-01

## Context

Deploying a banking application requires controlled traffic shifting with the
ability to automatically roll back on errors. Kubernetes built-in RollingUpdate
supports `maxSurge` and `maxUnavailable` but does not natively support:
- Percentage-based traffic splitting at the load-balancer level
- Automated analysis (metrics-based pass/fail)
- Pause-and-promote workflow for human approval

## Decision

Use Argo Rollouts with a canary strategy: 10% → 50% → 100%, with a 1-minute
pause between steps. Traffic is split via the NGINX ingress canary annotation.

## Consequences

**Positive:**
- Production traffic is split at ingress level, not at replica level — users
  on the stable version are unaffected until promotion.
- If the canary step fails a health check, Argo Rollouts rolls back automatically
  without human intervention.
- The rollout state is visible in the ArgoCD UI alongside the rest of the cluster.

**Negative:**
- Requires the `argo-rollouts` CRD and controller to be running — adds one more
  component to the control plane.
- `kubectl rollout` commands do not work on Rollout objects — need to use
  `kubectl argo rollouts` plugin or ArgoCD UI.
- An aborted rollout leaves the `status.abort` field set; must be cleared manually
  (`kubectl patch rollout ... --subresource=status`) before the next deploy.
