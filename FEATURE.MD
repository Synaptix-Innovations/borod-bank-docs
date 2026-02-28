# Borod Bank — Feature Roadmap & Pipeline Enhancements

---

## Current Stack Overview

| Component | Technology |
|---|---|
| Backend | Java 17, Spring Boot 3.4.5 |
| Database | AWS RDS MySQL 8.4 |
| Frontend | Thymeleaf + Bootstrap 4 |
| Container | Docker (Hub: vlad7774) |
| CI | GitHub Actions (PR checks) + Jenkins in EKS |
| CD | ArgoCD (App of Apps pattern) |
| Infrastructure | Terraform + AWS EKS (us-east-1) |
| Load Balancer | AWS NLB via AWS Load Balancer Controller |
| Ingress | NGINX + cert-manager (Let's Encrypt) |
| Secrets | External Secrets Operator + AWS Secrets Manager |
| Monitoring | Prometheus + Grafana |
| Code Quality | SonarQube |
| Security | Gitleaks, Trivy, Checkov, Syft (SBOM), Cosign, KMS |

---

## Current CI/CD Pipeline Flow

```
PR opened  →  GitHub Actions: lint + test + security  (<5min)
Merge main →  Jenkins: full pipeline → Docker push → update borod-bank-cd
ArgoCD     →  auto-sync borod-bank-cd → EKS (App of Apps)
```

**Jenkins CI Pipeline (Optimized with Parallel Stages):**
```
Checkout
  → [PARALLEL] Gitleaks + Hadolint + Trivy FS + Integration Tests
  → Compile
  → [PARALLEL] Unit Tests + SonarQube Analysis
  → Quality Gate (webhook callback, 5min timeout)
  → Build Package → Docker Build & Push
  → [PARALLEL] Trivy Image + Checkov IaC + Generate SBOM
  → Sign & Attest (Cosign)
  → Update K8s Manifests
  → Smoke Test (health + E2E Login: register → login → /dashboard)
  → Slack Notification
```

**Nightly Pipeline (12 stages, Build #8 SUCCESS 341s):**
```
Checkout
  → Unit Tests + Coverage (JaCoCo 70%)
  → Integration Tests (MySQL sidecar)
  → Architecture Tests (ArchUnit, 6 rules)
  → Mutation Testing (PIT 60%)
  → OWASP Dependency Check (skipped — too slow without NVD API key)
  → License Compliance (GPL/AGPL check)
  → Performance Test (k6: 10 VUs, 30s, internal cluster URL)
  → AI Nightly Analysis (Claude API Haiku)
  → Smoke Tests + E2E Login (conditional — app reachable)
  → DAST Scan / ZAP (conditional — app reachable)
  → Push Metrics (17 metrics → Pushgateway → Prometheus → Grafana)
```

**ArgoCD:**
```
auto-sync borod-bank-cd → EKS
  → Argo Rollouts: canary deploy (10→50→100%)
  → AnalysisTemplate: Prometheus error-rate check (auto-rollback if > 5%)
```

---

## Infrastructure Hardening

| # | Feature | Status | Notes |
|---|---------|--------|-------|
| 009 | Loki + Promtail (S3 storage) | ✅ Ready | Manifests ready. Runtime fix applied (emptyDir). Needs `terraform apply` for S3/IRSA |
| 017 | Kyverno (3 policies) | ✅ Deployed | In ArgoCD (18 apps). Cleanup job fix: bitnami/kubectl:1.31 |
| 021 | PDB (app, jenkins, sonarqube) | ✅ Deployed | In ArgoCD via app manifests |
| 022 | HPA (min 2, max 6) | ✅ Ready | hpa.yaml + rollout.yml updated. Activates on deploy |

---

## Advanced Security & AIOps

| # | Feature | Type | Cost | Status |
|---|---------|------|------|--------|
| 031 | Chaos Engineering (Chaos Mesh) | Reliability | $0 | ✅ Done (v3.7-chaos-mesh) |
| 032 | Runtime Security (Falco) | Security | $0 | ✅ Ready (falco.yml) |
| 033 | DAST (OWASP ZAP) | Security | $0 | ✅ Ready (Jenkinsfile.nightly) |
| 034 | Sigstore Policy Controller | Security | $0 | ✅ Ready (policy-controller.yml) |
| 035 | ArgoCD Drift Detection | GitOps | $0 | ✅ Ready (notifications-configmap) |
| 041 | AIOps Phase 1 — Log Intelligence | AIOps | $0 | ✅ Done (v3.8-aiops-prep, v3.9-aiops-analyzer) |
| 042 | Cost Optimization (Kubecost) | FinOps | $0 | ✅ Ready (kubecost.yml) |
| 043 | AIOps Phase 2 — Real-time Intelligence | AIOps | ~$10/мес | ✅ Done (v3.12-aiops-phase2) |

---

## Skipped Features (Overkill / Too Expensive)

### FEATURE-023: Preview Environments per PR
**Status:** ❌ Skipped
**Why:** Каждый PR требует LoadBalancer ($18/мес) + RDS ($37/мес). При 5 PR: $275/месяц.
**Alternative:** Локальное тестирование с `docker-compose.yml`.

### FEATURE-024: Blue/Green Deployments
**Status:** ❌ Skipped
**Why:** Требует X2 ресурсов. У нас уже есть Canary (Argo Rollouts) — современнее и экономичнее.

### FEATURE-025: Secret Rotation
**Status:** ❌ Skipped (for now)
**Why:** Ротация RDS паролей требует сложной Lambda. Manual rotation раз в квартал достаточно.

### FEATURE-026: Distributed Tracing (Tempo/Jaeger)
**Status:** ❌ Skipped (for now)
**Why:** Tracing нужен для микросервисов (10+ сервисов). У нас монолит — логи + метрики достаточно.

### FEATURE-027: Terratest
**Status:** ❌ Skipped
**Why:** Для статической инфраструктуры — overkill. `terraform validate` + `tflint` + `checkov` достаточно.

---

## Architecture Comparison

| Component | Borod Bank | Typical Pet Project | Enterprise |
|---|---|---|---|
| Database | ✅ RDS MySQL | ❌ MySQL в поде | ✅ RDS/Aurora |
| Secrets | ✅ AWS SM + ESO | ❌ kubectl create secret | ✅ Vault/SM |
| Terraform State | ✅ S3 + DynamoDB | ❌ Local | ✅ S3/TF Cloud |
| GitOps | ✅ ArgoCD App of Apps | ❌ kubectl apply | ✅ ArgoCD/Flux |
| Security Scanning | ✅ 6 tools | ⚠️ 1-2 | ✅ 5+ |
| Supply Chain | ✅ SBOM + Cosign | ❌ None | ✅ SBOM + Signing |
| Canary Deploys | ✅ Argo Rollouts | ❌ None | ✅ Spinnaker/Flagger |
| Observability | ✅ Prometheus + Grafana (5 dashboards, 17 nightly metrics) | ⚠️ Logs only | ✅ Full stack |
| Centralized Logs | ✅ Loki Ready (S3 pending) | ❌ None | ✅ ELK/Loki |
| Policy Enforcement | ✅ Kyverno Deployed | ❌ None | ✅ OPA/Kyverno |
| Autoscaling | ✅ HPA Ready | ❌ None | ✅ HPA + VPA |

**Current Status:** 85-90% Enterprise-grade (Kyverno + PDB deployed) → **After Loki + HPA deploy:** 95%
