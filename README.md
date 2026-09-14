# HomeServer Infrastructure — GitOps

GitOps repository for managing HomeServer Kubernetes infrastructure using **Flux v2.9.5**. Git is the single source of truth for cluster state; Flux controllers continuously reconcile the actual cluster state against the desired state declared in this repository. Two independent clusters are configured: `clusters/kind` (development) and `clusters/homeserver-cluster` (production), each with its own Flux instance and reconciliation loop.

**Current state:** Only the Flux bootstrap layer is deployed — no application workloads exist yet.

---

## Repository Layout

```
.
├── clusters/
│   ├── kind/                          # Development cluster (Kubernetes in Docker)
│   │   └── flux-system/               # Flux bootstrap: gotk-components.yaml, gotk-sync.yaml, kustomization.yaml
│   └── homeserver-cluster/            # Production cluster
│       └── flux-system/               # Flux bootstrap (identical components, cluster-specific sync path)
├── docs/
│   ├── arc42/                         # 12-section arc42 architecture documentation
│   └── plans/
│       └── PLAN.md                    # Content plan used to author the arc42 docs
├── specs/
│   └── REQUIREMENTS.md                # Requirements baseline (REQ/NFR/CON/GAP IDs)
└── infrastructure/                    # Empty scaffolding (placeholder for future workloads)
```

---

## Documentation

### Architecture Documentation (arc42)

The `docs/arc42/` directory contains the complete arc42 v9.0 architecture documentation (12 sections):

| Section | File | Description |
|---------|------|-------------|
| 1 | [01_introduction_and_goals.md](docs/arc42/01_introduction_and_goals.md) | Requirements overview, quality goals, and stakeholders |
| 2 | [02_architecture_constraints.md](docs/arc42/02_architecture_constraints.md) | Technical and organizational constraints, conventions |
| 3 | [03_context_and_scope.md](docs/arc42/03_context_and_scope.md) | Business and technical context diagrams, external actors, interfaces |
| 4 | [04_solution_strategy.md](docs/arc42/04_solution_strategy.md) | GitOps strategy, Flux engine, multi-cluster layout, bootstrap, security, evolution path |
| 5 | [05_building_block_view.md](docs/arc42/05_building_block_view.md) | Whitebox system view, black-box component details, reconciliation pipeline |
| 6 | [06_runtime_view.md](docs/arc42/06_runtime_view.md) | Runtime scenarios: bootstrap, steady-state reconciliation, drift correction |
| 7 | [07_deployment_view.md](docs/arc42/07_deployment_view.md) | Infrastructure mapping: kind (dev) and homeserver-cluster (prod) |
| 8 | [08_concepts.md](docs/arc42/08_concepts.md) | Cross-cutting concepts: GitOps, reconciliation, secrets, network, RBAC, observability |
| 9 | [09_architecture_decisions.md](docs/arc42/09_architecture_decisions.md) | 10 ADRs (ADR-001…ADR-010) with status, context, decision, consequences |
| 10 | [10_quality_requirements.md](docs/arc42/10_quality_requirements.md) | Quality tree, NFR table (NFR-001…NFR-011), quality scenarios |
| 11 | [11_technical_risks.md](docs/arc42/11_technical_risks.md) | Risk overview (GAP-001…GAP-014), technical debts, mitigation roadmap |
| 12 | [12_glossary.md](docs/arc42/12_glossary.md) | Terminology definitions used throughout the documentation |

**Start here:** [Introduction and Goals](docs/arc42/01_introduction_and_goals.md) → [Solution Strategy](docs/arc42/04_solution_strategy.md) → [Building Block View](docs/arc42/05_building_block_view.md)

### Supporting Documents

- **[specs/REQUIREMENTS.md](specs/REQUIREMENTS.md)** — Formal requirements baseline: 10 functional requirements (REQ-001…REQ-010), 12 non-functional requirements (NFR-001…NFR-012), 10 constraints (CON-001…CON-010), and 14 known gaps (GAP-001…GAP-014). All arc42 sections trace to these IDs.
- **[docs/plans/PLAN.md](docs/plans/PLAN.md)** — The content plan that guided the arc42 authoring, including ground-truth facts, per-file outlines, cross-section dependencies, and QA checklist.

---

## Getting Started (Bootstrap)

This repository is a **GitOps control repo** — it is not an application you run locally. To bootstrap a new cluster:

1. Ensure you have the `flux` CLI installed and a Kubernetes context pointing to the target cluster.
2. Generate an SSH deploy key (read-only) and add it to the GitHub repository as a deploy key.
3. Run `flux bootstrap git` with the repository URL, branch, cluster-specific path, and the private key file. Example semantics:

   ```bash
   flux bootstrap git \
     --url=ssh://git@github.com/saapo-ka-baadshah/HomeServer.Infrastructure.GitOps \
     --branch=main \
     --path=clusters/<cluster-name> \
     --private-key-file=<path-to-ssh-private-key>
   ```

   This installs the Flux controllers into the `flux-system` namespace, creates the `GitRepository` and `Kustomization` resources, and commits the generated bootstrap manifests (`gotk-components.yaml`, `gotk-sync.yaml`) to the repository under `clusters/<cluster-name>/flux-system/`.

4. After bootstrap, Flux will reconcile the cluster every 10 minutes (GitRepository polls every 1 minute). All further changes are made by committing to the `main` branch.

**Note:** The SSH deploy key is required for Git access. Each cluster uses its own key stored as a Kubernetes Secret named `flux-system` in the `flux-system` namespace. See [ADR-002](docs/arc42/09_architecture_decisions.md#adr-002-use-ssh-deploy-keys-for-git-authentication) and [REQ-010](specs/REQUIREMENTS.md#req-010-authenticate-to-git-via-ssh-deploy-key) for details.

---

## References

- Flux v2.9.5 Documentation: https://fluxcd.io/flux/v2.9.5/
- GitOps Principles (OpenGitOps): https://opengitops.dev/
- arc42 Template v9.0: https://arc42.org/