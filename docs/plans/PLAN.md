# Content Plan — arc42 Architecture Documentation (docs/arc42/*.md)

**Project:** HomeServer Infrastructure — GitOps
**Plan version:** 1.0
**Date:** 2026-09-15
**Inputs:** `specs/REQUIREMENTS.md` (requirements baseline v1.0), repository manifests under `clusters/`, git history.
**Goal:** Fill all 12 arc42 v9.0 template files in `docs/arc42/` with accurate, substantive, honest content about the current (early-stage) GitOps HomeServer infrastructure.

---

## 1. Purpose & Scope

This plan tells the developer agent exactly what to write in each of the 12 arc42 files: which template headings to keep/rename/remove, which content points to include, which tables/diagrams to produce, and which requirement IDs from `specs/REQUIREMENTS.md` each section must reflect.

**Scope boundary:** The documentation describes the repository **as it exists today** — Flux v2.9.5 bootstrap only, two clusters, no application workloads, no webhook receiver, no backup, no CI/CD, no external secret management. The docs must be honest about this state and must not claim features that do not exist.

---

## 2. Ground-Truth Facts (verified from repository — do not contradict these)

These facts were verified by reading the actual manifests. Every factual claim in the arc42 docs must match this table.

| # | Fact | Source |
|---|------|--------|
| F1 | Flux version **v2.9.5**; components: **source-controller v1.9.5**, **kustomize-controller v1.9.5**, **helm-controller v1.6.4**, **notification-controller v1.9.4** | `gotk-components.yaml` header + Deployment `image:` fields |
| F2 | `GitRepository` `flux-system` in namespace `flux-system`: `interval: 1m0s`, `ref.branch: main`, `url: ssh://git@github.com/saapo-ka-baadshah/HomeServer.Infrastructure.GitOps`, `secretRef.name: flux-system` | `gotk-sync.yaml` (both clusters) |
| F3 | `Kustomization` `flux-system`: `interval: 10m0s`, `path: ./clusters/<cluster-name>`, `prune: true`, `sourceRef: GitRepository/flux-system` | `gotk-sync.yaml` (both clusters) |
| F4 | `kustomization.yaml` uses `kustomize.config.k8s.io/v1beta1`; resources = `gotk-components.yaml` + `gotk-sync.yaml` | `kustomization.yaml` (both clusters) |
| F5 | `gotk-components.yaml` is **byte-identical** across both clusters; `gotk-sync.yaml` differs **only** in the Kustomization `path` (`./clusters/kind` vs `./clusters/homeserver-cluster`) | `diff` of both clusters' files |
| F6 | Namespace `flux-system` labels: `pod-security.kubernetes.io/warn: restricted`, `pod-security.kubernetes.io/warn-version: latest`, plus `app.kubernetes.io/instance: flux-system`, `app.kubernetes.io/part-of: flux`, `app.kubernetes.io/version: v2.9.5` | `gotk-components.yaml` lines 5–14 |
| F7 | NetworkPolicies in `flux-system`: `allow-egress` (egress all `{}`, ingress same-namespace podSelector), `allow-scraping` (ingress 8080/TCP from any namespace), `allow-webhooks` (ingress 9292/TCP from any namespace, podSelector `app: notification-controller`) | `gotk-components.yaml` lines 16–76 |
| F8 | `ResourceQuota` `critical-pods-flux-system`: `hard.pods: "1000"`, scopeSelector PriorityClass ∈ {system-node-critical, system-cluster-critical} | `gotk-components.yaml` lines 78–96 |
| F9 | RBAC: ClusterRoleBinding `cluster-reconciler-flux-system` binds `cluster-admin` to kustomize-controller + helm-controller SAs; ClusterRoleBinding `crd-controller-flux-system` binds CRD-management ClusterRole to all controller SAs (incl. image-reflector, image-automation, source-watcher); aggregated ClusterRoles `flux-edit-flux-system` and `flux-view-flux-system` | `gotk-components.yaml` lines 98–311 |
| F10 | Services: `source-controller` (ClusterIP 80→http), `notification-controller` (ClusterIP 80→http), `webhook-receiver` (ClusterIP 80→http-webhook) | `gotk-components.yaml` |
| F11 | Controllers expose metrics on **8080** and healthz; notification-controller webhook listener on **9292** | `gotk-components.yaml` Deployment containerPorts + NetworkPolicies |
| F12 | Repository layout: `clusters/<cluster-name>/flux-system/{gotk-components.yaml, gotk-sync.yaml, kustomization.yaml}`; clusters present: `kind`, `homeserver-cluster` | filesystem |
| F13 | `infrastructure/` directory exists as **empty scaffolding**: `controllers/loadbalancer.yaml` (empty file), `namespaces/`, `overlays/`, `sources/` (all empty) | filesystem |
| F14 | `temp/` directory is empty and gitignored | filesystem + `.gitignore` |
| F15 | No application workloads. Git history shows podinfo manifests were added (commit `2773685`) and removed (commit `41fdb0c`) — do **not** document podinfo as current | git log |
| F16 | No webhook receiver configured (no `Receiver` CRD resources); polling only. No backup, no observability config, no CI/CD, no external secrets operator | repository scan + `specs/REQUIREMENTS.md` GAPs |
| F17 | README.md is minimal (7 lines) and links to arc42 docs | README.md |
| F18 | `specs/REQUIREMENTS.md` is the requirements baseline: REQ-001…REQ-010, NFR-001…NFR-012, CON-001…CON-010, GAP-001…GAP-014, stakeholders, assumptions | specs/REQUIREMENTS.md |

