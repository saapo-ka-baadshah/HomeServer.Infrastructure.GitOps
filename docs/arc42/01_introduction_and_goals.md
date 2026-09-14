# Introduction and Goals

*Last verified against repository state: 2026-09-15.*

## Requirements Overview

This repository implements a **GitOps infrastructure** for a HomeServer Kubernetes cluster using **Flux v2.9.5**. Git is the single source of truth for cluster state; Flux controllers continuously reconcile the actual cluster state against the desired state declared in this repository. **Current state:** only the Flux bootstrap layer is deployed — no application workloads (see [Current Scope](#current-scope) below).

The functional requirements are defined in [specs/REQUIREMENTS.md](../../specs/REQUIREMENTS.md) §2 and summarized below.

| ID | Requirement | Short description |
|----|-------------|-------------------|
| REQ-001 | Bootstrap Flux Controllers | Deploy Flux v2.9.5 components (source-controller v1.9.5, kustomize-controller v1.9.5, helm-controller v1.6.4, notification-controller v1.9.4) into the `flux-system` namespace on each cluster. |
| REQ-002 | Configure GitRepository Source | Define a `GitRepository` (`flux-system`) watching the `main` branch of the Git remote via SSH with a 1-minute polling interval. |
| REQ-003 | Reconcile Cluster State from Git | Define a `Kustomization` (`flux-system`) applying `./clusters/<cluster-name>/` every 10 minutes with `prune: true`. |
| REQ-004 | Support Multiple Clusters | Independent Flux bootstrap configurations for `kind` and `homeserver-cluster`, each with its own sync path. |
| REQ-005 | Enforce Restricted Pod Security Standards | Carry `restricted` pod security warn labels on the `flux-system` namespace. |
| REQ-006 | Restrict Network Traffic via NetworkPolicies | Apply `allow-egress`, `allow-scraping` (8080/TCP) and `allow-webhooks` (9292/TCP) NetworkPolicies. |
| REQ-007 | Reserve Resources for Critical System Pods | Create the `critical-pods-flux-system` ResourceQuota (1000 pods, critical priority classes). |
| REQ-008 | Provide RBAC for Flux Controllers | Grant `cluster-admin` to the reconcilers, aggregated edit/view roles, and CRD management permissions. |
| REQ-009 | Use Kustomize for Manifest Composition | Compose `gotk-components.yaml` + `gotk-sync.yaml` per cluster via `kustomize.config.k8s.io/v1beta1`. |
| REQ-010 | Authenticate to Git via SSH Deploy Key | Reference a Kubernetes Secret (`flux-system`) containing an SSH private key for read access to GitHub. |

Full acceptance criteria and traceability are in [specs/REQUIREMENTS.md](../../specs/REQUIREMENTS.md) §5 and §7.

### Current Scope

Only the Flux bootstrap layer is deployed today. There are **no application workloads** (CON-010) and no example workload or tenant namespace structure to validate the end-to-end GitOps flow (GAP-003). See [Risks and Technical Debts](11_technical_risks.md) for the full gap list.

## Quality Goals

| Goal | Description | NFR ref | Priority |
|------|-------------|---------|----------|
| Declarative state management | All cluster state is declared in Git; no manual `kubectl apply` for managed resources. | NFR-001 | High |
| Drift detection & self-healing | Flux continuously reconciles and automatically corrects drift within ~10 minutes. | NFR-002 | High |
| Reproducibility | Any cluster can be recreated from scratch from the same Git commit. | NFR-003 | High |
| Minimal human access | Operations are performed via Git commits; direct cluster access is exceptional. | NFR-004 | Medium |
| Security hardening | SSH-based auth, network segmentation, pod security, RBAC and resource quotas. | NFR-005 to NFR-008 | High |
| Version pinning | Flux and component versions are pinned; upgrades are intentional Git commits. | NFR-009 | Medium |
| Auditability | All changes to cluster state are traceable via Git history. | NFR-010 | Medium |
| Environment separation | Kind (dev) and homeserver-cluster (prod) have independent Flux instances and reconciliation loops. | NFR-011 | Medium |

The **top three priorities** are **reliability** (self-healing reconciliation that keeps the cluster converged to Git), **security** (SSH-only Git access, network segmentation, pod security and least-privilege RBAC), and **reproducibility** (a fresh cluster can be bootstrapped to an identical state from the same commit).

## Stakeholders

| Role | Contact | Expectations |
|------|---------|--------------|
| Platform Engineer (Owner) | `saapo-ka-baadshah` (GitHub) | Reliable, reproducible GitOps pipeline; minimal manual cluster operations. |
| Developer (Workload Author) | — | Clear path to deploy applications via Git; self-service namespace model. |
| Security Auditor | — | SSH key management, network policies, pod security standards, audit trail. |
| Operator (On-call) | — | Observability into reconciliation health; alerting on drift and failures. |

Stakeholder expectations are defined in [specs/REQUIREMENTS.md](../../specs/REQUIREMENTS.md) §8.