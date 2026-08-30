# Burrito (Tier-2 IaC)

[Burrito](https://docs.burrito.tf) — "ArgoCD for Terraform". A Kubernetes operator
that runs `plan`/`apply` on OpenTofu/Terragrunt driven by `TerraformRepository`
and `TerraformLayer` CRDs. It is **Tier 2** of the
[IaC strategy](../iac/tiers.md) — it replaced Crossplane.

Installed under `gitops.core-addons` (not a separate repo):

```
helm/burrito/              → argocd/burrito.yaml         (wave 3) → ns burrito-system
  templates/               → ExternalSecrets (datastore AWS, GitHub App, git creds)
kustomize/burrito-config/  → argocd/burrito-config.yaml  (wave 4) → ns burrito-tenant
  → burrito-runner-aws ES + TerraformRepository + TerraformLayer
terraform/burrito/         → the AWS infra Burrito needs (outside GitOps)
```

## Components

| Namespace | Contents |
|---|---|
| `burrito-system` | controllers, datastore (plan/log state in S3), server |
| `burrito-tenant` | `Terraform*` CRDs, runner pods, `burrito-runner` SA |

- **Datastore → S3** `cmoreira-dev-burrito-datastore`, provisioned by
  `terraform/burrito/`. No IRSA on this cluster → static credentials via env
  (secret `burrito-datastore-aws` ← SSM). TLS via cert-manager.
- **Runners** get AWS credentials via `overrideRunnerSpec.envFrom`
  (`burrito-runner-aws` ← SSM), backed by the `burrito-runner` IAM user
  (per-layer perms, added at onboarding).
- **Git auth / PR comments** — a GitHub App. A shared secret
  `burrito-git-shared` (`credentials.burrito.tf/shared`, `url:
  https://github.com/cmoreira-dev`) covers every repo in the org.

## PR comments — no public endpoint

The repository controller polls every 5m, detects PRs/commits, runs the plan and
posts the comment via the **GitHub App API (outbound)**. There is no webhook and
no exposed `burrito.cmoreira.dev` — Burrito's server does support OIDC (Entra ID)
for when the UI is published, but that hasn't been done yet.

## Drift and auto-remediation

`driftDetection: 10m` — the controller replans each layer every 10m. With
`remediationStrategy.autoApply: true` on the layer, drift is re-applied. The
pilot (`ecr-pull-iam`) runs with `autoApply: false` — plan only.

## Onboarding a layer

1. Backend `s3://cmoreira-dev-terraform-state` in the repo's `terraform/` folder,
   `tofu init -migrate-state` from a laptop, confirm an empty plan.
2. Add a `TerraformRepository` (matched by `url`, no `secretName`) + a
   `TerraformLayer` (`path`) to `kustomize/burrito-config/`.
3. Grant `burrito-runner` the AWS perms that folder needs — a new policy in
   `terraform/burrito/main.tf`, re-apply.