---

## 3. Current State Summary (to be reflected consistently everywhere)

- **What exists:** Flux v2.9.5 bootstrap layer only. Two independent cluster configurations (`kind` = dev, `homeserver-cluster` = prod), each with its own `flux-system` namespace, GitRepository (1m poll), Kustomization (10m reconcile, prune), and identical component manifests.
- **What does NOT exist yet:** application workloads, HelmReleases, webhook `Receiver`, backup/DR, observability (ServiceMonitors/dashboards/alerting), CI/CD validation, external secret management, policy engine, multi-tenancy namespace strategy, documented upgrade procedure, versioned kind cluster config.
- **Intent:** Git is the single source of truth; all future workloads and infrastructure should be added via Git commits and reconciled by Flux.

---

## 4. Per-File Content Plans

For each file: **Template disposition** (what to do with the current template headings), **Outline** (exact heading structure to produce), **Key content points**, **Tables/Diagrams**, and **Requirements to reflect**.

---

### 4.1 `docs/arc42/01_introduction_and_goals.md`

**Template disposition:** Keep the three template headings (`Requirements Overview`, `Quality Goals`, `Stakeholders`). Replace the placeholder stakeholder table rows (`<Role-1>` etc.) with real stakeholders.

**Outline:**
```
# Introduction and Goals
## Requirements Overview
## Quality Goals
## Stakeholders
```

**Key content points:**
- **Requirements Overview:**
  - 1–2 sentence intro: this repository implements a GitOps infrastructure for a HomeServer Kubernetes cluster using Flux v2.9.5; Git is the single source of truth; current state = Flux bootstrap only.
  - Table of functional requirements (ID, Requirement, Short description) — one row per REQ-001…REQ-010, condensed from `specs/REQUIREMENTS.md` §2. Add a note: "Full acceptance criteria and traceability in `specs/REQUIREMENTS.md`."
  - Explicit "Current Scope" note: only the Flux bootstrap layer is deployed; no workloads (CON-010, GAP-003).
- **Quality Goals:**
  - Table: Goal | Description | NFR ref | Priority (High/Medium). Map: Declarative state management (NFR-001), Drift detection & self-healing (NFR-002), Reproducibility (NFR-003), Minimal human access (NFR-004), Security hardening (NFR-005…008), Version pinning (NFR-009), Auditability (NFR-010), Environment separation (NFR-011).
  - One short paragraph stating the top-3 priorities: reliability (self-healing), security, reproducibility.
- **Stakeholders:**
  - Replace placeholder rows with the four stakeholders from `specs/REQUIREMENTS.md` §8: Platform Engineer (Owner), Developer (Workload Author), Security Auditor, Operator (On-call). Contact column: use `—` or the GitHub handle `saapo-ka-baadshah` for the owner; Expectations copied/paraphrased from §8.

**Tables/Diagrams:** Two tables (requirements overview, quality goals, stakeholders). No diagram needed.

**Requirements to reflect:** REQ-001…REQ-010 (overview table), NFR-001…NFR-011 (quality goals), CON-010 / GAP-003 (current scope note).

---

### 4.2 `docs/arc42/02_architecture_constraints.md`

**Template disposition:** Keep the `# Architecture Constraints` heading; add three subsections (template file currently has none).

**Outline:**
```
# Architecture Constraints
## Technical Constraints
## Organizational Constraints
## Conventions
```

**Key content points:**
- **Technical Constraints** — table of CON-001…CON-010 from `specs/REQUIREMENTS.md` §4:
  - CON-001 Flux v2.9.5 + component versions (F1)
  - CON-002 Kubernetes API versions in use (`source.toolkit.fluxcd.io/v1`, `kustomize.toolkit.fluxcd.io/v1`, `kustomize.config.k8s.io/v1beta1`, `networking.k8s.io/v1`, `rbac.authorization.k8s.io/v1`, `apiextensions.k8s.io/v1`)
  - CON-003 Kustomize API v1beta1 (F4)
  - CON-004 Cluster naming convention `clusters/<name>/flux-system/` (F12)
  - CON-005 Only `main` branch tracked (F2)
  - CON-006 GitHub via SSH (F2)
  - CON-007 helm-controller installed but no HelmReleases (F1, F16)
  - CON-008 No webhook receiver; polling only (F16)
  - CON-009 Single SSH deploy key secret per cluster (F2)
  - CON-010 No application workloads (F15)
