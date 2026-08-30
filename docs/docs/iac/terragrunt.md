# Terragrunt (`iac.homelab-live-infra`)

**Tier 1** da [estratégia de IaC](tiers.md) — infra base, gerida por
Terragrunt + **OpenTofu**. Os módulos reutilizáveis (`iac-proxmox-lxc`,
`iac-aws-ecr-pipeline`) não fazem nada sozinhos; precisam ser consumidos a partir
daqui. Recursos de dependência das aplicações são Tier 2 ([Burrito](../gitops/burrito.md)).

## Backend remoto

Configurado uma única vez no `root.hcl` da raiz e herdado por todo stack via
`include`:

- **State**: S3 (`cmoreira-dev-terraform-state`, `us-east-1`)
- **Lock**: DynamoDB (`terraform-lock-table`)

## Estrutura por provider

```
iac.homelab-live-infra/
├── root.hcl                    ← backend remoto compartilhado
├── bootstrap/                  ← roles OIDC do pipeline (AWS CLI, fora do Terraform)
├── _providers/                 ← blocos de provider gerados dinamicamente
│   ├── aws.hcl
│   ├── azure.hcl
│   └── proxmox.hcl
├── aws/cmoreira-dev/us-east-1/
│   ├── ecr/                    ← stack ativo, consome iac-aws-ecr-pipeline
│   └── ssm-parameters/
├── azure/
│   ├── account.hcl
│   ├── entra-id/
│   └── management/
└── proxmox/
    ├── account.hcl
    └── homelab/
```

Cada `account.hcl` define `provider = "aws" | "azure" | "proxmox"` (qual
`_providers/*.hcl` gerar) e os inputs de conta. O backend de state é sempre o
mesmo bucket S3 — só a *key* muda por camada.

`azure/entra-id/` reserva o lado Azure da federação de login usada para
acesso humano — Entra ID é hoje o provedor de identidade para a conta AWS,
o ArgoCD e o Headlamp; ver
[Secrets & Segurança → Autenticação humana](../architecture/secrets.md#autenticacao-humana-microsoft-entra-id).
Esse mecanismo é separado do OIDC usado pelo CI (que autentica jobs de
workflow, não pessoas) — ver [Build & Registry](../cicd/build-registry.md).

## Stack ativo: ECR

`aws/cmoreira-dev/us-east-1/ecr/terragrunt.hcl` consome o módulo
[`iac-aws-ecr-pipeline`](modules.md#iac-aws-ecr-pipeline) para provisionar:

- Um template de criação de repositório ECR por produto (`ecr_products =
  ["teupadel", "sara"]`) — cada push de imagem cria o repositório sob demanda
  (`CREATE_ON_PUSH`), sem precisar declarar um `aws_ecr_repository` por
  componente.
- Configuração de scanning do registro.
- O identity provider OIDC do GitHub Actions e a IAM Role que os workflows de
  CI assumem para publicar imagens — ver [Build & Registry](../cicd/build-registry.md).

## Aplicando mudanças

Camadas `aws/**` e `azure/**` rodam via **GitHub Actions**, cada job um único
`terragrunt run --all --filter '!./proxmox/**'` sobre a árvore de cloud,
ordenado por dependência:

- **PR** → `plan`, comentado no PR.
- **merge em `main`** → `apply` (atrás do environment `iac-apply`).
- **nightly** → `plan -detailed-exitcode`; drift abre/atualiza uma issue.

Autenticação por OIDC — roles `gha-cmoreira-dev-iac-plan` (ReadOnly + state) e
`gha-cmoreira-dev-iac-apply` (Admin), geridas por `bootstrap/bootstrap.sh`
**fora do Terraform** (ver [Bootstrap](bootstrap.md#identidades-do-pipeline-de-iac-out-of-band)).

Camadas `proxmox/**` **não** rodam no CI (runners hospedados não alcançam a LAN)
— aplicação local:

```bash
export TF_VAR_proxmox_api_token='user@pve!token=...'
AWS_PROFILE=terraform terragrunt run --all --working-dir proxmox -- apply
```
