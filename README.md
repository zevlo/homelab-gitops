# homelab-gitops

Source of truth for the K3s homelab cluster. ArgoCD reconciles cluster state from this repo.

Wraps [`cluster-build.md`](../dev/cluster-build.md) **Phase G — GitOps foundation**.

## Structure

| Path | Purpose |
|------|---------|
| `apps/` | ArgoCD `Application` manifests. `root.yaml` is the app-of-apps entry point; each child app points at a path below. |
| `metallb/` | MetalLB config (IPAddressPool + L2Advertisement) — managed by the `metallb-config` app. |
| `helm/` | Helm values overrides (observability stack lands here in Phase H). |
| `manifests/` | Raw K8s manifests not yet under a dedicated app. |

## Bootstrap model (chicken-and-egg)

ArgoCD can't manage its own first install. Resolution: a **manual bootstrap layer**, then ArgoCD adopts (and owns) everything.

1. ArgoCD v3.4.5 + Sealed Secrets v0.38.4 installed manually (`kubectl apply`).
2. The `root` Application (in `apps/root.yaml`) is applied manually once → ArgoCD reconciles all child apps from then on.
3. (Stage 5) `argocd` + `sealed-secrets` become child apps so ArgoCD reconciles its own existence — auto-sync ON, **self-heal OFF** (self-managing ArgoCD with self-heal is a known footgun).

## Repo access

ArgoCD reads this private repo over SSH using a **GitHub deploy key** (read-only, scoped to this repo). The deploy private key lives in an ArgoCD `repository` Secret, applied manually at bootstrap (the one manifest deliberately not stored in Git).

## Conventions

- Cluster changes get committed, not `kubectl apply`-ed ad hoc.
- Secrets are Sealed Secrets — encrypted at rest in this repo via `kubeseal`.
- Config/workload apps use auto-sync + self-heal; self-management apps (ArgoCD, sealed-secrets) use auto-sync only.
