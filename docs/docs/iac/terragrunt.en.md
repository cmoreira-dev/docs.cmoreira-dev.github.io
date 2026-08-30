# Terragrunt (`iac.homelab-live-infra`)

**Tier 1** of the [IaC strategy](tiers.md) — base infra, managed by Terragrunt +
**OpenTofu**. The reusable modules (`iac-proxmox-lxc`, `iac-aws-ecr-pipeline`)
don't do anything on their own; they need to be consumed from here.
Application-dependency resources are Tier 2 ([Burrito](../gitops/burrito.md)).

## Remote backend

Configured once in the root `root.hcl` and inherited by every stack via
`include`:

- **State**: S3 (`cmoreira-dev-terraform-state`, `us-east-1`)
- **Lock**: DynamoDB (`terraform-lock-table`)

## Structure by provider

```
iac.homelab-live-infra/
├── root.hcl                    ← shared remote backend
├── bootstrap/                  ← pipeline OIDC roles (AWS CLI, outside Terraform)
├── _providers/                 ← dynamically generated provider blocks
│   ├── aws.hcl
│   ├── azure.hcl
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

Each `account.hcl` sets `provider = "aws" | "azure" | "proxmox"` (which
`_providers/*.hcl` to generate) and the account inputs. The state backend is
always the same S3 bucket — only the *key* changes per layer.

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

`aws/**` and `azure/**` layers run via **GitHub Actions**, each job a single
`terragrunt run --all --filter '!./proxmox/**'` over the cloud tree, DAG-ordered:

- **PR** → `plan`, commented on the PR.
- **merge to `main`** → `apply` (behind the `iac-apply` environment).
- **nightly** → `plan -detailed-exitcode`; drift opens/updates an issue.

Auth is OIDC — roles `gha-cmoreira-dev-iac-plan` (ReadOnly + state) and
`gha-cmoreira-dev-iac-apply` (Admin), managed by `bootstrap/bootstrap.sh`
**outside Terraform** (see [Bootstrap](bootstrap.md#iac-pipeline-identities-out-of-band)).

`proxmox/**` layers do **not** run in CI (hosted runners can't reach the LAN) —
apply locally:

```bash
export TF_VAR_proxmox_api_token='user@pve!token=...'
AWS_PROFILE=terraform terragrunt run --all --working-dir proxmox -- apply
```
