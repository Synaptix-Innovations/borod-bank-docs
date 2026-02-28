# ADR-006: Claude API over Local LLM for AIOps

**Status:** Accepted
**Date:** 2025-02-01

## Context

The platform includes an AIOps module that performs two AI-powered tasks:
1. **Code review** — analyzes code diffs on every CI build and nightly regression metrics
2. **Log analysis** — queries Loki for ERROR logs daily, deduplicates patterns, and
   generates root cause analysis with actionable recommendations sent to Slack

The initial implementation used **Ollama** running the **qwen2.5-coder:7b** model on a
dedicated GPU SPOT node (g4dn.xlarge). This required a separate EKS node group with
NVIDIA device plugin, GPU drivers, and a tainted node to isolate GPU workloads.

Two options were evaluated:
1. **Local LLM (Ollama)** — self-hosted 7B model on a GPU node inside the cluster
2. **Claude API (Haiku)** — external API call to Anthropic's Claude Haiku model

## Decision

Replace Ollama with **Claude API (Haiku 4.5)** accessed through a lightweight FastAPI
gateway running on standard CPU nodes. The API key is stored in AWS Secrets Manager
and injected via External Secrets Operator.

## Consequences

**Positive:**
- **60x cost reduction** — GPU SPOT node costs ~$120/month even at minimum usage
  (instance must be running to serve requests). Claude API Haiku costs ~$2/month for
  nightly-only usage with small payloads (4KB diffs, 500-token responses).
- **Higher review quality** — the 7B parameter local model produces generic,
  surface-level feedback ("consider adding error handling"). Claude Haiku identifies
  real logic bugs, security issues, and provides specific, actionable recommendations.
- **Zero GPU ops overhead** — eliminates NVIDIA device plugin, CUDA driver
  compatibility, model download/update management, GPU memory OOM debugging, and
  tainted node scheduling complexity.
- **No cold start** — Ollama on SPOT had 2-5 minute cold starts when the node was
  scaled down. Claude API responds in 1-3 seconds consistently.
- **Simplified infrastructure** — removed `aiops.tf` GPU node group, NVIDIA device
  plugin ArgoCD app, and GPU-related Terraform variables. The gateway runs on a
  standard `python:3.12-slim` container with 50m CPU.

**Negative:**
- **External dependency** — API availability depends on Anthropic's infrastructure.
  Mitigated by wrapping all AI stages in `catchError` — gateway downtime does not
  break CI/CD pipelines.
- **Data leaves the cluster** — source code diffs are sent to an external API.
  Mitigated by: (a) only diffs are sent, never full source, secrets, or env vars;
  (b) payloads are capped at 4KB; (c) Anthropic's API has a zero data retention
  policy — inputs are not stored or used for model training.
- **API key management** — requires storing and rotating an API key. Mitigated by
  AWS Secrets Manager with automatic sync via External Secrets Operator (1-hour
  refresh interval).
