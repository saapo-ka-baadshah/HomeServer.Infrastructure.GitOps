# Requirements Baseline Document

**Project:** HomeServer Infrastructure — GitOps  
**Repository:** ssh://git@github.com/saapo-ka-baadshah/HomeServer.Infrastructure.GitOps (branch: main)  
**Document Version:** 1.0  
**Date:** 2026-09-15  
**Purpose:** Formal requirements baseline to ground arc42 architecture documentation (docs/arc42/)

---

## 1. System Overview

This repository implements a **GitOps infrastructure** for managing a HomeServer Kubernetes cluster using **Flux v2 (v2.9.5)**. The system follows the GitOps principle: **Git is the single source of truth** for cluster state. Flux controllers continuously reconcile the actual cluster state against the desired state declared in this repository.

**Current State:** Only the Flux bootstrap layer is deployed (no application workloads). Two clusters are configured:
- `clusters/kind/` — Local development cluster (Kubernetes in Docker)
- `clusters/homeserver-cluster/` — Production HomeServer cluster

---

## 2. Functional Requirements

| ID | Requirement | Description | Source/Traceability |
|----|-------------|-------------|---------------------|
| **REQ-001** | **Bootstrap Flux Controllers** | The system shall deploy Flux v2.9.5 components (source-controller v1.9.5, kustomize-controller v1.9.5, helm-controller v1.6.4, notification-controller v1.9.4) into the `flux-system` namespace on each target cluster. | `gotk-components.yaml` (lines 3-4) |
| **REQ-002** | **Configure GitRepository Source** | The system shall define a `GitRepository` resource (`flux-system`) that watches the `main` branch of the Git remote `ssh://git@github.com/saapo-ka-baadshah/HomeServer.Infrastructure.GitOps` with a 1-minute polling interval, authenticating via SSH deploy key (secretRef: `flux-system`). | `gotk-sync.yaml` (lines 3-14) |
| **REQ-003** | **Reconcile Cluster State from Git (Kustomization)** | The system shall define a `Kustomization` resource (`flux-system`) that applies manifests from `./clusters/<cluster-name>/` with a 10-minute reconciliation interval and **prune enabled** (removes resources deleted from Git). | `gotk-sync.yaml` (lines 16-27) |
| **REQ-004** | **Support Multiple Clusters** | The repository structure shall support independent Flux bootstrap configurations for multiple clusters (at minimum: `kind` and `homeserver-cluster`), each with its own `gotk-sync.yaml` pointing to its cluster-specific path. | Directory structure: `clusters/kind/`, `clusters/homeserver-cluster/` |
| **REQ-005** | **Enforce Restricted Pod Security Standards** | The `flux-system` namespace shall carry pod security labels (`pod-security.kubernetes.io/warn: restricted`, `pod-security.kubernetes.io/warn-version: latest`) to warn on privileged pod admissions. | `gotk-components.yaml` (lines 12-13) |
| **REQ-006** | **Restrict Network Traffic via NetworkPolicies** | The system shall apply NetworkPolicies in `flux-system` namespace: (a) `allow-egress` — allow all egress, restrict ingress to same-namespace pods; (b) `allow-scraping` — allow ingress on port 8080/TCP from any namespace for metrics scraping; (c) `allow-webhooks` — allow ingress on port 9292/TCP from any namespace to notification-controller. | `gotk-components.yaml` (lines 16-76) |
| **REQ-007** | **Reserve Resources for Critical System Pods** | The system shall create a `ResourceQuota` (`critical-pods-flux-system`) limiting pods to 1000 for system-node-critical and system-cluster-critical priority classes. | `gotk-components.yaml` (lines 78-96) |
| **REQ-008** | **Provide RBAC for Flux Controllers** | The system shall create ClusterRoles and ClusterRoleBindings granting: (a) `cluster-admin` to kustomize-controller and helm-controller service accounts (full reconciliation rights); (b) aggregated `flux-edit` and `flux-view` roles for Flux CRDs; (c) CRD management permissions for controllers. | `gotk-components.yaml` (lines 98-299+) |
| **REQ-009** | **Use Kustomize for Manifest Composition** | The system shall use `kustomize.config.k8s.io/v1beta1` Kustomization resources to compose `gotk-components.yaml` + `gotk-sync.yaml` per cluster. | `kustomization.yaml` (both clusters) |
| **REQ-010** | **Authenticate to Git via SSH Deploy Key** | The GitRepository shall reference a Kubernetes Secret (`flux-system`) containing an SSH private key for read access to the GitHub repository. | `gotk-sync.yaml` (line 13) |

