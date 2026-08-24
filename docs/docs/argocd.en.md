# Argo CD

ArgoCD is the only component in the cluster that applies manifests directly
— everything else in the deploy flow is git push + automatic
reconciliation. See [GitOps pattern](gitops/pattern.md) for the full
mechanism, end to end.

## Installation

Installed via Helm (official `argo/argo-cd` chart) by the Ansible playbooks
at cluster bootstrap — see [Cluster bootstrap](iac/bootstrap.md). SSO is
configured via Dex, connected to Microsoft Entra ID — there's no local
username/password in everyday use.

Exposed at `argocd.cmoreira.dev`, behind the same Cloudflare Tunnel →
Gateway API chain described in [Networking & Ingress](architecture/networking.md).

## `AppProject/homelab`

A single `AppProject`, created at bootstrap, with a broad scope:

- Any `cmoreira-dev/*` org repository as an allowed source
- Any destination/namespace in the cluster
- Any Kubernetes resource kind

This keeps the project simple for a single-operator homelab — there's no
project segregation by team/environment, because there's no multiple teams
or multiple environments in the cluster.

## `ApplicationSet/gitops-repos`

The automatic-discovery generator — detailed in
[GitOps pattern → Layer 1](gitops/pattern.md#layer-1-automatic-discovery).
Summary:

- `scmProvider` generator against the GitHub org, filter `^gitops\..*`
- Polls every 300s for new `gitops.*` repositories
- One `Application` per repository found, pointed at that repo's `argocd/`

## Sync policy

Every generated `Application` uses:

```yaml
syncPolicy:
  automated:
    prune: true      # removes resources that left Git
    selfHeal: true    # reverts any change made outside of Git
  syncOptions:
    - ServerSideApply=true
    - CreateNamespace=true
```

There's no manual sync in the normal flow — any drift between the cluster
and Git is automatically corrected by `selfHeal`.

## Image updates

`argocd-image-updater` (an addon in `gitops.core-addons`) watches ECR and
writes back to the `gitops.*` repos when a new image tag lands, using its
own credentials via `ExternalSecret` — see
[Secrets & Security](architecture/secrets.md).
