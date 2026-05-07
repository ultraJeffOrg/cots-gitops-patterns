# COTS on OpenShift with GitOps

Patterns for deploying commercial off-the-shelf (COTS) applications on OpenShift
using ArgoCD when the vendor's Helm chart wasn't written with OpenShift in mind.

The most common issue: the chart's pods fail with SCC-related errors because the
chart assumes vanilla Kubernetes security defaults.

## Four approaches

### [Approach 1: Helm with Kustomize](helm-with-kustomize/)
Helm is the primary source. ArgoCD uses multi-source to render the Helm chart and
apply Kustomize patches on top. Use this when you want ArgoCD to manage the Helm
chart lifecycle directly.

### [Approach 2: Kustomize with Helm](kustomize-with-helm/)
Kustomize is the primary source. It pulls in the Helm chart via `helmCharts`,
renders it, and applies patches — all in one `kustomization.yaml`. Simpler ArgoCD
Application config (single source).

### [Approach 3: App of Apps with Layered Overrides](app-of-apps-layered/)
An app-of-apps pattern for COTS platforms that ship multiple Helm charts. Two
child Applications run in sync waves — wave 0 installs the vendor charts, wave 1
patches them for OpenShift using Server-Side Apply with Force. The two apps use
`ignoreDifferences` + SSA to avoid fighting over field ownership.

### [Approach 4: Kustomize with Helm + JSON Patch](kustomize-with-helm-jsonpatch/)
Same as Approach 2 but uses JSON Patch (RFC 6902) instead of strategic merge
patches. This is the approach you need when the vendor chart sets fields you must
**remove** (`runAsUser: 0`, `capabilities.add`) — strategic merge can only add
or update fields, never remove them.

## Which one should I use?

| | Helm with Kustomize | Kustomize with Helm | App of Apps Layered | JSON Patch |
|---|---|---|---|---|
| ArgoCD source type | Multi-source | Single source | App of apps | Single source |
| Helm lifecycle | ArgoCD manages it | Kustomize manages it | ArgoCD manages it | Kustomize manages it |
| Number of charts | One | One | Multiple | One |
| Complexity | Moderate | Simple | Most complex | Simple |
| Can remove fields | No | No | Partially (SSA `null`) | **Yes** (`op: replace/remove`) |
| ArgoCD Helm visibility | Full | None | Full per chart | None |
| Best for | Single chart, Helm-first | Single chart, Kustomize-first | Multi-chart platforms | Charts with insecure defaults that need removal |

### Strategic merge vs JSON Patch — when does it matter?

If the vendor chart sets `runAsUser: 0` and you need it gone, strategic merge
**cannot help** — it can only add `runAsNonRoot: true` alongside the existing
`runAsUser: 0`, and SCC still rejects the pod. JSON Patch's `replace` operation
overwrites the entire `securityContext` object, removing everything the vendor set.

**Rule of thumb:** if you only need to add fields, use Approach 2 (strategic merge).
If you need to remove or replace fields, use Approach 4 (JSON Patch).
