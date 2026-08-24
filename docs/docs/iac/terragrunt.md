# Terragrunt (`iac.homelab-live-infra`)

Estado de infraestrutura viva, gerenciado por Terragrunt. É o único repositório
que efetivamente aplica infraestrutura de nuvem — os módulos reutilizáveis
(`iac-proxmox-lxc`, `iac-aws-ecr-pipeline`) não fazem nada sozinhos; precisam ser
consumidos a partir daqui.

## Backend remoto

Configurado uma única vez no `root.hcl` da raiz e herdado por todo stack via
`include`:

- **State**: S3 (`cmoreira-dev-terraform-state`, `us-east-1`)
- **Lock**: DynamoDB (`terraform-lock-table`)

## Estrutura por provider

```
iac.homelab-live-infra/
├── root.hcl                    ← backend remoto compartilhado
├── _providers/                 ← blocos de provider gerados dinamicamente
│   ├── aws.hcl
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

Cada provider tem seu `account.hcl` — inclusive o stack `proxmox/`, cuja conta
Terragrunt (`provider = "aws"`) é a mesma conta AWS usada como backend de state,
já que Proxmox não tem um "backend" próprio.

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

```bash
cd aws/cmoreira-dev/us-east-1/ecr
terragrunt plan
terragrunt apply   # requer confirmação explícita — nunca automatizado
```

Não há CI que rode `terragrunt apply` automaticamente — toda aplicação de
infraestrutura AWS é manual, revisada antes de rodar.
