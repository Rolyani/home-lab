# home-lab

 GitOps-managed Kubernetes homelab running on [k3s](https://k3s.io/) and reconciled by [Flux CD](https://fluxcd.io/). 
 Everything in the cluster — controllers, monitoring, and applications — is declared in this repository and applied automatically; there is no `kubectl apply` in the day-to-day workflow.

 The cluster hosts real workloads (a multi-service blog, a bookmark manager, a dashboard) on top of a production-style platform layer: a managed Postgres operator, replicated block storage, automated dependency updates, a full metrics stack, and zero-trust ingress.

 > **Status:** single `staging` environment, actively used. Treated as a real platform (GitOps, encrypted secrets, monitoring, and backups-capable storage), but it runs on a single-node footprint and is still a personal learning environment. Expect things to change.

## Architecture

 ```
                     Internet
                        │
               Cloudflare Tunnel            (no inbound ports opened)
                        │
         ┌──────────────┴──────────────┐
         │           k3s cluster        │
         │                              │
         │  Flux CD ──reconciles──┐     │
         │                        │     │
         │  infrastructure/  monitoring/  apps/
         │   • CloudNativePG    • kube-     • blog (Go + Nuxt + Laravel)
         │   • Longhorn           prometheus  • linkding
         │   • Renovate           -stack      • homepage
         │                                     │
         └──────────────────────────────────────┘
                        │
                   Git (this repo) = source of truth
 ```

## Tech stack

 | Concern            | Choice                       | Notes |
 |--------------------|------------------------------|------------------------------------------------------------------|
 | Cluster            | **k3s**                      | Lightweight, single-node |
 | GitOps             | **Flux CD**                  | Reconciles every 1m, `prune: true`|
 | Secrets            | **SOPS + age**               | Only `data`/`stringData` encrypted; decrypted in-cluster by Flux |
 | Database           | **CloudNativePG**            | Operator-managed Postgres, app credentials generated as Secrets  |
 | Storage            | **Longhorn**                 | Replicated block storage / PVCs|
 | Ingress            | **Cloudflare Tunnel**        | Outbound-only; no ports exposed on the home network|
 | Monitoring         | **kube-prometheus-stack**    | Prometheus, Grafana, Alertmanager|
 | Dependency updates | **Renovate** (self-hosted)   | Automated PRs for images and charts|

## Repository layout

 ```
 clusters/staging/          Flux entrypoints for the cluster
   flux-system/             Flux's own components (bootstrapped)
   infrastructure.yaml      Kustomization → infrastructure/controllers/staging  (wait: true)
   monitoring.yaml          Kustomizations → monitoring controllers + configs
   apps.yaml                Kustomization → apps/staging
   .sops.yaml               age recipient / encryption rules

 infrastructure/
   controllers/
     base/                  cnpg, longhorn, renovate
     staging/               overlay

 monitoring/
   controllers/base/        kube-prometheus-stack (operator + CRDs)
   configs/staging/         dashboards, scrape configs, alerting

 apps/
   base/                    blog, homepage, linkding (env-agnostic manifests)
   staging/                 per-environment overlays
 ```

The `base` / `staging` split is standard Kustomize: `base/` holds environment-agnostic manifests, `staging/` overlays environment-specific values and pins image tags.

## How it works

 1. **Flux watches this repo** (`flux-system` GitRepository) and reconciles the `clusters/staging` Kustomizations on a 1-minute interval.
 2. **Layered ordering** — infrastructure controllers reconcile with `wait: true` so dependents (e.g. CNPG-backed apps) only roll out once their operators are healthy.
 3. **Secrets stay encrypted in Git.** Files are encrypted with `age` per the rules in `.sops.yaml`; Flux decrypts them at apply time using the `sops-age` secret. Plaintext secrets never hit the repo.
 4. **Image updates flow through Renovate.** It opens PRs to bump container images and Helm charts; merging the PR is the deploy.

## Applications

 | App          | What it is                                          | Source |
 |--------------|-----------------------------------------------------|-----------------------------------------------------|
 | **blog**     | "Lessons in Progress" — Go API + Nuxt SSR + Laravel/Filament admin, backed by CNPG Postgres | [Rolyani/blog](https://lessons-in-progress.ian-naylor.com/) |
 | **linkding** | Self-hosted bookmark manager                        | upstream image |
 | **homepage** | Cluster/service dashboard                           | upstream image|

## Bootstrapping

 The cluster is bootstrapped with Flux pointed at this repository:

 ```bash
 flux bootstrap github \
   --owner=Rolyani \
   --repository=home-lab \
   --branch=main \
   --path=clusters/staging \
   --personal
 ```

Secrets management requires the cluster to hold the `age` private key as the `sops-age` secret in `flux-system`:

 ```bash
 cat age.agekey | kubectl create secret generic sops-age \
   --namespace=flux-system \
   --from-file=age.agekey=/dev/stdin
 ```

The public recipient lives in `clusters/staging/.sops.yaml`; encrypt files with `sops --encrypt --in-place <file> .yaml` before committing.

## Roadmap

 - [ ] Promote to a `production` overlay alongside `staging`
 - [ ] Configure and test CloudNativePG scheduled backups + restore
 - [ ] Multi-node k3s for real HA
 - [ ] Golden-signal Grafana dashboards + Alertmanager routes per app

## Disclaimer
A personal project for learning and professional development. Configurations may change without notice; use anything here at your own risk.
