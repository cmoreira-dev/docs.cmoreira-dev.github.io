# Terragrunt (`iac.homelab-live-infra`)

Live infrastructure state, managed by Terragrunt. It's the only repository
that actually applies cloud infrastructure — the reusable modules
(`iac-proxmox-lxc`, `iac-aws-ecr-pipeline`) don't do anything on their own;
they need to be consumed from here.

## Remote backend

Configured once in the root `root.hcl` and inherited by every stack via
`include`:

- **State**: S3 (`cmoreira-dev-terraform-state`, `us-east-1`)
- **Lock**: DynamoDB (`terraform-lock-table`)

## Structure by provider

```
iac.homelab-live-infra/
├── root.hcl                    ← shared remote backend
├── _providers/                 ← dynamically generated provider blocks
│   ├── aws.hcl
│   └── proxmox.hcl
├── aws/cmoreira-dev/us-east-1/
│   ├── ecr/                    ← active stack, consumes iac-aws-ecr-pipeline
│   └── ssm-parameters/
├── azure/
│   ├── account.hcl
│   ├── entra-id/
│   └── management/
└── proxmox/
    ├── account.hcl
    └── homelab/
```

Every provider has its own `account.hcl` — including the `proxmox/` stack,
whose Terragrunt account (`provider = "aws"`) is the same AWS account used
as the state backend, since Proxmox has no "backend" of its own.

`azure/entra-id/` reserves the Azure side of the login federation used for
human access — Entra ID is today the identity provider for the AWS account,
ArgoCD, and Headlamp; see
[Secrets & Security → Human authentication](../architecture/secrets.md#human-authentication-microsoft-entra-id).
This mechanism is separate from the OIDC used by CI (which authenticates
workflow jobs, not people) — see [Build & Registry](../cicd/build-registry.md).

## Active stack: ECR

`aws/cmoreira-dev/us-east-1/ecr/terragrunt.hcl` consumes the
[`iac-aws-ecr-pipeline`](modules.md#iac-aws-ecr-pipeline) module to
provision:

- One ECR repository creation template per product (`ecr_products =
  ["teupadel", "sara"]`) — every image push creates the repository on demand
  (`CREATE_ON_PUSH`), with no need to declare an `aws_ecr_repository` per
  component.
- Registry scanning configuration.
- The GitHub Actions OIDC identity provider and the IAM Role that CI
  workflows assume to publish images — see [Build & Registry](../cicd/build-registry.md).

## Applying changes

```bash
cd aws/cmoreira-dev/us-east-1/ecr
terragrunt plan
terragrunt apply   # requires explicit confirmation — never automated
```

There's no CI that runs `terragrunt apply` automatically — all AWS
infrastructure application is manual, reviewed before running.
