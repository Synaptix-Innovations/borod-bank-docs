# ADR-004: Kaniko over Docker-in-Docker for Container Builds

**Status:** Accepted
**Date:** 2024-01-01

## Context

Jenkins builds run in ephemeral Kubernetes pods (pod agent pattern). Building and
pushing Docker images from inside a pod requires access to a container runtime.

The two standard approaches:
1. Docker-in-Docker (DinD) — run a Docker daemon inside the build pod with
   `privileged: true`. Gives the pod full root access to the host node.
2. Kaniko — a Google-built tool that builds OCI images from a Dockerfile by
   unpacking layers in user space, without a daemon and without `privileged` mode.

## Decision

Use Kaniko (`gcr.io/kaniko-project/executor`) as a sidecar container in the
Jenkins pod agent. The executor runs as UID 0 (root inside the container) but
without `allowPrivilegeEscalation` and without node-level privileges.

## Consequences

**Positive:**
- No `privileged: true` on build pods — aligns with the Architectural Manifest
  ("No privileged builds").
- Kaniko is Checkov-compliant out of the box — passes all IaC security checks
  that DinD fails.
- Layer caching is supported via `--cache` and `--cache-repo` flags pointing to
  Docker Hub — build times are comparable to DinD for cached layers.

**Negative:**
- Kaniko requires UID 0 inside the container — not fully rootless. The Kubernetes
  `allowPrivilegeEscalation: false` securityContext mitigates host escape risk.
- Some advanced Dockerfile features (e.g. `--mount=type=cache`) may behave
  differently in Kaniko than in standard Docker builds.
- Debugging failed builds is harder — no interactive Docker daemon to inspect
  intermediate layers with `docker run`.
