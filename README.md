# COTS on OpenShift with GitOps

Patterns for deploying commercial off-the-shelf (COTS) applications on OpenShift
using ArgoCD when the vendor's Helm chart wasn't written with OpenShift in mind.

The most common issue: the chart's pods fail with SCC-related errors because the
chart assumes vanilla Kubernetes security defaults.

## Three approaches

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

## Which one should I use?

| | Helm with Kustomize | Kustomize with Helm | App of Apps Layered |
|---|---|---|---|
| ArgoCD source type | Multi-source | Single source (Kustomize) | App of apps (multi-source + Kustomize) |
| Helm lifecycle | ArgoCD manages it | Kustomize manages it | ArgoCD manages it |
| Number of charts | One | One | Multiple |
| Complexity | Moderate | Simple | More moving parts |
| ArgoCD Helm visibility | Full (chart version, values) | None (just sees Kustomize) | Full per chart |
| Field ownership | Kustomize patches | Kustomize patches | SSA with explicit ownership handoff |
| Best for | Single chart, Helm-first teams | Single chart, Kustomize-first teams | Multi-chart COTS platforms |
