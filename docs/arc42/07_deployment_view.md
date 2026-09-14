# Deployment View

*Last verified against repository state: 2026-09-15.*

## Infrastructure Level 1

### Overview Diagram

```mermaid
flowchart LR
    GH[GitHub Repository<br/>saapo-ka-baadshah/HomeServer.Infrastructure.GitOps]
    K[Kind Cluster - dev] -->|SSH fetch (private key)| GH
    H[Homeserver Cluster - prod] -->|SSH fetch (private key)| GH
    K --> KF[flux-system: 4 controllers]
    H --> HF[flux-system: 4 controllers]
```

### Motivation

Two independent environments are deployed from the same repository: `clusters/kind` (development) and `clusters/homeserver-cluster` (production). Each cluster runs its own Flux instance with its own SSH deploy key and its own reconciliation loop, so there is no cross-contamination between environments (REQ-004, NFR-011).

### Quality and/or Performance Features

- Reconciliation cadence: GitRepository 1m, Kustomization 10m.
- ResourceQuota `critical-pods-flux-system` protects cluster-critical pods (REQ-007).
- `gotk-components.yaml` is byte-identical across clusters; only the sync path differs, keeping environments consistent.

### Mapping of Building Blocks to Infrastructure

| Building block | Cluster | Location |
|----------------|---------|----------|
| source-controller | kind, homeserver-cluster | `flux-system` namespace |
| kustomize-controller | kind, homeserver-cluster | `flux-system` namespace |
| helm-controller | kind, homeserver-cluster | `flux-system` namespace |
| notification-controller | kind, homeserver-cluster | `flux-system` namespace |
| GitRepository `flux-system` | kind, homeserver-cluster | `flux-system` namespace |
| Kustomization `flux-system` | kind, homeserver-cluster | `flux-system` namespace |

## Infrastructure Level 2

### Kind Cluster (Development)

A local Kubernetes-in-Docker cluster used for development and testing changes before they reach production (assumption 4: kind = local dev/test). It is bootstrapped from `clusters/kind`. **Note:** the kind cluster configuration is not versioned in this repository, so the dev environment is not fully reproducible from the repo alone (GAP-013).

### Homeserver Cluster (Production)

A long-lived, persistent cluster hosting the HomeServer workloads (assumption 5: homeserver-cluster = long-lived persistent). It is bootstrapped from `clusters/homeserver-cluster`. No application workloads are deployed yet (CON-010, GAP-003).

### Flux Controllers (per cluster)

Each cluster runs the same Flux topology in the `flux-system` namespace: four controller Deployments (source-controller v1.9.5, kustomize-controller v1.9.5, helm-controller v1.6.4, notification-controller v1.9.4), Services for source-controller and notification-controller (ClusterIP, port 80), metrics on port 8080, and a webhook listener on port 9292 (currently unused). See the [Building Block View](05_building_block_view.md) for details.

### GitHub Repository (external)

The repository `saapo-ka-baadshah/HomeServer.Infrastructure.GitOps` is hosted on GitHub and accessed over SSH (CON-006). Each cluster authenticates with its own read-only SSH deploy key (REQ-010). The repository is the single source of truth for cluster state.