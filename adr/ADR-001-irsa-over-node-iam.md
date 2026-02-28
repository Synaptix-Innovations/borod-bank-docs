# ADR-001: IRSA over Node-level IAM for AWS API Access

**Status:** Accepted
**Date:** 2024-01-01

## Context

Multiple cluster components need AWS API access: External Secrets Operator reads
from Secrets Manager, AWS Load Balancer Controller manages NLB resources, Cluster
Autoscaler scales node groups, EFS CSI driver mounts file systems, and Loki writes
logs to S3.

The two main options are:
1. Node IAM role — attach a broad policy to EC2 instance profile; all pods on that
   node share the same credentials.
2. IRSA (IAM Roles for Service Accounts) — bind an IAM role to a specific
   Kubernetes Service Account via the EKS OIDC provider; each workload gets its
   own scoped credentials.

## Decision

Use IRSA for every component that needs AWS access. Each service gets a dedicated
IAM role with only the permissions it requires.

## Consequences

**Positive:**
- Least-privilege by default — a compromised pod cannot access resources outside
  its own IAM role scope.
- Audit trail is per-role, not per-node — CloudTrail events are attributable to
  a specific workload.
- No credential rotation needed — IRSA tokens are short-lived (rotated by the
  EKS token webhook automatically).

**Negative:**
- Requires OIDC provider configuration per cluster — one-time Terraform setup.
- Every new service that needs AWS access requires a new IAM role and a
  `serviceAccountName` annotation — slightly more upfront work than adding
  a policy to the node role.
