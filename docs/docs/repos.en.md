# Repository Patterns

The `cmoreira-dev` GitHub org is split into categories — each top-level
folder in the development workspace groups related repositories, but **each
repository is independent**, with its own history, CI, and release cycle.

| Category | Contents |
|---|---|
| `infra-as-code/` | Terraform/Terragrunt — provisioned infrastructure |
| `platform-engineering/` | Cluster addons (`gitops.*`) and internal portal (Backstage) |
| `teupadel.com/` | Padel movement analysis application |
| `sara/` | Chords/lyrics practice application |
| `documentation/` | This site |

## Prefix convention

The repository name's prefix indicates what it is and how it's operated:

| Prefix | Meaning | How it's applied in the environment |
|---|---|---|
| `gitops.*` | ArgoCD-managed workload | Auto-discovered by `ApplicationSet/gitops-repos`, continuously synced |
| `iac.*` | Live infrastructure state (Terragrunt) | CI: `plan` on PR, `apply` on merge, nightly `drift` (see [Two tiers of IaC](iac/tiers.md)) |
| `api.*` | Backend service | Docker image built via CI, deployed via a matching `gitops.*` |
| `ui.*` | Frontend application | Same — built via CI, deployed via `gitops.*` |
| `docs.*` | Documentation site | Built locally with MkDocs |
| `backstage.*` | Internal portal (Backstage) | Deployed via `gitops.*` like any other app |

## Expected internal structure of a `gitops.<app>` repo

```
gitops.<app>/
├── argocd/
│   └── <app>.yaml         ← Application(s) — ArgoCD points here
├── helm/
│   ├── Chart.yaml          ← depends on gitops.generic-app-chart
│   └── values.yaml
├── catalog-info.yaml       ← Backstage registration
└── renovate.json           ← automatic chart version bumps
```

Two variations of this pattern coexist:

- **Own app** (`gitops.template`, `gitops.teupadel.com`,
  `gitops.local-sara`) — depends on the
  [generic chart](kubernetes/generic-app-chart.md), `argocd/` is normally
  empty (the `ApplicationSet` generates the Application on its own) — it's
  only populated manually when the repo delivers **more than one
  component** (e.g. api + ui in the same repo), as is the case for
  `gitops.teupadel.com` and `gitops.local-sara`.
- **Third-party addon** (`gitops.core-addons`, `gitops.ai-core-addons`,
  `gitops.cnpg`, `gitops.monitoring`, `gitops.headlamp`) —
  `helm/<addon>/Chart.yaml` depends directly on the upstream chart of the
  project (cert-manager, CNPG, Burrito, etc.), bypassing the generic chart.

## Application repos (`api.*` / `ui.*`)

Convention shared by both:

```
api.<app>/ or ui.<app>/
├── Dockerfile
├── CLAUDE.md          ← architecture/decision context for that component
├── README.md
└── (source code)
```

- **`api.*`**: Python 3.11 + FastAPI + Uvicorn is the current standard across
  all backend services.
- **`ui.*`**: Next.js (App Router) is the current standard; always built via
  Docker multi-stage (never a locally published `npm install`) — the
  `Dockerfile` sets production values via `ARG`, since the reusable CI
  workflow doesn't pass `--build-arg`.

## Workflow for a change

1. **Scope the change to the right category** — confirm which folder owns
   the file before touching it.
2. **Apply the change within its repo** — each subfolder is an independent
   Git repository.
3. **Update the documentation** — this site (`docs.cmoreira-dev`), if the
   change affects anything described here.
4. **Check cross-repo impact** — some important couplings:
    - `iac.homelab-live-infra` consumes `iac-proxmox-lxc` and
      `iac-aws-ecr-pipeline` via a Git `ref` — a module change may need a
      version bump in the consumer.
    - `api.*`/`ui.*` components of the same app share an API contract and,
      sometimes, the same `gitops.*` repo — a route/env var change on one
      side may require a matching change on the other.
    - `gitops.core-addons` is a dependency of practically every other
      `gitops.*` repo (Gateway API, cert-manager, External Secrets) —
      changes there have cluster-wide blast radius.

## General rules

- Never commit directly to `main` without flagging it — prefer branch + PR.
- `terraform apply` / `terragrunt apply` always under explicit confirmation.
- Direct `kubectl apply` isn't used in GitOps repos — ArgoCD is what applies.