---

## 3. Non-Functional Requirements

| ID | Requirement | Description | Category |
|----|-------------|-------------|----------|
| **NFR-001** | **Declarative State Management** | All cluster state (Flux components, future workloads) must be declared in Git. No manual `kubectl apply` for managed resources. | Architecture |
| **NFR-002** | **Drift Detection & Self-Healing** | Flux shall continuously reconcile (poll every 1m for Git, 10m for Kustomization) and automatically correct drift (prune=true). | Reliability |
| **NFR-003** | **Reproducibility** | Any cluster can be recreated from scratch by applying the cluster's `kustomization.yaml` — same Git commit → same cluster state. | Reproducibility |
| **NFR-004** | **Minimal Human Access to Clusters** | Operations are performed via Git commits; direct cluster access (kubectl) is exceptional. Flux controllers run with cluster-admin but human operators should not need it. | Security |
| **NFR-005** | **SSH-Based Git Authentication** | Git access uses SSH deploy keys (not HTTPS tokens), stored as Kubernetes Secrets. Keys are cluster-scoped (each cluster has its own secretRef). | Security |
| **NFR-006** | **Network Segmentation** | NetworkPolicies restrict traffic to/from Flux controllers (ingress/egress controls, metrics scraping allowlist, webhook allowlist). | Security |
| **NFR-007** | **Pod Security Baseline** | Flux namespace enforces restricted pod security standard warnings (baseline for future workload namespaces). | Security |
| **NFR-008** | **Resource Quotas for System Stability** | ResourceQuota prevents runaway pods in flux-system from starving cluster-critical workloads. | Reliability |
| **NFR-009** | **Version Pinning** | Flux version (v2.9.5) and component versions are explicitly pinned in bootstrap manifests. Upgrades are intentional Git commits. | Maintainability |
| **NFR-010** | **Auditability** | All changes to cluster state are traceable via Git history (commits, authors, timestamps). | Auditability |
| **NFR-011** | **Separation of Environments** | Kind (dev) and homeserver-cluster (prod) have independent Flux instances, paths, and reconciliation loops — no cross-contamination. | Architecture |
| **NFR-012** | **No External Secret Management (Current)** | Secrets (SSH key) are stored as plain Kubernetes Secrets in the cluster. No external secret operator (e.g., External Secrets Operator, Sealed Secrets) is currently configured. | Security (Gap) |

---

## 4. Constraints

| ID | Constraint | Description |
|----|------------|-------------|
| **CON-001** | **Flux Version** | Flux v2.9.5 (components: source-controller v1.9.5, kustomize-controller v1.9.5, helm-controller v1.6.4, notification-controller v1.9.4). |
| **CON-002** | **Kubernetes API Versions** | Uses `source.toolkit.fluxcd.io/v1`, `kustomize.toolkit.fluxcd.io/v1`, `kustomize.config.k8s.io/v1beta1`, `networking.k8s.io/v1`, `rbac.authorization.k8s.io/v1`. |
| **CON-003** | **Kustomize API** | `kustomize.config.k8s.io/v1beta1` (not v1beta2 or v1alpha1). |
| **CON-004** | **Cluster Naming Convention** | Clusters live under `clusters/<cluster-name>/flux-system/` with `gotk-components.yaml`, `gotk-sync.yaml`, `kustomization.yaml`. |
| **CON-005** | **Git Branch** | Only `main` branch is tracked (ref.branch: main). No multi-branch strategy (e.g., dev/prod branches). |
| **CON-006** | **Git Hosting** | GitHub via SSH (`ssh://git@github.com/...`). |
| **CON-007** | **No Helm Charts Currently** | helm-controller is installed but no HelmRelease resources exist. |
| **CON-008** | **No Git Webhook Receiver** | Only polling-based reconciliation (1m GitRepository, 10m Kustomization). No `notification-controller` webhook endpoint exposed for push-based triggers. |
| **CON-009** | **Single Secret per Cluster** | One SSH deploy key secret (`flux-system`) per cluster for Git access. |
| **CON-010** | **No Application Workloads** | Repository currently contains only Flux bootstrap layer. |

---

## 5. Acceptance Criteria

### REQ-001: Bootstrap Flux Controllers
- [ ] `flux-system` namespace exists with correct labels (version v2.9.5)
- [ ] All 4 controllers (source, kustomize, helm, notification) are running as Deployments
- [ ] Controller images match pinned versions in `gotk-components.yaml`

