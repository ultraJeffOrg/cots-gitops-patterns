# Approach 4: Kustomize with Helm + JSON Patch

Same structure as [Approach 2](../kustomize-with-helm/) — Kustomize pulls in the
Helm chart and patches the output. The difference is the **type of patch**.

## Why this exists

Approach 2 uses strategic merge patches to fix Deployments. Strategic merge works
when you need to **add or update** fields. But COTS charts often set fields you
need to **remove**:

| Field the vendor sets | What you need | Strategic merge | JSON Patch |
|---|---|---|---|
| `runAsUser: 0` | Remove it entirely | Can't — merge only adds | `op: replace` on the parent object |
| `capabilities.add: [NET_BIND_SERVICE]` | Remove it entirely | Can't | `op: replace` on the parent object |
| `privileged: true` | Remove it | Can set `privileged: false`, but the field remains | `op: replace` clears everything |

JSON Patch (RFC 6902) has three operations that strategic merge doesn't:

- **`replace`** — overwrite a field or object entirely (old value is gone)
- **`remove`** — delete a field from the resource
- **`test`** — assert a value before patching (conditional patches)

## How it works

```yaml
# Strategic merge — ADDS runAsNonRoot but CAN'T remove runAsUser: 0
# Result: { runAsUser: 0, runAsNonRoot: true } ← SCC still rejects
patches:
  - target: { kind: Deployment, name: cots-app-api }
    patch: |
      spec:
        template:
          spec:
            securityContext:
              runAsNonRoot: true

# JSON Patch — REPLACES the entire securityContext
# Result: { runAsNonRoot: true, seccompProfile: ... } ← runAsUser: 0 is gone
patches:
  - target: { kind: Deployment, name: cots-app-api }
    patch: |
      - op: replace
        path: /spec/template/spec/securityContext
        value:
          runAsNonRoot: true
          seccompProfile:
            type: RuntimeDefault
```

## When to use this instead of Approach 2

Use Approach 2 (strategic merge) when the chart sets fields you just need to
override to a different value. Use this approach (JSON Patch) when the chart sets
fields you need to completely remove or replace wholesale.

In practice, most COTS-on-OpenShift fixes need JSON Patch because the insecure
fields (`runAsUser: 0`, `capabilities.add`, `privileged`) need to be gone, not
just overridden.

## Structure

```
kustomize-with-helm-jsonpatch/
├── kustomization.yaml           # Pulls Helm chart + JSON Patch (RFC 6902)
├── values-ocp.yaml              # Helm values for OpenShift
├── scc/
│   ├── cots-app-scc.yaml        # Custom SCC
│   └── scc-rolebinding.yaml     # Bind SCC to service account
└── argocd-application.yaml      # Single-source ArgoCD Application
```
