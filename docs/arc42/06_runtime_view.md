# Runtime View

*Last verified against repository state: 2026-09-15.*

This section describes the runtime behavior of the system through three scenarios. Component names and interfaces follow the [Building Block View](05_building_block_view.md).

## Scenario 1: Initial Bootstrap

The only human-triggered operation in the system (NFR-004): an operator bootstraps a fresh cluster with `flux bootstrap git`.

```mermaid
sequenceDiagram
    participant O as Operator
    participant CLI as "flux CLI"
    participant API as kube-apiserver
    participant SC as source-controller
    participant KC as kustomize-controller
    participant GH as GitHub
    O->>CLI: flux bootstrap git
    CLI->>API: create flux-system namespace + apply kustomization.yaml
    API-->>CLI: applied
    CLI->>API: create GitRepository + Kustomization
    API-->>CLI: created
    SC->>API: watch GitRepository
    SC->>GH: SSH fetch (1m)
    GH-->>SC: artifact revision
    KC->>API: watch Kustomization
    KC->>SC: fetch artifact
    KC->>API: kustomize build + apply
    API-->>KC: applied
```

**Notable aspects.** The bootstrap is the only operation performed directly against the cluster; everything after it is driven by Git. The same procedure on any cluster produces an identical Flux deployment from the same commit (NFR-003). The bootstrap procedure itself is not yet documented in the repository (GAP-011).

## Scenario 2: Steady-State Reconciliation

The normal operating mode: Flux continuously polls Git and reconciles the cluster.

```mermaid
sequenceDiagram
    participant GH as GitHub
    participant SC as source-controller
    participant KC as kustomize-controller
    participant API as kube-apiserver
    SC->>GH: poll Git (1m)
    GH-->>SC: new commit detected
    SC->>SC: build artifact + revision
    KC->>SC: fetch artifact (10m)
    SC-->>KC: artifact
    KC->>KC: kustomize build + diff
    KC->>API: apply changes
    API-->>KC: applied
    KC->>KC: update status conditions
```

**Notable aspects.** The cadence is fixed by the resource intervals: 1m for the GitRepository and 10m for the Kustomization. Because there is no webhook Receiver, the maximum latency for a change to reach the cluster is ~10 minutes (GAP-001).

## Scenario 3: Drift Detection and Correction

Flux detects and corrects drift between the Git state and the live cluster state (NFR-002).

```mermaid
sequenceDiagram
    participant U as "Human (kubectl)"
    participant API as kube-apiserver
    participant KC as kustomize-controller
    participant SC as source-controller
    participant GH as GitHub
    U->>API: kubectl edit managed resource
    API-->>U: changed (drift)
    KC->>SC: fetch artifact (10m)
    SC->>GH: poll (if needed)
    GH-->>SC: Git state
    SC-->>KC: artifact
    KC->>API: revert to Git state
    API-->>KC: reverted
```

**Notable aspects.** Two drift cases are handled:

1. **Manual modification** — a Flux-managed resource is edited out-of-band; at the next reconcile (≤10m) kustomize-controller reverts it to the Git state.
2. **Deletion from Git** — a resource is removed from the repository; at the next reconcile it is **pruned** from the cluster (`prune: true`).

**Failure handling.** If GitHub is unreachable, source-controller reports `Ready=False` and kustomize-controller keeps applying the last-known-good artifact. There is no alerting configured, so failures are not proactively surfaced (GAP-004).