### REQ-002: Configure GitRepository Source
- [ ] `GitRepository` resource `flux-system` exists in `flux-system` namespace
- [ ] `spec.url` = `ssh://git@github.com/saapo-ka-baadshah/HomeServer.Infrastructure.GitOps`
- [ ] `spec.ref.branch` = `main`
- [ ] `spec.interval` = `1m0s`
- [ ] `spec.secretRef.name` = `flux-system`
- [ ] GitRepository status shows `Ready=True` and correct artifact revision

### REQ-003: Reconcile Cluster State from Git
- [ ] `Kustomization` resource `flux-system` exists in `flux-system` namespace
- [ ] `spec.path` = `./clusters/<cluster-name>` (kind or homeserver-cluster)
- [ ] `spec.interval` = `10m0s`
- [ ] `spec.prune` = `true`
- [ ] Kustomization status shows `Ready=True` and applied resources match Git

### REQ-004: Support Multiple Clusters
- [ ] `clusters/kind/flux-system/kustomization.yaml` references `./clusters/kind`
- [ ] `clusters/homeserver-cluster/flux-system/kustomization.yaml` references `./clusters/homeserver-cluster`
- [ ] Both clusters can be bootstrapped independently via `flux bootstrap git`

### REQ-005: Enforce Restricted Pod Security Standards
- [ ] `flux-system` namespace has labels: `pod-security.kubernetes.io/warn: restricted`, `pod-security.kubernetes.io/warn-version: latest`

### REQ-006: Restrict Network Traffic via NetworkPolicies
- [ ] `allow-egress` NetworkPolicy exists: egress allow all, ingress from same namespace only
- [ ] `allow-scraping` NetworkPolicy exists: ingress port 8080/TCP from any namespace
- [ ] `allow-webhooks` NetworkPolicy exists: ingress port 9292/TCP from any namespace to notification-controller pods

### REQ-007: Reserve Resources for Critical System Pods
- [ ] `ResourceQuota` `critical-pods-flux-system` exists with `hard.pods: "1000"` scoped to system-node-critical and system-cluster-critical priority classes

### REQ-008: Provide RBAC for Flux Controllers
- [ ] ClusterRole `cluster-reconciler-flux-system` binds `cluster-admin` to kustomize-controller and helm-controller SAs
- [ ] ClusterRole `crd-controller-flux-system` grants CRD management to all three controller SAs
- [ ] Aggregated ClusterRoles `flux-edit-flux-system` and `flux-view-flux-system` exist

### REQ-009: Use Kustomize for Manifest Composition
- [ ] Each cluster's `kustomization.yaml` lists `gotk-components.yaml` and `gotk-sync.yaml` as resources
- [ ] `kustomize build clusters/<cluster-name>/flux-system` produces valid combined manifest

### REQ-010: Authenticate to Git via SSH Deploy Key
- [ ] Secret `flux-system` exists in `flux-system` namespace with `identity` (private key) and `known_hosts` keys
- [ ] GitRepository can successfully fetch from the SSH URL

### NFR-002: Drift Detection & Self-Healing
- [ ] Manual modification of a Flux-managed resource is reverted within 10 minutes (Kustomization interval)
- [ ] Deletion of a resource from Git results in resource removal from cluster (prune=true)

### NFR-003: Reproducibility
- [ ] Fresh cluster + `flux bootstrap git` with this repo → identical Flux deployment
- [ ] Same Git commit on two clusters → same Flux component versions and configuration

---

## 6. Known Gaps / Open Questions

