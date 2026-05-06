# Approach 3: App of Apps with Layered Overrides

An app-of-apps pattern for COTS applications that ship multiple Helm charts.
Two ArgoCD Applications run in sync waves — the first installs the vendor charts,
the second patches them for OpenShift using Server-Side Apply.

## The problem

COTS platforms often ship as 3+ Helm charts (frontend, API, worker, etc.).
Each chart needs the same OpenShift fixes (SCC-compliant security contexts,
Routes instead of Ingresses, resource limits). With a single ArgoCD Application
you can set Helm values, but you can't patch fields the chart doesn't expose
as values. With two separate Applications touching the same resources, you need
clear field ownership or they fight.

## How it works

```
┌────────────────────────────────┐
│  Root App (app of apps)        │
│  manages two child apps:       │
└──────┬───────────────┬─────────┘
       │               │
       ▼               ▼
┌──────────────┐ ┌──────────────────┐
│ Wave 0:      │ │ Wave 1:          │
│ base-install │ │ ocp-overrides    │
│              │ │                  │
│ 3 Helm charts│ │ SSA + Force      │
│ with values  │ │ patches over     │
│              │ │ base fields      │
│ ignoreDiffs  │ │                  │
│ on patched   │ │ Kustomize patches│
│ fields       │ │ for SCC, Routes, │
│              │ │ resources        │
└──────────────┘ └──────────────────┘
```

1. **Wave 0 — `base-install`**: Multi-source ArgoCD Application that installs
   three vendor Helm charts with OpenShift-safe values. Uses `ignoreDifferences`
   on fields that the overrides app will own.

2. **Wave 1 — `ocp-overrides`**: Directory of plain partial manifests that ArgoCD
   applies via SSA. Each manifest only contains the fields you want to override —
   the API server merges them into the existing resources. Uses `ServerSideApply=true`
   and `Force=true` to take field ownership. Includes retry logic for the race
   condition where wave 0's resources may not exist yet.

## Why two apps instead of one?

- **Separation of concerns** — the base values track what the vendor chart supports
  natively; the overrides handle what it doesn't
- **Different update cadences** — chart version bumps happen in wave 0, OpenShift
  policy changes happen in wave 1, independently
- **Clarity** — when something breaks after a chart upgrade, you know the base
  install changed; when an SCC policy changes, you know the overrides changed

## The SSA handshake

The two apps avoid fighting over fields through a two-sided agreement:

| App | Mechanism | Effect |
|---|---|---|
| `base-install` (wave 0) | `ignoreDifferences` on security context, resources, annotations | Stops re-syncing fields it doesn't own |
| `ocp-overrides` (wave 1) | `ServerSideApply=true` + `Force=true` | Takes ownership of those fields |

Without both sides, you get a sync loop where each app overwrites the other.

## Structure

```
app-of-apps-layered/
├── root/
│   ├── appofapps.yaml           # The root Application
│   ├── base-install.yaml        # Wave 0: three Helm charts
│   └── ocp-overrides.yaml       # Wave 1: SSA patches
├── base/
│   └── chart-values/
│       ├── frontend.yaml        # Helm values for the frontend chart
│       ├── api.yaml             # Helm values for the API chart
│       └── worker.yaml          # Helm values for the worker chart
└── overrides/                       # Plain partial manifests (no kustomization.yaml)
    ├── fix-scc-frontend.yaml        # ArgoCD applies these via SSA — only the fields
    ├── fix-scc-api.yaml             # present get merged into the existing resources
    ├── fix-scc-worker.yaml
    └── routes.yaml                  # OpenShift Route for the frontend
```
