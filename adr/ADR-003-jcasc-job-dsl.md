# ADR-003: Jenkins Configuration as Code + Job DSL

**Status:** Accepted
**Date:** 2024-01-01

## Context

Jenkins configuration (credentials, tools, security realm, job definitions) needs
to be version-controlled and reproducible. Without it, recreating Jenkins after a
cluster wipe requires manual steps that are error-prone and undocumented.

The alternatives considered:
1. Manual configuration — click through the Jenkins UI, export XML blobs.
2. JCasC (Jenkins Configuration as Code) — YAML-driven config loaded at startup.
3. JCasC + Job DSL — JCasC for Jenkins settings, Job DSL scripts for job
   definitions, both stored in the Docker image.

## Decision

Use JCasC (`casc.yml`) for all Jenkins system configuration and Job DSL
(embedded in `casc.yml` under `jobs:`) for job definitions. Both are baked into
the Jenkins Docker image at build time.

## Consequences

**Positive:**
- Jenkins is fully reproducible from `docker build` — no manual steps after
  initial `kubectl apply`.
- Configuration drift is impossible — every change must go through a PR in
  `jenkins-setup`.
- New jobs can be added in a PR without logging into Jenkins at all.

**Negative:**
- A YAML syntax error in `casc.yml` prevents Jenkins from starting — the image
  build CI catches most issues, but subtle DSL bugs may only surface at runtime.
- Two `unclassified:` blocks in the same YAML file silently overwrite each other
  — all unclassified settings must live in a single block.
- Secrets cannot be stored in `casc.yml` — they must be injected via environment
  variables sourced from External Secrets Operator at pod startup.