- **Organizational Constraints:**
  - Single maintainer/owner; single `main` branch workflow (no promotion flow) — GAP-006.
  - No CI/CD pipeline yet — GAP-010.
  - No documented Flux upgrade procedure — GAP-008.
  - Kind cluster config not versioned — GAP-013.
- **Conventions:**
  - Directory layout convention (F12) with a small tree code block.
  - `gotk-components.yaml` / `gotk-sync.yaml` are Flux-generated: header comment "This manifest was generated by flux. DO NOT EDIT." — edits to bootstrap manifests should be made via `flux bootstrap` regeneration, not hand-editing.
  - Version pinning convention: Flux version + component versions pinned in manifests (NFR-009).
  - Naming conventions: `flux-system` namespace, resources named `flux-system`, `critical-pods-flux-system`, `cluster-reconciler-flux-system`, etc.

**Tables/Diagrams:** One constraints table (CON-001…CON-010); one small directory-tree code block.

**Requirements to reflect:** CON-001…CON-010, NFR-009, GAP-006, GAP-008, GAP-010, GAP-013.

---

### 4.3 `docs/arc42/03_context_and_scope.md`

**Template disposition:** Keep `## Business Context` and `## Technical Context`. Replace the `<Diagram or Table>` / `<optionally: ...>` / `<Mapping Input/Output to Channels>` placeholders with real content. Rename the last placeholder block to a real heading `### Mapping Input/Output to Channels` (or fold it into Technical Context as a table).

**Outline:**
```
# Context and Scope
## Business Context
### Context Diagram
### External Actors and Interfaces
## Technical Context
### Technical Context Diagram
### Technical Interfaces
### Mapping Input/Output to Channels
```

**Key content points:**
- **Business Context:**
  - Mermaid flowchart (context diagram): central system node "HomeServer GitOps (Flux controllers in Kubernetes cluster)"; external actors: GitHub repository, Platform Engineer (commits), Developer (commits), Operator (monitors), Security Auditor (reviews). Arrows: engineers → Git commits → system; system → GitHub (SSH fetch); system → cluster state.
  - Table of external actors: Actor | Role | Interaction.
  - Explanation paragraph: the system's business purpose is to keep the HomeServer cluster state identical to the Git repository; humans interact only via Git.
- **Technical Context:**
  - Mermaid flowchart (technical): GitHub repo → (SSH, 1m poll) → source-controller → artifact → kustomize-controller → kube-apiserver → cluster resources; notification-controller (webhook port 9292, currently unused); metrics port 8080.
  - Table of technical interfaces: Interface | Protocol | Direction | Frequency | Purpose. Rows: Git fetch (SSH, inbound to GitHub, 1m), Artifact consumption (HTTP, internal), Reconciliation apply (kube-apiserver, internal, 10m), Metrics (HTTP 8080, inbound, scrape), Webhook (HTTP 9292, inbound, unused — GAP-001/GAP-012).
  - **Mapping Input/Output to Channels** table: Input (Git commit on `main`) → Channel (SSH fetch by source-controller) → Output (artifact revision) → Downstream (kustomize-controller apply). Include the "no webhook push channel" note.

**Tables/Diagrams:** 2 Mermaid flowcharts + 2 tables (actors, interfaces) + 1 mapping table.

**Requirements to reflect:** REQ-002, REQ-003, REQ-010, CON-005, CON-006, CON-008, NFR-002, GAP-001, GAP-012.

---

### 4.4 `docs/arc42/04_solution_strategy.md`

**Template disposition:** Keep the `# Solution Strategy` heading; add subsections (template file currently has none).

**Outline:**
```
# Solution Strategy
## GitOps as the Core Strategy
## Flux v2 as the GitOps Engine
## Repository Layout and Multi-Cluster Strategy
## Bootstrap Approach
## Security Posture
## Evolution Path (Current Gaps)
```

**Key content points:**
- **GitOps as the Core Strategy:** Git = single source of truth (NFR-001); declarative desired state; pull-based reconciliation; OpenGitOps principles (link in references). No manual `kubectl apply` for managed resources.
- **Flux v2 as the GitOps Engine:** Why Flux (mature, kustomize-native, CNCF); the four controllers and their roles (F1); version pinning (NFR-009); reference ADR-001.
- **Repository Layout and Multi-Cluster Strategy:** `clusters/<name>/flux-system/` layout (F12); per-cluster isolation (NFR-011, REQ-004); identical components + cluster-specific sync path (F5); reference ADR-005.
- **Bootstrap Approach:** `flux bootstrap git` flow; kustomize composition of `gotk-components.yaml` + `gotk-sync.yaml` (REQ-009); reproducibility from a fresh cluster (NFR-003); reference ADR-004.
- **Security Posture:** SSH deploy keys (REQ-010, NFR-005, ADR-002); NetworkPolicies (REQ-006, NFR-006); pod security labels (REQ-005, NFR-007); RBAC (REQ-008, NFR-004); ResourceQuota (REQ-007, NFR-008). Honest note: SSH key stored as plain Secret (GAP-005).
- **Evolution Path (Current Gaps):** short bullet list of what's next, each with GAP ref: workloads (GAP-003), observability (GAP-004), CI/CD (GAP-010), secrets (GAP-005), webhook (GAP-001), backup (GAP-002), multi-tenancy (GAP-007). Explicit statement: these are planned, not implemented.

