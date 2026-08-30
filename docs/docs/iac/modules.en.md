# Reusable modules

Plain Terraform modules, with no state of their own — they only exist when
consumed via a `source` (local or `git::`) from another repository.

## `iac-aws-ecr-pipeline`

Provisions the Docker image build/push pipeline for one or more "products"
(groups of image repositories, e.g. `teupadel`, `sara`).

Resources created:

- **`aws_ecr_repository_creation_template`** per product — the image
  repositories (`teupadel/api`, `teupadel/ui`, ...) are born automatically on
  first push, with no explicit provisioning per component.
- **Registry scanning** (`BASIC` level by default).
- **Push IAM Role**, assumable via OIDC by the repos listed in `github_repos`,
  restricted to the `main` branch + the `build-push-ecr.yml` reusable workflow —
  no static access key is ever created.

The GitHub Actions **OIDC provider** is now a `data` source here — it is created
and owned by `iac.homelab-live-infra/bootstrap/bootstrap.sh` (one per account,
shared with the IaC pipeline roles).

Consumed today by
`iac.homelab-live-infra/aws/cmoreira-dev/us-east-1/ecr`.

## `iac-proxmox-lxc`

Module for creating LXC containers on Proxmox.

```
iac-proxmox-lxc/
├── lxc.tf           ← the container resource
├── remote-exec.tf   ← post-creation provisioning over SSH
├── variables.tf
└── outputs.tf
```

The `provider "proxmox"` block was removed from the module — it's injected by
Terragrunt (`_providers/proxmox.hcl`). Consumed today only by `Terraform-Proxmox`
(the `rasp-ansible` runner's LXC, currently stopped — see [Bootstrap](bootstrap.md)).

## `_modules/` in `gitops.core-addons`

A different pattern: **repo-local** modules inside
`gitops.core-addons/terraform/` (no Git `ref` ceremony), consumed by that repo's
subfolders:

- **`s3`** — bucket with versioning, SSE, block-public, lifecycle.
- **`ssm-user`** — IAM user + access key written straight to SSM (no manual
  paste). Used by `terraform/burrito/` (users `burrito-datastore` /
  `burrito-runner`).

## Versioning

Modules consumed via `git::` reference a `ref` (branch or tag) in the consumer's
`terragrunt.hcl`/`main.tf` — a change to the module only reaches the stack once
that reference is updated. Repo-local modules (`../_modules/...`) skip that step.
