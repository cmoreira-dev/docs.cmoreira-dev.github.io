# Two tiers of IaC

Infrastructure in three places — AWS, Azure and Proxmox — managed by **two
solutions**, both using the same language (OpenTofu/Terragrunt), the same state
backend (S3 `cmoreira-dev-terraform-state` + DynamoDB lock `terraform-lock-table`)
and the same modules. What differs is *who runs it*.

## Tier 1 — base infra / bootstrap / DR

**Repo:** [`iac.homelab-live-infra`](terragrunt.md) · **Executor:** GitHub Actions
(hosted runners) for `aws/**` and `azure/**`; local `terragrunt` for `proxmox/**`.

What must exist *before and independently of* the cluster: state backend, OIDC
providers / IAM roles, Entra ID, DNS zones, Proxmox VMs/LXC, and the IAM users +
SSM parameters Tier 2 consumes.

| Layer | How it runs |
|---|---|
| `aws/**`, `azure/**` | PR → `plan` comment; merge → `apply`; nightly → `drift` (opens an issue) |
| `proxmox/**` | Local only — hosted runners can't reach the LAN. CI skips it via a filter. |

Each run is a single `terragrunt run --all --filter '!./proxmox/**'` over the
cloud tree, DAG-ordered. Auth is OIDC (`gha-cmoreira-dev-iac-plan` / `-apply`
roles — managed **outside Terraform**, see
[Bootstrap](bootstrap.md#iac-pipeline-identities-out-of-band)).

## Tier 2 — application-dependency resources

**Executor:** [Burrito](../gitops/burrito.md), an in-cluster operator (installed
under `gitops.core-addons`). The HCL lives in each `gitops.<app>` repo's
`terraform/` folder (or `gitops.core-addons`'s — the pilot is there).

Covers what an app needs: a managed database, a bucket, per-app DNS, secrets.
Burrito plans continuously (every 10m) and, when a layer has
`remediationStrategy.autoApply: true`, re-applies on its own. On PRs it comments
the plan.

It replaced Crossplane — same paradigm as Tier 1 (Terraform/state/modules), just
reconciled in-cluster.

### Boundary

Tier 1 and Tier 2 **never** touch the same path: Tier 1 CI only touches
`iac.homelab-live-infra`; Burrito only touches `terraform/` folders in GitOps
repos. Distinct state keys in the same bucket.