**Tables/Diagrams:** 1 table mapping strategy element → implementation → requirement/ADR refs. No diagram required (context/building-block views carry the diagrams), but a small Mermaid flowchart of "commit → reconcile → apply" is acceptable if kept simple.

**Requirements to reflect:** REQ-001…REQ-010, NFR-001…NFR-011, CON-001…CON-010, GAP-001…GAP-007, GAP-010; ADR-001…ADR-010 (see §4.9).

---

### 4.5 `docs/arc42/05_building_block_view.md`

**Template disposition:** Keep `## Whitebox Overall System`; replace the `<Name black box 1..n>` placeholders with real black boxes; **remove** the generic `## Level 2` / `## Level 3` template sections and replace with a single `## Level 2 — Reconciliation Pipeline` (Level 3 is unnecessary for this system — over-documentation risk). Remove the `<Name interface 1..m>` placeholder block and replace with a real `## Important Interfaces` table.

**Outline:**
```
# Building Block View
## Whitebox Overall System
### Contained Building Blocks
### Important Interfaces
## Black Boxes
### source-controller
### kustomize-controller
### helm-controller
### notification-controller
### flux-system Namespace & Security Primitives
### GitRepository (flux-system)
### Kustomization (flux-system)
## Level 2 — Reconciliation Pipeline
### White Box: Source Pipeline (source-controller)
### White Box: Apply Pipeline (kustomize-controller)
```

**Key content points:**
- **Whitebox Overall System:** Mermaid flowchart showing: GitHub repo → source-controller → artifact → kustomize-controller → kube-apiserver → cluster resources; helm-controller (idle); notification-controller (idle); all inside `flux-system` namespace boundary. Motivation paragraph; contained building blocks list; important interfaces summary.
- **Black boxes** — each with: Purpose/Responsibility, Interface(s), Quality/Performance characteristics, Directory/File location, Fulfilled requirements, Open issues:
  - `source-controller` — fetches Git (1m), produces artifact; interfaces: SSH fetch, artifact HTTP; file: `gotk-components.yaml`; REQ-001, REQ-002, REQ-010.
  - `kustomize-controller` — reconciles Kustomization (10m), applies + prunes; REQ-001, REQ-003, REQ-009; NFR-002.
  - `helm-controller` — installed, no HelmReleases (CON-007); REQ-001; note: idle.
  - `notification-controller` — installed, webhook listener 9292, no Receivers (GAP-012); REQ-001, REQ-006 (allow-webhooks policy); note: idle.
  - `flux-system Namespace & Security Primitives` — namespace labels (F6), NetworkPolicies (F7), ResourceQuota (F8), RBAC (F9); REQ-005…REQ-008.
  - `GitRepository (flux-system)` — the source object (F2); REQ-002.
  - `Kustomization (flux-system)` — the apply object (F3); REQ-003.
- **Important Interfaces:** table: Interface | From | To | Protocol/Port | Purpose. Rows: SSH Git fetch, artifact HTTP (source-controller svc :80), metrics (:8080), webhook (:9292, unused), kube-apiserver (RBAC).
- **Level 2 — Reconciliation Pipeline:** two white boxes describing internals: (a) source pipeline: clone → archive → artifact + revision; (b) apply pipeline: fetch artifact → kustomize build → diff → apply → prune → status. Keep textual, no nested diagrams.

**Tables/Diagrams:** 1 Mermaid flowchart (whitebox overall) + 1 interfaces table. Black boxes as structured text.

**Requirements to reflect:** REQ-001…REQ-010, NFR-002, CON-007, GAP-012.

---

### 4.6 `docs/arc42/06_runtime_view.md`

**Template disposition:** Replace `<Runtime Scenario 1..n>` placeholders with three named scenarios. Keep the `# Runtime View` heading.

**Outline:**
```
# Runtime View
## Scenario 1: Initial Bootstrap
## Scenario 2: Steady-State Reconciliation
## Scenario 3: Drift Detection and Correction
```

**Key content points:**
- **Scenario 1: Initial Bootstrap** — Mermaid sequence diagram: Operator runs `flux bootstrap git` → flux CLI creates namespace + applies `kustomization.yaml` → controllers start → GitRepository created → source-controller fetches (SSH) → artifact ready → Kustomization applies manifests → cluster converges. Note: bootstrap is the only human-triggered operation (NFR-004).
- **Scenario 2: Steady-State Reconciliation** — Mermaid sequence diagram: source-controller polls Git every 1m → new commit detected → new artifact revision → kustomize-controller (every 10m) fetches artifact → kustomize build → diff vs live state → apply changes → update status conditions. Note the 1m/10m cadence (F2, F3) and that without a webhook the max latency for a change is ~10m (GAP-001).
- **Scenario 3: Drift Detection and Correction** — Mermaid sequence diagram: (a) manual `kubectl edit` of a Flux-managed resource → kustomize-controller at next reconcile detects drift → reverts to Git state; (b) resource deleted from Git → next reconcile prunes it from cluster (prune=true). Reference NFR-002 acceptance criteria.
- Optional short paragraph: failure handling (Git unreachable → source-controller reports `Ready=False`, kustomize-controller keeps last-known-good state; no alerting yet — GAP-004).

