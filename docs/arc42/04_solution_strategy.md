# Solution Strategy

*Last verified against repository state: 2026-09-15.*

The overall strategy is summarized below and detailed in the following sections.

| Strategy element | Implementation | Refs |
|------------------|----------------|------|
| Git as single source of truth | GitRepository + Kustomization reconcile from Git; no manual `kubectl apply` | NFR-001, REQ-002, REQ-003 |
| Flux v2 as GitOps engine | Four controllers pinned to v2.9.5 | REQ-001, CON-001, ADR-001 |
| Multi-cluster isolation | `clusters/<name>/flux-system` per cluster | REQ-004, NFR-011, ADR-005 |
| Bootstrap via Kustomize | `kustomization.yaml` composes components + sync | REQ-009, CON-003, ADR-004 |
| Security posture | SSH keys, NetworkPolicies, pod security, RBAC, quota | REQ-005 to REQ-010, ADR-002, ADR-006, ADR-007, ADR-008 |
| Evolution path | GAP-driven roadmap | GAP-001 to GAP-007, GAP-010 |

## GitOps as the Core Strategy

Git is the **single source of truth** for the desired state of the cluster (NFR-001). All cluster state is declared declaratively in YAML manifests; Flux controllers pull the desired state from Git and reconcile the live cluster to it. There is no manual `kubectl apply` for managed resources. This follows the OpenGitOps principles ([opengitops.dev](https://opengitops.dev/)).

## Flux v2 as the GitOps Engine

Flux v2 (v2.9.5) is the GitOps engine ([ADR-001](09_architecture_decisions.md#adr-001-use-flux-v2-for-gitops)). Four controllers run in each cluster:

| Controller | Version | Role |
|------------|---------|------|
| source-controller | v1.9.5 | Fetches Git sources and produces artifacts (1m poll). |
| kustomize-controller | v1.9.5 | Reconciles Kustomization resources: build, diff, apply, prune (10m). |
| helm-controller | v1.6.4 | Reconciles HelmReleases (installed, currently idle — CON-007). |
| notification-controller | v1.9.4 | Webhook receivers and notifications (installed, currently unused — GAP-012). |

Versions are pinned in the bootstrap manifests (NFR-009); upgrades are intentional Git commits.

## Repository Layout and Multi-Cluster Strategy

Each cluster has its own bootstrap directory `clusters/<cluster-name>/flux-system/` ([ADR-005](09_architecture_decisions.md#adr-005-per-cluster-bootstrap-directories)), giving per-cluster isolation and independent reconciliation loops (REQ-004, NFR-011). The `gotk-components.yaml` is byte-identical across clusters; only the Kustomization `path` in `gotk-sync.yaml` differs (`./clusters/kind` vs `./clusters/homeserver-cluster`).

## Bootstrap Approach

New clusters are bootstrapped with `flux bootstrap git`, which installs the controllers and commits the generated `gotk-components.yaml` and `gotk-sync.yaml` to the repository. Each cluster's `kustomization.yaml` composes these two manifests via `kustomize.config.k8s.io/v1beta1` ([ADR-004](09_architecture_decisions.md#adr-004-use-kustomize-for-bootstrap-manifest-composition), REQ-009). Any cluster can be recreated from scratch from the same Git commit (NFR-003).

## Security Posture

| Element | Implementation | Refs |
|---------|----------------|------|
| Git authentication | Read-only SSH deploy key per cluster, stored as Secret `flux-system` | REQ-010, NFR-005, ADR-002 |
| Network segmentation | NetworkPolicies `allow-egress`, `allow-scraping` (8080), `allow-webhooks` (9292) | REQ-006, NFR-006 |
| Pod security | `restricted` warn labels on `flux-system` namespace | REQ-005, NFR-007, ADR-006 |
| RBAC | `cluster-admin` for reconcilers, aggregated edit/view roles, CRD management | REQ-008, NFR-004, ADR-007 |
| Resource management | ResourceQuota `critical-pods-flux-system` (1000 pods, critical priority classes) | REQ-007, NFR-008 |

**Honest note:** the SSH deploy key is stored as a plain Kubernetes Secret; external secret management is not yet in place (GAP-005, ADR-008).

## Evolution Path (Current Gaps)

The following are **planned, not implemented**:

- Application workloads and a tenant namespace structure (GAP-003)
- Observability: ServiceMonitors, dashboards, alerting (GAP-004)
- CI/CD validation of manifests before merge (GAP-010)
- External secret management (GAP-005)
- Webhook Receiver for push-based reconciliation (GAP-001)
- Backup and disaster recovery (GAP-002)
- Multi-tenancy / namespace strategy (GAP-007)

See [Risks and Technical Debts](11_technical_risks.md) for the full roadmap.