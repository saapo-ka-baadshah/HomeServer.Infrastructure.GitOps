# Architecture Decisions

*Last verified against repository state: 2026-09-15.*

This section records the significant architecture decisions for the HomeServer GitOps infrastructure. Each ADR follows the format **Status / Context / Decision / Consequences**. The decisions are referenced from [Solution Strategy](04_solution_strategy.md) and [Cross-cutting Concepts](08_concepts.md).

## Summary

| ADR | Decision | Status | Refs |
|-----|----------|--------|------|
| ADR-001 | Use Flux v2 for GitOps | Accepted | REQ-001, CON-001 |
| ADR-002 | Use SSH deploy keys for Git authentication | Accepted | REQ-010, NFR-005, CON-006 |
| ADR-003 | Polling-based reconciliation (no webhook receiver) | Accepted | CON-008, GAP-001 |
| ADR-004 | Kustomize for bootstrap manifest composition | Accepted | REQ-009, CON-003 |
| ADR-005 | Per-cluster bootstrap directories | Accepted | REQ-004, NFR-011, CON-004 |
| ADR-006 | Restricted pod security warnings on `flux-system` | Accepted | REQ-005, NFR-007 |
| ADR-007 | `cluster-admin` for kustomize-controller and helm-controller | Accepted | REQ-008, NFR-004 |
| ADR-008 | SSH key as plain Kubernetes Secret | Accepted (known gap) | GAP-005 |
| ADR-009 | Track only the `main` branch | Accepted | GAP-006 |
| ADR-010 | Install helm-controller despite no current HelmReleases | Accepted | CON-007 |

## ADR-001: Use Flux v2 for GitOps

**Status:** Accepted.

**Context:** The repository needed a GitOps engine to reconcile cluster state from Git. The alternatives considered were Flux v2, Argo CD, and manual `kubectl apply`.

**Decision:** Use Flux v2 (v2.9.5). Flux was chosen for its native Kustomize support, its `flux bootstrap` tooling, and its CNCF-graduated ecosystem.

**Consequences:** The cluster is managed by four Flux controllers (source-controller, kustomize-controller, helm-controller, notification-controller) pinned to v2.9.5 (REQ-001, CON-001). All cluster state must be declared in Git (NFR-001).

## ADR-002: Use SSH Deploy Keys for Git Authentication

**Status:** Accepted.

**Context:** Flux needs read access to the Git repository. The alternatives were an HTTPS personal access token or an SSH deploy key.

**Decision:** Use a read-only SSH deploy key per cluster, stored in a Kubernetes Secret (`flux-system`) and referenced via `secretRef` (REQ-010, CON-006).

**Consequences:** No token material is stored in the cluster; each cluster has its own key (NFR-005, CON-009). Key rotation requires updating the Secret (see GAP-005).

## ADR-003: Use Polling-Based Reconciliation (No Webhook Receiver)

**Status:** Accepted.

**Context:** Flux can be triggered by polling or by push-based webhooks via a `Receiver` resource.

**Decision:** Use polling only: the GitRepository polls every 1m and the Kustomization reconciles every 10m (CON-008).

**Consequences:** Changes take up to ~10 minutes to apply. A webhook Receiver can be added later to reduce latency (GAP-001).

## ADR-004: Use Kustomize for Bootstrap Manifest Composition

**Status:** Accepted.

**Context:** The bootstrap manifests (`gotk-components.yaml` + `gotk-sync.yaml`) need to be composed per cluster. The alternatives were Kustomize or a Helm chart.

**Decision:** Use a `kustomize.config.k8s.io/v1beta1` Kustomization that lists both generated manifests as resources (REQ-009, CON-003).

**Consequences:** Bootstrap is Flux-native with no chart dependency; `kustomize build clusters/<cluster>/flux-system` produces the full manifest set.

## ADR-005: Per-Cluster Bootstrap Directories

**Status:** Accepted.

**Context:** Multiple clusters (dev and prod) need independent Flux instances.

**Decision:** Each cluster has its own directory `clusters/<cluster-name>/flux-system/` with its own `gotk-sync.yaml` pointing at its cluster-specific path (REQ-004, CON-004).

**Consequences:** Clusters are isolated with independent reconciliation loops (NFR-011). The `gotk-components.yaml` is byte-identical across clusters; only the sync path differs.

## ADR-006: Enforce Restricted Pod Security Warnings on flux-system

**Status:** Accepted.

**Context:** The `flux-system` namespace should not admit privileged pods.

**Decision:** Apply `pod-security.kubernetes.io/warn: restricted` and `pod-security.kubernetes.io/warn-version: latest` labels to the namespace (REQ-005).

**Consequences:** Privileged admissions generate warnings rather than being blocked; this establishes a baseline for future workload namespaces (NFR-007).

## ADR-007: Grant cluster-admin to kustomize-controller and helm-controller

**Status:** Accepted.

**Context:** Flux reconcilers need broad permissions to apply arbitrary manifests.

**Decision:** Bind the `cluster-admin` ClusterRole to the kustomize-controller and helm-controller ServiceAccounts via the `cluster-reconciler-flux-system` ClusterRoleBinding (REQ-008).

**Consequences:** The reconcilers can manage any resource, which is required for full reconciliation. The risk is mitigated by the Git-only operational model — humans do not normally hold cluster-admin (NFR-004).

## ADR-008: Store SSH Key as Plain Kubernetes Secret (No External Secret Manager)

**Status:** Accepted (known gap).

**Context:** The SSH deploy key must be available to source-controller.

**Decision:** Store the key as a plain Kubernetes Secret named `flux-system` in the `flux-system` namespace (REQ-010).

**Consequences:** This is simple and works today, but key rotation is manual and the key is visible in the cluster. An external secret manager (Sealed Secrets, External Secrets Operator, SOPS) is planned (GAP-005).

## ADR-009: Track Only the main Branch (Single-Branch Strategy)

**Status:** Accepted.

**Context:** The repository needs a branching strategy.

**Decision:** Track only the `main` branch (`ref.branch: main`); all changes go directly to the tracked branch (CON-005).

**Consequences:** Simple workflow, but there is no promotion flow (dev → staging → prod); all changes reach the production cluster directly (GAP-006).

## ADR-010: Install helm-controller Despite No Current HelmReleases

**Status:** Accepted.

**Context:** The repository currently has no Helm-based workloads.

**Decision:** Install helm-controller (v1.6.4) as part of the standard Flux component set so HelmReleases can be added later without re-bootstrapping (CON-007).

**Consequences:** One additional controller runs idle today; it will be used when Helm-based workloads are introduced.