# Cross-cutting Concepts

*Last verified against repository state: 2026-09-15.*

## GitOps and Declarative State Management

**Concept.** Git is the single source of truth for the desired state of the cluster; all managed resources are declared declaratively in manifests (NFR-001).

**How it is implemented here.** The `GitRepository` and `Kustomization` resources in `gotk-sync.yaml` drive reconciliation from the `main` branch. There is no manual `kubectl apply` for managed resources.

**Related requirements.** NFR-001, REQ-002, REQ-003.

**Current gap.** None.

## Pull-Based Reconciliation and Drift Correction

**Concept.** Flux pulls the desired state from Git and continuously corrects drift.

**How it is implemented here.** source-controller polls Git every 1m; kustomize-controller reconciles every 10m with `prune: true`. Resources edited or deleted out-of-band are reverted or pruned at the next reconcile (NFR-002).

**Current gap.** There is no push channel — a webhook Receiver is not configured, so the maximum latency for a change is ~10m (GAP-001).

## Secrets Management (SSH Deploy Keys)

**Concept.** Git access is authenticated with SSH deploy keys stored as Kubernetes Secrets.

**How it is implemented here.** Each cluster has one Secret named `flux-system` in the `flux-system` namespace, referenced by the GitRepository via `secretRef` (REQ-010, NFR-005, CON-009).

**Current gap.** The key is stored as a plain Secret; rotation is manual and there is no external secret manager (GAP-005, ADR-008).

## Network Security (NetworkPolicies)

**Concept.** Network traffic to and from Flux controllers is restricted at the network layer (NFR-006).

**How it is implemented here.** Three NetworkPolicies in `flux-system` (REQ-006):
- `allow-egress` — allow all egress; ingress only from same-namespace pods.
- `allow-scraping` — allow ingress on port 8080/TCP from any namespace (metrics).
- `allow-webhooks` — allow ingress on port 9292/TCP from any namespace to notification-controller pods.

**Current gap.** None — policies exist but no policy engine (GAP-014).

## Pod Security Standards

**Concept.** Namespaces carry pod security labels to warn on privileged admissions (NFR-007).

**How it is implemented here.** The `flux-system` namespace carries `pod-security.kubernetes.io/warn: restricted` and `pod-security.kubernetes.io/warn-version: latest` (REQ-005). This is a warn-only baseline; future workload namespaces should follow it.

**Current gap.** Pod security is warn-only (not enforced); no policy engine (GAP-014).

## Resource Management and Quotas

**Concept.** Resource usage in the Flux namespace is bounded to protect cluster-critical workloads (NFR-008).

**How it is implemented here.** The `critical-pods-flux-system` ResourceQuota limits pods to 1000, scoped to the `system-node-critical` and `system-cluster-critical` priority classes (REQ-007).

**Current gap.** None.

## RBAC and Least Privilege

**Concept.** Controllers get the permissions they need; humans do not normally need cluster access (NFR-004).

**How it is implemented here.** `cluster-reconciler-flux-system` binds `cluster-admin` to kustomize-controller and helm-controller ServiceAccounts; `crd-controller-flux-system` grants CRD management to all controller ServiceAccounts; aggregated `flux-edit-flux-system` and `flux-view-flux-system` ClusterRoles provide edit/view access to Flux resources (REQ-008).

**Current gap.** None.

## Multi-Cluster Strategy

**Concept.** Environments are separated with independent Flux instances (NFR-011).

**How it is implemented here.** Each cluster has its own `clusters/<cluster-name>/flux-system/` directory and its own `gotk-sync.yaml` pointing at its cluster-specific path (REQ-004, ADR-005). The component manifests are byte-identical across clusters.

**Current gap.** None.

## Version Pinning and Upgrade Strategy

**Concept.** Versions are pinned so upgrades are deliberate (NFR-009).

**How it is implemented here.** Flux v2.9.5 and component versions are pinned in the bootstrap manifests.

**Current gap.** There is no documented upgrade procedure (GAP-008).

## Observability (Current State and Gap)

**Concept.** Reconciliation health should be observable.

**How it is implemented here.** Controllers expose Prometheus metrics on port 8080 and healthz endpoints; the NetworkPolicy `allow-scraping` permits scraping from any namespace.

**Current gap.** No ServiceMonitors, dashboards or alerting are configured; notification-controller is unused (GAP-004, GAP-012).

## Auditability and Traceability

**Concept.** All changes to cluster state are traceable (NFR-010).

**How it is implemented here.** Git history records every commit, author and timestamp; because Git is the only path to change managed state, the commit history is the audit trail.

**Current gap.** None.