**Tables/Diagrams:** 3 Mermaid sequence diagrams + short textual descriptions per scenario.

**Requirements to reflect:** REQ-002, REQ-003, NFR-002, NFR-003, NFR-004, GAP-001, GAP-004.

---

### 4.7 `docs/arc42/07_deployment_view.md`

**Template disposition:** Keep `## Infrastructure Level 1` and `## Infrastructure Level 2`. Replace `<Overview Diagram>`, `<Infrastructure Element 1..n>` placeholders with real elements. Rename Level 2 elements to: `### Kind Cluster (Development)`, `### Homeserver Cluster (Production)`, `### Flux Controllers (per cluster)`, `### GitHub Repository (external)`.

**Outline:**
```
# Deployment View
## Infrastructure Level 1
### Overview Diagram
### Motivation
### Quality and/or Performance Features
### Mapping of Building Blocks to Infrastructure
## Infrastructure Level 2
### Kind Cluster (Development)
### Homeserver Cluster (Production)
### Flux Controllers (per cluster)
### GitHub Repository (external)
```

**Key content points:**
- **Infrastructure Level 1:** Mermaid diagram: GitHub (remote) → two clusters (kind = dev, homeserver-cluster = prod); each cluster contains `flux-system` namespace with 4 controller Deployments; SSH deploy keys connect clusters to GitHub.
  - Motivation: two independent environments, no cross-contamination (NFR-011, REQ-004).
  - Quality/performance: reconciliation intervals (1m/10m), ResourceQuota (F8), identical component manifests (F5).
  - Mapping table: Building block | Cluster | Location. Rows: source-controller, kustomize-controller, helm-controller, notification-controller → both clusters → `flux-system` namespace; GitRepository/Kustomization → both clusters → `flux-system` namespace.
- **Infrastructure Level 2:**
  - `Kind Cluster (Development)` — local Kubernetes-in-Docker; purpose (test changes before prod); note: kind config not versioned in repo (GAP-013).
  - `Homeserver Cluster (Production)` — long-lived persistent cluster (assumption 5); purpose.
  - `Flux Controllers (per cluster)` — deployment topology: 4 Deployments, Services (F10), metrics port 8080, webhook port 9292 (unused).
  - `GitHub Repository (external)` — hosting, SSH access, deploy key (CON-006, REQ-010).

**Tables/Diagrams:** 1 Mermaid diagram (Level 1) + 1 mapping table. Level 2 elements as text with small inline notes.

**Requirements to reflect:** REQ-004, REQ-010, NFR-011, CON-004, CON-006, GAP-013, assumptions 4–5.

---

### 4.8 `docs/arc42/08_concepts.md`

**Template disposition:** Replace `<Concept 1..n>` placeholders with named concept sections. Keep the `# Cross-cutting Concepts` heading.

**Outline:**
```
# Cross-cutting Concepts
## GitOps and Declarative State Management
## Pull-Based Reconciliation and Drift Correction
## Secrets Management (SSH Deploy Keys)
## Network Security (NetworkPolicies)
## Pod Security Standards
## Resource Management and Quotas
## RBAC and Least Privilege
## Multi-Cluster Strategy
## Version Pinning and Upgrade Strategy
## Observability (Current State and Gap)
## Auditability and Traceability
```

**Key content points:** Each concept = short explanation + "How it is implemented here" (with manifest facts) + "Related requirements" + "Current gap/limitation" where applicable:
- GitOps & declarative state: NFR-001; Git as single source of truth.
- Pull-based reconciliation & drift: NFR-002; 1m/10m intervals, prune=true (F2, F3); no push channel (GAP-001).
- Secrets management: REQ-010, NFR-005; SSH deploy key as plain Secret `flux-system`; rotation implications (GAP-005).
- Network security: REQ-006, NFR-006; the three NetworkPolicies (F7) explained (egress default, scraping allowlist 8080, webhook allowlist 9292).
- Pod security: REQ-005, NFR-007; warn-level restricted labels (F6); baseline for future namespaces.
- Resource management: REQ-007, NFR-008; ResourceQuota scoped to critical priority classes (F8).
- RBAC: REQ-008, NFR-004; cluster-admin for reconcilers (F9), aggregated edit/view roles; human access minimal.
- Multi-cluster: REQ-004, NFR-011; per-cluster paths and sync (F5).
- Version pinning & upgrades: NFR-009; pinned versions (F1); no documented upgrade procedure (GAP-008).
- Observability: metrics on 8080 exist but no ServiceMonitors/dashboards/alerting (GAP-004); notification-controller unused (GAP-012).
- Auditability: NFR-010; Git history as audit trail.