| ID | Gap / Question | Impact | Priority |
|----|----------------|--------|----------|
| **GAP-001** | **No Git Webhook Receiver** — Only polling (1m/10m). Push-based reconciliation not configured. | Latency up to 10m for changes to apply. | Medium |
| **GAP-002** | **No Backup/Disaster Recovery Strategy** — No etcd backup, no Git repo backup, no cluster restore procedure documented. | Risk of data loss. | High |
| **GAP-003** | **No Application Workloads** — Repository only contains Flux bootstrap. No example workload, no tenant namespace structure. | Blocks validation of end-to-end GitOps flow. | Medium |
| **GAP-004** | **No Observability Configuration** — No Prometheus ServiceMonitors, no Grafana dashboards, no alerting rules for Flux controllers. | Cannot monitor reconciliation health. | Medium |
| **GAP-005** | **No External Secret Management** — SSH deploy key stored as plain Kubernetes Secret. No Sealed Secrets, External Secrets Operator, or SOPS. | Secret rotation difficult; key visible in cluster. | High |
| **GAP-006** | **Single Branch Strategy** — Only `main` branch. No promotion flow (dev → staging → prod) via Git branches. | All changes go directly to production cluster. | Medium |
| **GAP-007** | **No Multi-Tenancy / Namespace Strategy** — No defined namespace structure for workloads, no NetworkPolicy defaults for tenant namespaces. | Security/isolation gaps when workloads added. | Medium |
| **GAP-008** | **No Flux Upgrade Procedure** — Version pinned to v2.9.5. No documented process for upgrading Flux components. | Risk of version skew, missed security patches. | Low |
| **GAP-009** | **`temp/` Directory Empty** — Placeholder with no documented purpose. | Confusion; potential for misuse. | Low |
| **GAP-010** | **No CI/CD Pipeline** — No GitHub Actions, GitLab CI, or other pipeline to validate manifests (kustomize build, kubeconform, flux lint) before merge. | Broken manifests can be merged. | High |
| **GAP-011** | **No Documentation for Bootstrap Procedure** — README only links to arc42. No `flux bootstrap git` command documented. | Onboarding friction. | Medium |
| **GAP-012** | **Notification Controller Configured but Unused** — Webhook NetworkPolicy exists but no alerting/notification receivers defined. | Dead code / unused component. | Low |
| **GAP-013** | **Kind Cluster Config Not in Repo** — Kind cluster configuration (if any) not versioned. Dev environment not reproducible from repo alone. | Dev environment drift. | Medium |
| **GAP-014** | **No Policy Enforcement (OPA/Gatekeeper/Kyverno)** — No admission control for workload policies (e.g., require limits, non-root, read-only rootfs). | Workloads can violate security baseline. | Medium |

---

## 7. Traceability Matrix

| Requirement | Implementation Artifact | Verification Method |
|-------------|------------------------|---------------------|
| REQ-001 | `clusters/*/flux-system/gotk-components.yaml` | `kubectl get deployments -n flux-system` |
| REQ-002 | `clusters/*/flux-system/gotk-sync.yaml` (GitRepository) | `flux get sources git -n flux-system` |
| REQ-003 | `clusters/*/flux-system/gotk-sync.yaml` (Kustomization) | `flux get kustomizations -n flux-system` |
| REQ-004 | Directory structure `clusters/kind/`, `clusters/homeserver-cluster/` | `ls clusters/` |
| REQ-005 | `gotk-components.yaml` Namespace labels | `kubectl get ns flux-system --show-labels` |
| REQ-006 | `gotk-components.yaml` NetworkPolicies | `kubectl get netpol -n flux-system` |
| REQ-007 | `gotk-components.yaml` ResourceQuota | `kubectl get quota -n flux-system` |
| REQ-008 | `gotk-components.yaml` ClusterRoles/Bindings | `kubectl get clusterrole,clusterrolebinding -l app.kubernetes.io/part-of=flux` |
| REQ-009 | `clusters/*/flux-system/kustomization.yaml` | `kustomize build clusters/<cluster>/flux-system` |
| REQ-010 | `gotk-sync.yaml` secretRef + Secret in cluster | `kubectl get secret flux-system -n flux-system` |
| NFR-002 | Kustomization `interval` + `prune` | Manual drift test |
| NFR-003 | Full bootstrap procedure | Fresh cluster bootstrap test |
| NFR-006 | NetworkPolicies | `kubectl get netpol -n flux-system -o yaml` |
| NFR-009 | `gotk-components.yaml` header comment | Version check in manifests |

---

## 8. Stakeholders

| Role | Expectation |
|------|-------------|
| Platform Engineer (Owner) | Reliable, reproducible GitOps pipeline; minimal manual cluster ops |
| Developer (Workload Author) | Clear path to deploy apps via Git; self-service namespace model |
| Security Auditor | SSH key management, network policies, pod security standards, audit trail |
| Operator (On-call) | Observability into reconciliation health; alerting on drift/failures |

---

## 9. Assumptions

1. Target clusters run Kubernetes version compatible with Flux v2.9.5 (Kubernetes 1.26+ recommended).
2. SSH deploy keys have been generated and added to GitHub repository as deploy keys (read-only).
3. `flux` CLI is available for bootstrap operations.
4. Kind cluster is used only for local development/testing.
5. Production cluster (`homeserver-cluster`) is a long-lived, persistent cluster.
6. No existing Flux installation on target clusters (clean bootstrap).

---

## 10. References

- Flux v2.9.5 Documentation: https://fluxcd.io/flux/v2.9.5/
- GitOps Principles: https://opengitops.dev/
- arc42 Template v9.0: https://arc42.org/
- Repository: ssh://git@github.com/saapo-ka-baadshah/HomeServer.Infrastructure.GitOps

---

*End of Requirements Baseline Document*