# Burrito (Tier-2 IaC)

[Burrito](https://docs.burrito.tf) — "ArgoCD para Terraform". Operador Kubernetes
que faz `plan`/`apply` de OpenTofu/Terragrunt a partir de `TerraformRepository` e
`TerraformLayer` CRDs. É o **Tier 2** da [estratégia de IaC](../iac/tiers.md).

Instalado em `gitops.core-addons` (não é repo separado):

```
helm/burrito/              → argocd/burrito.yaml         (wave 3) → ns burrito-system
  templates/               → ExternalSecrets (datastore AWS, GitHub App, git creds)
kustomize/burrito-config/  → argocd/burrito-config.yaml  (wave 4) → ns burrito-tenant
  → burrito-runner-aws ES + TerraformRepository + TerraformLayer
terraform/burrito/         → infra AWS que o Burrito precisa (fora do GitOps)
```

## Componentes

| Namespace | Conteúdo |
|---|---|
| `burrito-system` | controllers, datastore (state de plans/logs no S3), server |
| `burrito-tenant` | CRDs `Terraform*`, runner pods, SA `burrito-runner` |

- **Datastore → S3** `cmoreira-dev-burrito-datastore`, provisionado por
  `terraform/burrito/`. Sem IRSA neste cluster → credenciais estáticas via env
  (secret `burrito-datastore-aws` ← SSM). TLS via cert-manager.
- **Runners** recebem credencial AWS via `overrideRunnerSpec.envFrom`
  (`burrito-runner-aws` ← SSM), lastreada pelo IAM user `burrito-runner`
  (perms por-layer, adicionadas no onboarding).
- **Auth Git / comentários em PR** — um GitHub App. Secret compartilhado
  `burrito-git-shared` (`credentials.burrito.tf/shared`, `url:
  https://github.com/cmoreira-dev`) cobre todos os repos da org.

## Comentários em PR — sem endpoint público

O repository controller faz *poll* a cada 5min, detecta PRs/commits, roda o plan,
e posta o comentário via **API do GitHub App (outbound)**. Não há webhook nem
`burrito.cmoreira.dev` exposto — o server do Burrito suporta OIDC (Entra ID) para
quando a UI for publicada, mas isso ainda não foi feito.

## Drift e auto-remediação

`driftDetection: 10m` — o controller re-planeja cada layer a cada 10min. Com
`remediationStrategy.autoApply: true` no layer, drift é re-aplicado. O piloto
(`ecr-pull-iam`) roda com `autoApply: false` — só plan.

## Onboarding de um layer

1. Backend `s3://cmoreira-dev-terraform-state` na pasta `terraform/` do repo,
   `tofu init -migrate-state` do laptop, confirmar plan vazio.
2. Adicionar `TerraformRepository` (matched pelo `url`, sem `secretName`) +
   `TerraformLayer` (`path`) ao `kustomize/burrito-config/`.
3. Dar ao `burrito-runner` as perms AWS daquele folder — nova policy em
   `terraform/burrito/main.tf`, re-aplicar.