**Tables/Diagrams:** No diagrams required. Optionally a small table per concept is acceptable; prefer prose + requirement refs to keep it readable.

**Requirements to reflect:** REQ-005…REQ-010, NFR-001…NFR-011, GAP-001, GAP-004, GAP-005, GAP-008, GAP-012.

---

### 4.9 `docs/arc42/09_architecture_decisions.md`

**Template disposition:** Keep the `# Architecture Decisions` heading; add ADR subsections. Use a consistent ADR format: **Status / Context / Decision / Consequences** (short, 3–6 sentences each).

**Outline:**
```
# Architecture Decisions
## ADR-001: Use Flux v2 for GitOps
## ADR-002: Use SSH Deploy Keys for Git Authentication
## ADR-003: Use Polling-Based Reconciliation (No Webhook Receiver)
## ADR-004: Use Kustomize for Bootstrap Manifest Composition
## ADR-005: Per-Cluster Bootstrap Directories
## ADR-006: Enforce Restricted Pod Security Warnings on flux-system
## ADR-007: Grant cluster-admin to kustomize-controller and helm-controller
## ADR-008: Store SSH Key as Plain Kubernetes Secret (No External Secret Manager)
## ADR-009: Track Only the main Branch (Single-Branch Strategy)
## ADR-010: Install helm-controller Despite No Current HelmReleases
```

**Key content points:**
- ADR-001: Flux v2 vs Argo CD vs manual kubectl → Flux chosen (kustomize-native, bootstrap tooling, CNCF). Refs: REQ-001, CON-001.
- ADR-002: SSH deploy key vs HTTPS PAT → SSH (read-only deploy key, no token in cluster). Refs: REQ-010, NFR-005, CON-006.
- ADR-003: Polling (1m/10m) vs webhook → polling chosen for simplicity; trade-off: up to 10m latency; webhook Receiver possible later. Refs: CON-008, GAP-001.
- ADR-004: Kustomize v1beta1 vs Helm for bootstrap → kustomize (Flux-native, no chart dependency). Refs: REQ-009, CON-003.
- ADR-005: `clusters/<name>/flux-system` layout → per-cluster isolation. Refs: REQ-004, NFR-011, CON-004.
- ADR-006: Pod security warn-level restricted labels → non-blocking warnings, baseline for future. Refs: REQ-005, NFR-007.
- ADR-007: cluster-admin for reconcilers → Flux default, required for full reconciliation; mitigated by Git-only operations. Refs: REQ-008, NFR-004.
- ADR-008: Plain Secret for SSH key → accepted for now; known gap (GAP-005); rotation procedure needed.
- ADR-009: Single `main` branch → simplicity; no promotion flow (GAP-006).
- ADR-010: helm-controller installed proactively → future HelmReleases; currently idle (CON-007).

