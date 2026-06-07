# home-lab
![Validate](https://github.com/Rolyani/home-lab/actions/workflows/validate.yaml/badge.svg)
<img alt="gitleaks badge" src="https://img.shields.io/badge/protected%20by-gitleaks-blue">


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

## CI/CD & dependency management

Every change to the cluster flows through the same gate: a pull request to main that must pass four required checks before it can merge. Direct pushes to main are blocked, so the checks are a hard gate rather than advisory. The rule applies equally to my own PRs and to Renovate's automated ones.

Validation on every PR (.github/workflows/validate.yaml):
| Check | What it guards against |
|-------|------------------------|
| **Build & schema-validate overlays** | Renders each Kustomize overlay exactly as Flux would (`kustomize build`) and schema-validates the output with `kubeconform`, including CNPG / Longhorn / Prometheus / Flux CRDs via the community schema catalogue. Catches broken manifests before they reach the cluster. |
| **Verify Secrets are SOPS-encrypted** | Fails the build if any `kind: Secret` with a committed payload is not SOPS-encrypted. ServiceAccount-token Secrets (cluster-minted payload) are exempt. |
| **Scan for leaked secrets** | Runs `gitleaks` over current file contents and full git history. SOPS ciphertext is allowlisted; any other credential-shaped string fails the build. |
| **Check for deprecated API versions** | Runs `flux migrate` and fails if it would rewrite any manifest, catching deprecated Kubernetes/Flux apiVersions before they become breaking. |

The SOPS-encryption and gitleaks checks are complementary: the first guarantees the allowlisted files are actually encrypted, the second guarantees nothing leaks outside them. Together, they let this repo hold encrypted secrets in public safely. The manifest-validation approach follows the official Flux CI example [`fluxcd/flux2-kustomize-helm-example`](https://github.com/fluxcd/flux2-kustomize-helm-example); the encryption and secret-scanning checks are additions specific to running a public repo that stores encrypted secrets.

Automated updates (Renovate, self-hosted as a CronJob) keep images, Helm charts, and GitHub Actions current and feed every update through the same gate:

- **Low-risk updates** (minor, patch, digest) auto-merge **only after the four
  required checks pass** auto-merge is a delegation of trust to the CI
  pipeline, not a bypass of it.
- **Major updates** never auto-merge; they wait for manual approval via the
  Renovate **dependency dashboard**, which also surfaces everything Renovate is
  tracking or holding back in a single issue.
- **Supply-chain quarantine:** a `minimumReleaseAge` of 14 days means no release
  auto-merges until it has been public for two weeks, giving the ecosystem time
  to detect and pull a compromised release before it could reach the cluster.
- **Scheduling:** Renovate only opens PRs and auto-merges over the weekend, with
  merges cut off by Sunday evening, so an automated change never lands right
  before the work week, when there'd be no time to address a regression.
- **Pinned and reproducible:** GitHub Actions are pinned to commit SHAs (and
  kept updated by Renovate), so every workflow runs exactly the reviewed code.

The result is a single, consistent change-control loop: nothing reaches main unvalidated, low-risk updates flow in automatically once they're proven safe, risky ones wait for a human, and the whole dependency picture is visible in one place.

<img width="809" height="389" alt="image" src="https://github.com/user-attachments/assets/73c908e4-8b9c-4c9a-afa6-f2639cf3e762" />

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
