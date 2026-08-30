# Dois níveis de IaC

Infra em três lugares — AWS, Azure e Proxmox — gerida por **duas soluções**,
ambas com a mesma linguagem (OpenTofu/Terragrunt), o mesmo backend de state
(S3 `cmoreira-dev-terraform-state` + lock DynamoDB `terraform-lock-table`) e os
mesmos módulos. O que muda é *quem executa*.

## Tier 1 — infra base / bootstrap / DR

**Repo:** [`iac.homelab-live-infra`](terragrunt.md) · **Executor:** GitHub Actions
(runners hospedados) para `aws/**` e `azure/**`; `terragrunt` local para
`proxmox/**`.

O que precisa existir *antes e independente* do cluster: backend de state,
provedores OIDC / IAM Roles, Entra ID, zonas DNS, VMs/LXC do Proxmox, e os IAM
users + parâmetros SSM que o Tier 2 consome.

| Camada | Como roda |
|---|---|
| `aws/**`, `azure/**` | PR → `plan` comentado; merge → `apply`; nightly → `drift` (abre issue) |
| `proxmox/**` | Só local — runners hospedados não alcançam a LAN. O CI ignora via filtro. |

Cada run é um único `terragrunt run --all --filter '!./proxmox/**'` sobre a
árvore de cloud, ordenado por dependência (DAG). Autenticação por OIDC (roles
`gha-cmoreira-dev-iac-plan` / `-apply` — geridas **fora do Terraform**, ver
[Bootstrap](bootstrap.md#identidades-do-pipeline-de-iac-out-of-band)).

## Tier 2 — recursos de dependência das aplicações

**Executor:** [Burrito](../gitops/burrito.md), operador no cluster (instalado em
`gitops.core-addons`). O código HCL vive na pasta `terraform/` de cada repo
`gitops.<app>` (ou de `gitops.core-addons` — o piloto está lá).

Cobre o que uma aplicação precisa: um banco gerenciado, um bucket, DNS por app,
secrets. O Burrito faz `plan` contínuo (a cada 10min) e, quando o layer tem
`remediationStrategy.autoApply: true`, re-aplica sozinho. Em PRs comenta o plan.

Mesma paradigma do Tier 1 (Terraform/state/módulos), só que reconciliada no
cluster em vez de num pipeline.

### Fronteira

Tier 1 e Tier 2 **nunca** tocam o mesmo caminho: o CI do Tier 1 só mexe em
`iac.homelab-live-infra`; o Burrito só mexe em pastas `terraform/` de repos
GitOps. State keys distintas no mesmo bucket.
