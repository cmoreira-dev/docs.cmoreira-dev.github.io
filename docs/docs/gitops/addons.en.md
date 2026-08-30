# Addon catalog

Third-party addons follow the pattern described in
[Repository Patterns](../repos.md#expected-internal-structure-of-a-gitopsapp-repo):
`helm/<addon>/Chart.yaml` with `dependencies:` directly on the upstream
chart, bypassing the generic chart (that one is reserved for the org's own
apps).

| Repo | Addon(s) | What it does |
|---|---|---|
| `gitops.core-addons` | cert-manager, External Secrets Operator, NGINX Gateway Fabric (+ CRDs), cloudflared, nvidia-device-plugin, argocd-image-updater, **Burrito** | The cluster's base layer — practically every other `gitops.*` depends on something here (Gateway API, `ClusterSecretStore/aws-ssm`, TLS) |
| `gitops.ai-core-addons` | Ollama, LiteLLM | Local LLM serving: Ollama runs inference on the GPU, LiteLLM exposes an OpenAI-compatible proxy in front of it (`llm.cmoreira.dev`) |
| `gitops.cnpg` | CloudNativePG (Postgres operator) | Operator + first consumer database (Backstage) |
| `gitops.monitoring` | Grafana Alloy, metrics-server | Observability (telemetry collection) and metrics for HPA |
| `gitops.headlamp` | Headlamp | Web dashboard for the cluster (`headlamp.cmoreira.dev`), login via Microsoft Entra ID |
| `gitops.echoserver` | echoserver | Test endpoint for validating routing (Gateway API/HTTPRoute) |

## `gitops.core-addons` in detail

The only `gitops.*` repo that also carries its own **Terraform** stack,
organised into subfolders (each a standalone root with its own backend `key`):

- `terraform/ecr-pull-iam/` — IAM users for ECR image pull + `argocd-image-updater`
  (complements `iac-aws-ecr-pipeline` on the CI side).
- `terraform/burrito/` — datastore bucket + IAM users for the Burrito operator.
- `terraform/_modules/{s3,ssm-user}` — repo-local modules (see
  [Reusable modules](../iac/modules.md#_modules-in-gitopscore-addons)).

Addons installed:

- **cert-manager** — issues the TLS certificates used by the Gateways (see
  [Networking & Ingress](../architecture/networking.md))
- **External Secrets Operator** — via `ClusterSecretStore/aws-ssm`; see
  [Secrets & Security](../architecture/secrets.md)
- **NGINX Gateway Fabric** — the Gateway API implementation that routes all
  inbound traffic
- **cloudflared** — the in-cluster side of the Cloudflare tunnel
- **nvidia-device-plugin** — exposes the dedicated GPU worker's GPU as a
  schedulable resource
- **argocd-image-updater** — automatically updates the image tag in
  `gitops.*` repos when a new image lands in ECR
- **[Burrito](burrito.md)** — Tier-2 IaC: reconciles the OpenTofu/Terragrunt in
  GitOps repos' `terraform/` folders

## `gitops.ai-core-addons` in detail

Explicitly modeled after the `core-addons` pattern. Two components:

- **Ollama** (`otwld/ollama-helm`) — pinned to the GPU worker via
  `nodeSelector`, runs a local model (`llama3.1:8b`)
- **LiteLLM** (`BerriAI/litellm`) — OpenAI-compatible proxy in front of
  Ollama, exposed at `llm.cmoreira.dev`

Prerequisites documented in the repo itself: `core-addons`'s
`nvidia-device-plugin`, the manual `nvidia.com/gpu.present=true` label on
the GPU node (no automatic node discovery), and an SSM parameter
(`/homelab/litellm/master-key`) for LiteLLM to start.

## `gitops.headlamp` in detail

Federated login via Microsoft Entra ID (direct OIDC, not through
ArgoCD's Dex) — the same human-authentication pattern as ArgoCD, see
[Secrets & Security → Human authentication](../architecture/secrets.md#human-authentication-microsoft-entra-id).

## Backstage registration

Several addons (`gitops.cnpg`, `gitops.echoserver`, `gitops.headlamp`) have
`catalog-info.yaml`, registering the component in [Backstage](backstage.md)'s
catalog with `type: website`, `lifecycle: lab`, and a link to the corresponding
Application in ArgoCD.
