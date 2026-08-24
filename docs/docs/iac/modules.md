# Módulos reutilizáveis

Dois módulos Terraform puros, sem estado próprio — só existem quando
consumidos por um `source` (local ou `git::`) a partir de outro repositório.

## `iac-aws-ecr-pipeline`

Provisiona a pipeline de build/push de imagens Docker para o ECR, para um ou
mais "produtos" (grupos de repositórios de imagem, ex.: `teupadel`, `sara`).

Recursos criados:

- **`aws_ecr_repository_creation_template`** por produto — os repositórios de
  imagem (`teupadel/api`, `teupadel/ui`, ...) nascem automaticamente no
  primeiro push, sem provisionamento explícito por componente.
- **Scanning do registro** (nível `BASIC` por padrão).
- **OIDC identity provider** do GitHub Actions na conta AWS.
- **IAM Role** assumível via OIDC pelos repositórios listados em
  `github_repos`, restrita à branch `main` — nenhuma access key estática é
  criada.

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

Consumido hoje apenas por `Terraform-Proxmox`, que o usa como `module` local
para provisionar um único LXC: o runner self-hosted do GitHub Actions usado
pelo bootstrap do cluster (ver [Bootstrap do cluster](bootstrap.md)).

## Versionamento

Módulos consumidos via `git::` referenciam uma `ref` (branch ou tag) no
`terragrunt.hcl`/`main.tf` do consumidor — uma mudança no módulo só chega ao
stack quando essa referência é atualizada.
