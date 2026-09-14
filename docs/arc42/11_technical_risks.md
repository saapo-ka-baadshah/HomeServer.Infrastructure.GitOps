# Risks and Technical Debts

*Last verified against repository state: 2026-09-15.*

## Risk Overview

The known gaps and open questions are defined in [specs/REQUIREMENTS.md](../../specs/REQUIREMENTS.md) §6. Priorities are taken from the baseline.

| Gap ID | Description | Impact | Priority | Mitigation |
|--------|-------------|--------|----------|------------|
| GAP-001 | No Git webhook Receiver — polling only (1m/10m) | Latency up to 10m for changes to apply | Medium | Add a `Receiver` resource and expose the webhook endpoint |
| GAP-002 | No backup/DR strategy — no etcd backup, no Git repo backup, no restore procedure | Risk of data loss | High | Document and implement etcd + Git backup and restore |
| GAP-003 | No application workloads — no example workload, no tenant namespace structure | Blocks validation of end-to-end GitOps flow | Medium | Add a first workload and namespace structure |
| GAP-004 | No observability configuration — no ServiceMonitors, dashboards, alerting | Cannot monitor reconciliation health | Medium | Add Prometheus ServiceMonitors, dashboards, alerting |
| GAP-005 | No external secret management — SSH key stored as plain Secret | Secret rotation difficult; key visible in cluster | High | Adopt Sealed Secrets / External Secrets Operator / SOPS |
| GAP-006 | Single branch strategy — no promotion flow | All changes go directly to production | Medium | Introduce promotion flow when needed |
| GAP-007 | No multi-tenancy / namespace strategy | Security/isolation gaps when workloads added | Medium | Define tenant namespaces and NetworkPolicy defaults |
| GAP-008 | No Flux upgrade procedure | Risk of version skew, missed security patches | Low | Document the upgrade process |
| GAP-009 | `temp/` directory empty and gitignored | Confusion; potential for misuse | Low | Remove it or document its purpose |
| GAP-010 | No CI/CD pipeline to validate manifests before merge | Broken manifests can be merged | High | Add GitHub Actions: kustomize build, kubeconform, flux lint |
| GAP-011 | No documentation for the bootstrap procedure | Onboarding friction | Medium | Document `flux bootstrap git` in the README |
| GAP-012 | notification-controller configured but unused | Dead code / unused component | Low | Wire alerting or remove the component |
| GAP-013 | Kind cluster config not in repo | Dev environment not reproducible from repo alone | Medium | Version the kind config |
| GAP-014 | No policy enforcement (OPA/Gatekeeper/Kyverno) | Workloads can violate security baseline | Medium | Add admission control policies |

## Technical Debts

- **Empty `infrastructure/` scaffolding.** The `infrastructure/` directory exists as empty placeholder scaffolding with subdirectories `controllers/`, `namespaces/`, `overlays/`, `sources/`. `infrastructure/controllers/loadbalancer.yaml` is an empty file (0 bytes). None of this scaffolding is wired into any cluster Kustomization or deployed.
- **Empty gitignored `temp/` directory (GAP-009).** A placeholder directory with no documented purpose.
- **notification-controller installed but unused (GAP-012).** The webhook NetworkPolicy exists, but no Receivers or alerting are configured.
- **No CI/CD validation (GAP-010).** Manifests are merged without automated validation.
- **No documented bootstrap procedure (GAP-011).** The README only links to the arc42 documentation.

## Mitigation Roadmap

The roadmap below is **aspirational** — nothing beyond the current Flux bootstrap is implemented.

**High priority**
- Backup/DR strategy (GAP-002)
- External secret management (GAP-005)
- CI/CD validation of manifests (GAP-010)

**Medium priority**
- Webhook Receiver for push-based reconciliation (GAP-001)
- Observability: ServiceMonitors, dashboards, alerting (GAP-004)
- Multi-tenancy / namespace strategy (GAP-007)
- Documented bootstrap procedure (GAP-011)
- Versioned kind cluster config (GAP-013)
- Policy enforcement (GAP-014)

**Low priority**
- Documented Flux upgrade procedure (GAP-008)
- Resolve or document the `temp/` directory (GAP-009)