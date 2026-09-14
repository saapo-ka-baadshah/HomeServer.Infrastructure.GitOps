# Glossary

*Last verified against repository state: 2026-09-15.*

This glossary defines the terminology used throughout the arc42 documentation. Definitions are kept consistent with the terms used in [specs/REQUIREMENTS.md](../../specs/REQUIREMENTS.md).

| Term | Definition |
|------|------------|
| **GitOps** | An operational model in which Git is the single source of truth for the desired state of a system; automated agents (here the Flux controllers) continuously reconcile the live system to that state. |
| **OpenGitOps** | A set of open-source standards and principles for GitOps, defining declarative, versioned, and pulled desired state. |
| **Single source of truth** | The principle that one authoritative location — the Git repository — holds the complete desired state of the cluster; no other source may define managed resources. |
| **Declarative state** | Desired state expressed as data (YAML manifests) that describes the end result rather than the steps to reach it. |
| **Drift** | The difference between the desired state declared in Git and the actual state of the cluster. |
| **Reconciliation** | The periodic process of comparing the desired state (Git) with the actual state (cluster) and applying changes to eliminate drift. |
| **Prune** | Flux's removal of cluster resources that exist in the cluster but are no longer declared in Git; enabled via `prune: true` on the Kustomization. |
| **Bootstrap** | The initial process of installing the Flux controllers and wiring them to a Git repository, performed with `flux bootstrap git`. |
| **`flux bootstrap git`** | The Flux CLI command that installs the controllers into a cluster and commits the bootstrap manifests to Git. |
| **Flux** | A CNCF-graduated GitOps toolkit for Kubernetes; the GitOps engine used in this repository (v2.9.5). |
| **CNCF** | The Cloud Native Computing Foundation; the organization that hosts and graduates Kubernetes and Flux. |
| **GitRepository** | A Flux custom resource (`source.toolkit.fluxcd.io/v1`) that defines a Git source to poll and from which artifacts are produced. |
| **Kustomization (Flux resource)** | A Flux custom resource (`kustomize.toolkit.fluxcd.io/v1`) that applies a set of manifests from a source artifact to the cluster. |
| **Kustomize (tool)** | A Kubernetes-native configuration management tool (`kustomize.config.k8s.io/v1beta1`) used to compose manifests; distinct from the Flux Kustomization resource. |
| **Artifact** | A compressed archive of a Git repository snapshot produced by source-controller and consumed by kustomize-controller. |
| **Artifact service** | The HTTP endpoint exposed by source-controller (Service `source-controller`, port 80) from which kustomize-controller downloads source artifacts. |
| **Revision** | A unique identifier of an artifact, typically `branch@sha1:<commit>`; recorded in the status of GitRepository and Kustomization resources. |
| **source-controller** | The Flux controller that fetches sources (Git) and produces artifacts; polls every 1m in this repository. |
| **kustomize-controller** | The Flux controller that reconciles Kustomization resources: builds, diffs, applies and prunes; runs every 10m here. |
| **helm-controller** | The Flux controller that reconciles HelmRelease resources; installed but currently idle (no HelmReleases exist). |
| **notification-controller** | The Flux controller that handles webhook receivers and dispatches notifications; installed with a webhook listener on port 9292, currently unused. |
| **Receiver (webhook)** | A Flux resource that exposes an HTTP endpoint so a Git host can push change notifications; not configured here (GAP-001). |
| **Interval** | The polling/reconciliation period of a Flux resource: 1m for the GitRepository, 10m for the Kustomization. |
| **Namespace** | A Kubernetes mechanism to scope and isolate resources within a cluster; the `flux-system` namespace hosts the Flux controllers. |
| **Deployment** | A Kubernetes workload resource that manages a set of replica Pods; each Flux controller runs as a Deployment. |
| **ServiceAccount** | A Kubernetes identity used by Pods to authenticate to the kube-apiserver. |
| **ClusterRole / ClusterRoleBinding** | RBAC resources that grant cluster-wide permissions to subjects (ServiceAccounts, users, groups). |
| **NetworkPolicy** | A Kubernetes resource that controls traffic to and from Pods at the network layer. |
| **ResourceQuota** | A Kubernetes resource that limits aggregate resource usage in a namespace. |
| **Pod Security Standards** | Kubernetes-defined security levels (privileged, baseline, restricted); the `flux-system` namespace carries `restricted` warn labels. |
| **Priority class** | A Kubernetes class that assigns scheduling priority; `system-node-critical` and `system-cluster-critical` scope the `critical-pods-flux-system` ResourceQuota. |
| **system-node-critical** | A Kubernetes priority class for pods essential to node operation; scopes the `critical-pods-flux-system` ResourceQuota. |
| **system-cluster-critical** | A Kubernetes priority class for pods essential to cluster operation; scopes the `critical-pods-flux-system` ResourceQuota. |
| **kube-apiserver** | The Kubernetes API server; the central control-plane component that the Flux controllers talk to. |
| **kind (Kubernetes in Docker)** | A tool for running local Kubernetes clusters in Docker containers; used for the development cluster. |
| **SSH deploy key** | A read-only SSH key registered with GitHub that grants access to a single repository; stored in a Kubernetes Secret. |
| **secretRef** | A field in a Flux resource that points to the Kubernetes Secret holding credentials (here the SSH deploy key). |
| **known_hosts** | SSH host-key verification data stored alongside the deploy key in the Secret. |
| **`clusters/<cluster-name>/flux-system`** | The per-cluster directory layout containing `gotk-components.yaml`, `gotk-sync.yaml` and `kustomization.yaml`. |
| **`kustomization.yaml`** | The Kustomize composition file that lists the resources to build for a directory; distinct from the Flux `Kustomization` resource (`kustomize.toolkit.fluxcd.io/v1`). |
| **`gotk-components.yaml`** | The Flux-generated manifest containing all controller components, RBAC, NetworkPolicies, ResourceQuota, Services and CRDs. |
| **`gotk-sync.yaml`** | The Flux-generated manifest containing the GitRepository and Kustomization resources. |
| **`flux-system` namespace** | The namespace hosting the Flux controllers and their resources in each cluster. |