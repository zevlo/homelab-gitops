# homelab-gitops

Source of truth for a 4-node K3s homelab cluster (3 Proxmox VMs + 1 GPU bare-metal node). ArgoCD reconciles the entire cluster from this repo — including ArgoCD itself.

## What this demonstrates

- **GitOps from the ground up** — app-of-apps pattern (`root` Application watches `apps/`, child apps own every layer), auto-sync + self-heal, ServerSideApply for large CRDs
- **Self-managed control plane** — ArgoCD and Sealed Secrets reconcile their own existence from Git (adopted from hand-installed bootstrap via no-op diff, sync-wave ordered, self-heal deliberately OFF for self-management)
- **Mixed-source apps** — raw manifests, vendored upstream installs pinned by version, and Helm charts fetched directly from chart repos with inline values
- **Secrets in a public-safe repo** — Sealed Secrets: ciphertext in Git, decryption keys never leave the cluster
- **Day-2 operations** — version upgrades = commit a new pinned manifest; disaster recovery = re-apply two bootstrap objects and wait

## Cluster at a glance

| | |
|---|---|
| Cluster | K3s v1.36 · 1 control-plane + 3 workers (2 Proxmox VMs + Dell bare-metal GPU node, RTX A2000) |
| GitOps | ArgoCD v3.4.5 (non-HA), app-of-apps, 9 Applications |
| Observability | kube-prometheus-stack 87.21.0 (Prometheus 14d retention, Grafana via LoadBalancer, Alertmanager) + NVIDIA DCGM exporter/ServiceMonitor/dashboard |
| Ingress/LB | MetalLB L2 |

## Structure

| Path | Purpose |
|------|---------|
| `apps/` | ArgoCD `Application` manifests. `root.yaml` is the app-of-apps entry point; each child app points at a path below (or a Helm chart repo). |
| `argocd/` | Vendored ArgoCD v3.4.5 install (pinned upstream manifest) — self-managed. |
| `sealed-secrets/` | Vendored sealed-secrets-controller v0.38.4 (pinned upstream manifest) — self-managed. |
| `metallb/` | MetalLB config (IPAddressPool + L2Advertisement). |
| `gpu/` | DCGM ServiceMonitor + Grafana dashboard for the NVIDIA GPU. |
| `observability/` | Sealed Grafana admin secret. |
| `helm/` | Helm values overrides (reserved). |
| `manifests/` | Raw K8s manifests not yet under a dedicated app. |

## Bootstrap model (chicken-and-egg)

ArgoCD can't manage its own first install. Resolution: a **minimal manual layer**, then ArgoCD adopts — and now owns — everything, including itself:

1. Install ArgoCD v3.4.5 manually once (`kubectl apply -n argocd -f argocd/install.yaml`).
2. Apply the repo-access Secret + `apps/root.yaml` manually once (`kubectl apply`).
3. `root` reconciles every child app from then on — including `argocd` and `sealed-secrets` (Stage 5: adoption was verified as a zero-drift no-op against the running cluster).
4. kube-prometheus-stack CRDs are applied once with `kubectl apply --server-side` — too large for ArgoCD's client-side apply (`skipCrds: true` on the app).

Self-management policy: `argocd` and `sealed-secrets` apps run auto-sync with **self-heal OFF** (self-managing ArgoCD with self-heal is a known footgun — the sync fights its own pod rolls). Everything else runs auto-sync + self-heal + prune.

Disaster recovery: install ArgoCD, apply the repo Secret + `root.yaml`, apply the kps CRDs, wait. The cluster rebuilds from Git.

## Repo access

ArgoCD reads this repo over SSH using a **GitHub deploy key** (read-only, scoped to this repo). The deploy private key lives in an ArgoCD `repository` Secret, applied manually at bootstrap (the one manifest deliberately not stored in Git).

## Current state (2026-08-26)

| App | Source | Wave | Sync | Manages |
|-----|--------|------|------|---------|
| `root` | `apps/` | — | auto + self-heal | all child Applications (app-of-apps) |
| `argocd` | `argocd/` (vendored v3.4.5) | −2 | auto-sync only | ArgoCD itself |
| `sealed-secrets` | `sealed-secrets/` (vendored v0.38.4) | −1 | auto-sync only | Sealed Secrets controller |
| `observability-secrets` | `observability/` | −1 | auto + self-heal | sealed Grafana admin password |
| `kube-prometheus-stack` | chart `prometheus-community/kube-prometheus-stack` 87.21.0 | 0 | auto + self-heal | Prometheus + Grafana + Alertmanager + node-exporter + kube-state-metrics |
| `metallb-config` | `metallb/` | 0 | auto + self-heal | IPAddressPool + L2Advertisement |
| `gpu-operator` | chart `nvidia/gpu-operator` v26.3.3 (host driver, operator-managed toolkit) | 0 | auto + self-heal | NVIDIA GPU Operator |
| `gpu-observability` | `gpu/` | 1 | auto + self-heal | DCGM ServiceMonitor + Grafana dashboard |

> **Grafana:** `http://192.168.1.242` · admin password = sealed `grafana-admin` secret (rotated 2026-08-26).

## Making a change

```bash
# edit a manifest, e.g. metallb/pool.yaml
git add -A && git commit -m "..." && git push
# ArgoCD auto-syncs within ~3 min (natural poll); force a fetch with:
argocd app get <app> --refresh
```

## Adding a new app

1. Put manifests in a new top-level dir (e.g. `observability/`).
2. Add `apps/<name>.yaml` — copy `apps/metallb-config.yaml`, change `name` + `path` + `destination.namespace`.
3. Commit + push. `root` auto-creates the child app and syncs it.

## Upgrading a vendored component

Replace the pinned manifest with the new version's upstream file (e.g. `argocd/install.yaml` from the new tag), commit, push. Expect a real diff on sync — review it in ArgoCD before it applies.

## Sealing a secret (run from the ops workstation)

```bash
kubectl create secret generic <name> --from-literal=key=value -n <ns> --dry-run=client -o yaml \
  | kubeseal --format yaml --context homelab > <dir>/<name>-sealedsecret.yaml
git add <dir>/<name>-sealedsecret.yaml && git commit -m "..." && git push
```

Ciphertext is safe in a public repo — only the in-cluster controller can decrypt. Rotate by generating a new value and re-sealing (Grafana's admin password was rotated this way before this repo went public).

## Roadmap

- Log aggregation (Loki attempted, dropped — revisit with a simpler single-binary design)
- Time-sliced GPU sharing for multi-tenant workloads

## Conventions

- Cluster changes get committed, not `kubectl apply`-ed ad hoc (helper pods for one-off fixes get deleted immediately after).
- Secrets are Sealed Secrets — encrypted at rest in this repo via `kubeseal`.
- Config/workload apps use auto-sync + self-heal; self-management apps (ArgoCD, sealed-secrets) use auto-sync only.
- Rollback of a self-management app: `argocd app delete <app> --cascade=false` — never cascade-delete a running ArgoCD or the SealedSecret CRD.
