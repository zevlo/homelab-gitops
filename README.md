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

## Current state (2026-07-28)

| App | Path | Sync | Manages |
|-----|------|------|---------|
| `root` | `apps/` | auto-sync + self-heal | all child Applications (app-of-apps) |
| `metallb-config` | `metallb/` | auto-sync + self-heal | IPAddressPool + L2Advertisement (adopted from Phase F) |
| `observability-secrets` | `observability/` | auto-sync + self-heal · wave −1 | sealed Grafana admin password |
| `kube-prometheus-stack` | chart `prometheus-community/kube-prometheus-stack` 87.21.0 | auto-sync + self-heal · wave 0 | Prometheus + Grafana (.242) + Alertmanager + node-exporter + kube-state-metrics |

Bootstrap layer (hand-installed, **not** Git-managed — Stage 5 deferred): ArgoCD v3.4.5, Sealed Secrets v0.38.4, **and the kube-prometheus-stack CRDs** (`kubectl apply --server-side` once — ArgoCD can't apply them via client-side; `skipCrds: true` is set on the kps app).

> **Grafana:** `http://192.168.1.242` · admin password = sealed `grafana-admin` secret. Datasources: Prometheus, Alertmanager.

### Making a change

```bash
# edit a manifest, e.g. metallb/pool.yaml
git add -A && git commit -m "..." && git push
# ArgoCD auto-syncs within ~3 min (natural poll); force a fetch with:
argocd app get <app> --refresh
```

### Adding a new app

1. Put manifests in a new top-level dir (e.g. `observability/`).
2. Add `apps/<name>.yaml` — copy `apps/metallb-config.yaml`, change `name` + `path` + `destination.namespace`.
3. Commit + push. `root` auto-creates the child app and syncs it.

### Sealing a secret (run from the ops workstation)

```bash
kubeseal --format yaml < my-secret.yaml > <dir>/<name>.yaml   # controller cert fetched via kubeconfig
git add <dir>/<name>.yaml && git commit -m "..." && git push
```

## Next

Phase H (observability: kube-prometheus-stack) lands here as a Helm-values-driven app under `helm/`. See [`cluster-build.md`](../dev/cluster-build.md) Phase H.
