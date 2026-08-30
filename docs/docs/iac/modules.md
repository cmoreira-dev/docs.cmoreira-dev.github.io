# Módulos reutilizáveis

Módulos Terraform puros, sem estado próprio — só existem quando consumidos por um
`source` (local ou `git::`) a partir de outro repositório.

## `iac-aws-ecr-pipeline`

Provisiona a pipeline de build/push de imagens Docker para o ECR, para um ou
mais "produtos" (grupos de repositórios de imagem, ex.: `teupadel`, `sara`).

Recursos criados:

- **`aws_ecr_repository_creation_template`** por produto — os repositórios de
  imagem (`teupadel/api`, `teupadel/ui`, ...) nascem automaticamente no
  primeiro push, sem provisionamento explícito por componente.
- **Scanning do registro** (nível `BASIC` por padrão).
- **IAM Role** de push assumível via OIDC pelos repositórios listados em
  `github_repos`, restrita à branch `main` + ao workflow reutilizável
  `build-push-ecr.yml` — nenhuma access key estática é criada.

O **OIDC provider** do GitHub Actions é hoje um `data` source aqui — quem cria e
é dono dele é `iac.homelab-live-infra/bootstrap/bootstrap.sh` (um por conta,
compartilhado com as roles do pipeline de IaC).

Consumido hoje por `iac.homelab-live-infra/aws/cmoreira-dev/us-east-1/ecr`.

## `iac-proxmox-lxc`

Módulo para criar containers LXC no Proxmox.

```
iac-proxmox-lxc/
├── lxc.tf           ← recurso do container
├── remote-exec.tf   ← provisionamento pós-criação via SSH
├── variables.tf
└── outputs.tf
```

O bloco `provider "proxmox"` foi removido do módulo — ele é injetado pelo
Terragrunt (`_providers/proxmox.hcl`). Consumido hoje só por `Terraform-Proxmox`
(o LXC do runner `rasp-ansible`, hoje parado — ver [Bootstrap](bootstrap.md)).

## `_modules/` do `gitops.core-addons`

Padrão diferente: módulos **repo-local** dentro de `gitops.core-addons/terraform/`
(sem a cerimônia de `ref` de Git), consumidos pelos subfolders daquele repo:

- **`s3`** — bucket com versioning, SSE, block-public, lifecycle.
- **`ssm-user`** — IAM user + access key escritos direto no SSM (sem paste
  manual). Usado por `terraform/burrito/` (users `burrito-datastore` /
  `burrito-runner`).

## Versionamento

Módulos consumidos via `git::` referenciam uma `ref` (branch ou tag) no
`terragrunt.hcl`/`main.tf` do consumidor — uma mudança no módulo só chega ao
stack quando essa referência é atualizada. Módulos repo-local (`../_modules/...`)
não têm essa etapa.
