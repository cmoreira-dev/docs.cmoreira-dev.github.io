# Reusable modules

Two plain Terraform modules, with no state of their own — they only exist
when consumed via a `source` (local or `git::`) from another repository.

## `iac-aws-ecr-pipeline`

Provisions the Docker image build/push pipeline for one or more "products"
(groups of image repositories, e.g. `teupadel`, `sara`).

Resources created:

- **`aws_ecr_repository_creation_template`** per product — the image
  repositories (`teupadel/api`, `teupadel/ui`, ...) are born automatically on
  first push, with no explicit provisioning per component.
- **Registry scanning** (`BASIC` level by default).
- **GitHub Actions OIDC identity provider** in the AWS account.
- **IAM Role**, assumable via OIDC by the repos listed in `github_repos`,
  restricted to the `main` branch — no static access key is ever created.

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

Consumed today only by `Terraform-Proxmox`, which uses it as a local
`module` to provision a single LXC: the GitHub Actions self-hosted runner
used by the cluster bootstrap (see [Cluster bootstrap](bootstrap.md)).

## Versioning

Modules consumed via `git::` reference a `ref` (branch or tag) in the
consumer's `terragrunt.hcl`/`main.tf` — a change to the module only reaches
the stack once that reference is updated.