**Tables/Diagrams:** None required. Each ADR as structured text. Optionally a summary table (ADR # | Decision | Status | Refs) at the top.

**Requirements to reflect:** REQ-001…REQ-010, NFR-004, NFR-005, NFR-007, NFR-011, CON-001, CON-003, CON-004, CON-006, CON-007, CON-008, GAP-001, GAP-005, GAP-006.

---

### 4.10 `docs/arc42/10_quality_requirements.md`

**Template disposition:** Keep `## Quality Requirements Overview` and `## Quality Scenarios`. Fill both with real content.

**Outline:**
```
# Quality Requirements
## Quality Requirements Overview
### Quality Tree
### Quality Requirements Table
## Quality Scenarios
```

**Key content points:**
- **Quality Tree:** Mermaid flowchart (or nested list) with root "Quality" → branches: Reliability (drift detection, self-healing, quotas), Security (network segmentation, pod security, secrets, RBAC), Maintainability (version pinning, reproducibility), Auditability (Git traceability), Reproducibility (bootstrap from scratch). Leaf nodes map to NFR IDs.
- **Quality Requirements Table:** NFR ID | Category | Requirement | Priority. Rows NFR-001…NFR-011 (skip NFR-012 or include as "known gap" row).
- **Quality Scenarios:** arc42-style scenario table (Stimulus | Source | Environment | Artifact | Response | Response measure) for 6 scenarios:
  1. Drift correction — someone edits a managed resource → reverted within 10m (NFR-002).
  2. Resource deleted from Git → pruned from cluster (NFR-002).
  3. Fresh cluster bootstrap → identical Flux deployment (NFR-003).
  4. SSH key rotation → GitRepository still fetches (NFR-005).
  5. Flux upgrade → pinned versions updated via Git commit (NFR-009).
  6. Audit query — who changed X → Git history (NFR-010).

**Tables/Diagrams:** 1 Mermaid quality tree + 1 quality requirements table + 1 scenario table.

**Requirements to reflect:** NFR-001…NFR-012, plus acceptance criteria from `specs/REQUIREMENTS.md` §5 (NFR-002, NFR-003).

---

### 4.11 `docs/arc42/11_technical_risks.md`

**Template disposition:** Keep the `# Risks and Technical Debts` heading; add subsections (template file currently has none).

**Outline:**
```
# Risks and Technical Debts
## Risk Overview
## Technical Debts
## Mitigation Roadmap
```

**Key content points:**
- **Risk Overview:** table of GAP-001…GAP-014 from `specs/REQUIREMENTS.md` §6: Gap ID | Description | Impact | Priority | Mitigation (short). Keep the exact GAP IDs and priorities from the baseline.
- **Technical Debts:** prose bullets for: empty `infrastructure/` scaffolding (F13), empty gitignored `temp/` (F14, GAP-009), notification-controller installed but unused (GAP-012), no CI/CD validation (GAP-010), no documented bootstrap procedure (GAP-011).
- **Mitigation Roadmap:** prioritized list:
  - High: backup/DR (GAP-002), external secrets (GAP-005), CI/CD validation (GAP-010).
  - Medium: webhook receiver (GAP-001), observability (GAP-004), multi-tenancy (GAP-007), bootstrap docs (GAP-011), kind config (GAP-013), policy engine (GAP-014).
  - Low: upgrade procedure (GAP-008), temp dir (GAP-009).
  - Note: roadmap is aspirational; nothing beyond the current bootstrap is implemented.

**Tables/Diagrams:** 1 risk table (GAP-001…GAP-014) + roadmap list.

**Requirements to reflect:** GAP-001…GAP-014.

---

### 4.12 `docs/arc42/12_glossary.md`

**Template disposition:** Keep the `# Glossary` heading and table format. Replace `<Term-1>` / `<Term-2>` placeholder rows with real terms.

**Outline:**
```
# Glossary
| Term | Definition |
```

**Key content points — terms to define (grouped):**
- GitOps concepts: GitOps, single source of truth, declarative state, drift, reconciliation, prune, bootstrap.
- Flux concepts: Flux, GitRepository, Kustomization (Flux resource), Kustomize (kubectl tool), artifact, revision, source-controller, kustomize-controller, helm-controller, notification-controller, Receiver (webhook), interval.
- Kubernetes concepts: namespace, Deployment, ServiceAccount, ClusterRole/ClusterRoleBinding, NetworkPolicy, ResourceQuota, Pod Security Standards, priority class (system-node-critical / system-cluster-critical), kube-apiserver, kind (Kubernetes in Docker).
- Security concepts: SSH deploy key, secretRef, known_hosts.
- Repo-specific terms: `clusters/<cluster-name>/flux-system`, `gotk-components.yaml`, `gotk-sync.yaml`, `flux-system` namespace.

Each definition: 1–2 sentences, plain language. Keep consistent with usage in all other sections.

**Tables/Diagrams:** 1 glossary table (~25–35 rows).

**Requirements to reflect:** none directly (definitions must be consistent with terms used across REQ/NFR/CON/GAP).

---

## 5. Cross-Section Dependencies

| Dependency | Direction | Why |
|---|---|---|
| Glossary (12) → all sections | 12 authored first | All sections use terms defined in the glossary; definitions must be settled before prose is written |
| ADRs (09) → Solution Strategy (04) | 09 → 04 | Strategy references ADR-001…ADR-010 as justification |
| ADRs (09) → Concepts (08) | 09 → 08 | Concepts reference the decisions (e.g., ADR-002 in Secrets concept) |
| Requirements baseline → 01, 10, 11 | specs → 01/10/11 | REQ/NFR/GAP IDs are the cross-reference backbone |
| Building Block View (05) → Runtime View (06) | 05 → 06 | Runtime scenarios use the component names/interfaces defined in 05 |
| Building Block View (05) → Deployment View (07) | 05 → 07 | Deployment maps the building blocks from 05 onto infrastructure |
| Ground-truth facts (F1–F18) → all sections | facts → all | Every factual claim must trace to a manifest fact; keep the fact table as the single source of truth |
| README.md → 01 | README → 01 | README links to arc42; optionally update README after docs are written (out of scope unless requested) |

**Consistency rules:**
- Component versions, intervals, URLs, ports, and names must be identical across all sections (use the fact table F1–F18).
- Requirement IDs must be spelled exactly as in `specs/REQUIREMENTS.md` (e.g., `REQ-002`, `NFR-002`, `CON-008`, `GAP-001`).
- ADR numbers referenced in 04/08 must match the ADR list in 09.

---

## 6. Suggested Authoring Order

Write files in this order (each step's output feeds the next):

1. **12_glossary.md** — settle terminology first (all other sections depend on it).
2. **01_introduction_and_goals.md** — overview, quality goals, stakeholders (sets the narrative).
3. **02_architecture_constraints.md** — constraints & conventions (boundaries for everything else).
4. **03_context_and_scope.md** — context diagrams (defines the system boundary).
5. **09_architecture_decisions.md** — ADRs (decisions justify the strategy).
6. **04_solution_strategy.md** — strategy (references ADRs + constraints).
7. **05_building_block_view.md** — component inventory (needed by runtime + deployment views).
8. **06_runtime_view.md** — runtime scenarios (uses component names from 05).
9. **07_deployment_view.md** — deployment (maps building blocks from 05 onto clusters).
10. **08_concepts.md** — cross-cutting concepts (references ADRs + NFRs).
11. **10_quality_requirements.md** — quality tree/scenarios (from NFRs).
12. **11_technical_risks.md** — risks & debts (from GAPs; write last so it can reference the final state of all other sections).

**Rationale:** glossary-first avoids terminology drift; ADRs before strategy avoids strategy/decision contradictions; views after strategy so diagrams match the described approach; risks last so the "current state" claims in risks match the finished docs.

---

## 7. Risks & Guardrails

| Risk | Guardrail |
|---|---|
| **Over-claiming features** (webhook receiver, backup, observability, workloads, CI/CD, external secrets) | Every section must state the current state honestly; use the GAP IDs when mentioning missing features; never write "we have X" for anything not in the fact table |
| **Version/interval drift** as repo evolves | All versions/intervals/URLs must come from the fact table F1–F18; add a "Last verified: 2026-09-15" note in 01 or a comment in each file header; recommend re-verifying facts on future doc updates |
| **Over-documentation** (Level 3 building blocks, excessive diagrams) | Keep Level 3 out of 05; keep diagrams simple and correct; 3 runtime scenarios max; no diagram for its own sake |
| **Documenting removed podinfo workloads** | Do not mention podinfo as current; it was added and removed in git history (F15); at most a one-line historical note in 11 if useful |
| **Empty scaffolding misread as implemented** (`infrastructure/`, `temp/`) | 11 must describe them as empty/placeholder (F13, F14); 04's evolution path must mark them as planned |
| **Invalid Mermaid syntax** | Keep diagrams minimal (flowchart/sequence only); verify rendering; prefer tables over complex diagrams |
| **Inconsistent requirement IDs** | Use exact IDs from `specs/REQUIREMENTS.md`; QA checklist verifies cross-references |
| **Secrets leaking into docs** | Never include actual key material, URLs with credentials, or `known_hosts` content; only reference secret names |
| **Template placeholders left behind** | QA checklist greps for `<Role-1>`, `<Diagram or Table>`, `*\<text explanation\>*`, `<Concept 1>`, `<Runtime Scenario`, `<Name black box`, `<Infrastructure Element`, `<Term-1>` |

---

## 8. QA Checklist (for the code-reviewer)

Run this checklist against the finished docs:

**Completeness**
- [ ] All 12 files in `docs/arc42/` are filled; no arc42 template placeholder text remains (grep for `<Role-`, `<Term-`, `<Diagram or Table>`, `<optionally`, `<Name black box`, `<Runtime Scenario`, `<Concept`, `<Infrastructure Element`, `*\<text explanation\>*`, `…​`).
- [ ] Each file follows the outline specified in §4 of this plan (headings kept/renamed/removed as specified).
- [ ] `docs/plans/PLAN.md` exists and matches the executed work.

**Factual accuracy (against fact table F1–F18)**
- [ ] Flux v2.9.5 and component versions (source-controller v1.9.5, kustomize-controller v1.9.5, helm-controller v1.6.4, notification-controller v1.9.4) are correct everywhere.
- [ ] GitRepository: interval 1m0s, branch `main`, SSH URL, secretRef `flux-system` — correct everywhere.
- [ ] Kustomization: interval 10m0s, `prune: true`, path `./clusters/<cluster-name>` — correct everywhere.
- [ ] NetworkPolicies (allow-egress, allow-scraping 8080, allow-webhooks 9292), ResourceQuota (pods 1000, critical priority classes), RBAC names (cluster-reconciler-flux-system, crd-controller-flux-system, flux-edit-flux-system, flux-view-flux-system) — correct everywhere.
- [ ] No claim of webhook receiver, backup, observability, workloads, CI/CD, or external secrets anywhere in the docs.

**Requirement traceability**
- [ ] REQ/NFR/CON/GAP IDs are spelled exactly as in `specs/REQUIREMENTS.md` and used in the sections specified in §4.
- [ ] ADR numbers referenced in 04/08 match the ADR list in 09.

**Diagrams**
- [ ] All Mermaid diagrams render (valid syntax); flowcharts/sequences are simple and correct.
- [ ] Diagrams match the text (no diagram contradicts the fact table).

**Consistency & quality**
- [ ] Terminology matches the glossary (12).
- [ ] Cross-file links (relative markdown links) resolve.
- [ ] No secrets, keys, or credential material in any file.
- [ ] Tone is honest about the early-stage state; "planned/not implemented" language used for gaps.
- [ ] Markdown formatting is consistent (heading levels, tables, code blocks).

---

*End of plan. Authoring order, per-file outlines, fact table, and QA checklist are the contract for the developer agent.*