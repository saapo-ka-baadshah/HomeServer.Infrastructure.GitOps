# Quality Requirements

*Last verified against repository state: 2026-09-15.*

## Quality Requirements Overview

### Quality Tree

```mermaid
flowchart TD
    Q[Quality] --> R[Reliability]
    Q --> S[Security]
    Q --> M[Maintainability]
    Q --> A[Auditability]
    Q --> RP[Reproducibility]
    R --> R1[Drift detection and self-healing - NFR-002]
    R --> R2[Resource quotas for stability - NFR-008]
    S --> S1[Network segmentation - NFR-006]
    S --> S2[Pod security baseline - NFR-007]
    S --> S3[SSH-based Git authentication - NFR-005]
    S --> S4[Minimal human access - NFR-004]
    M --> M1[Declarative state - NFR-001]
    M --> M2[Version pinning - NFR-009]
    A --> A1[Git traceability - NFR-010]
    RP --> RP1[Bootstrap from scratch (NFR-003)]
```

### Quality Requirements Table

| NFR ID | Category | Requirement | Priority |
|--------|----------|-------------|----------|
| NFR-001 | Architecture | All cluster state declared in Git; no manual `kubectl apply` for managed resources. | High |
| NFR-002 | Reliability | Continuous reconciliation (1m Git, 10m Kustomization) corrects drift; `prune: true`. | High |
| NFR-003 | Reproducibility | Any cluster recreated from scratch from the same Git commit. | High |
| NFR-004 | Security | Operations via Git commits; direct cluster access exceptional. | Medium |
| NFR-005 | Security | Git access via SSH deploy keys stored as Kubernetes Secrets. | High |
| NFR-006 | Security | NetworkPolicies restrict traffic to/from Flux controllers. | High |
| NFR-007 | Security | Restricted pod security warnings on `flux-system`; baseline for future namespaces. | High |
| NFR-008 | Reliability | ResourceQuota prevents runaway pods from starving cluster-critical workloads. | High |
| NFR-009 | Maintainability | Flux and component versions pinned; upgrades are intentional Git commits. | Medium |
| NFR-010 | Auditability | All changes traceable via Git history. | Medium |
| NFR-011 | Architecture | Kind (dev) and homeserver-cluster (prod) isolated with independent loops. | Medium |
| NFR-012 | Security (Gap) | No external secret management — SSH key stored as plain Secret. | Known gap |

## Quality Scenarios

| # | Stimulus | Source | Environment | Artifact | Response | Response measure |
|---|----------|--------|-------------|----------|----------|------------------|
| 1 | A managed resource is edited out-of-band | Human with `kubectl` | Running cluster | Flux-managed resource | kustomize-controller reverts it to the Git state at the next reconcile | Reverted within ≤10m (NFR-002) |
| 2 | A resource is deleted from Git | Developer commit | Running cluster | Flux-managed resource | kustomize-controller prunes it from the cluster | Resource removed within ≤10m (NFR-002) |
| 3 | A fresh cluster is bootstrapped | Operator | New cluster | Flux deployment | `flux bootstrap git` produces the identical Flux deployment | Same Git commit → same component versions and configuration (NFR-003) |
| 4 | The SSH deploy key is rotated | Operator | Running cluster | Secret `flux-system` | GitRepository continues to fetch after the Secret is updated | `Ready=True` with current artifact revision (NFR-005) |
| 5 | Flux is upgraded | Platform Engineer | Repository | Bootstrap manifests | Pinned versions updated via a Git commit and reconciled | Controllers run the new pinned versions (NFR-009) |
| 6 | "Who changed X?" audit query | Security Auditor | Repository | Git history | Commit history shows author, timestamp and diff for the change | Traceable via `git log` (NFR-010) |