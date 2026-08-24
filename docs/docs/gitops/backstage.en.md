# Backstage (`backstage.homelab`)

Internal developer portal, giving unified visibility over the org's
components and services (software catalog), built on
[Backstage](https://backstage.io/).

## Catalog registration

Several repositories already carry a `catalog-info.yaml`, registering
themselves in Backstage's catalog:

- Addons: `gitops.cnpg`, `gitops.crossplane`, `gitops.echoserver`,
  `gitops.headlamp`, `gitops.template`
- Infra: `homelab-bootsrap-k3s`
- Apps: `gitops.local-sara`, `gitops.teupadel.com`

The annotation pattern (`type: website`, `lifecycle: lab`, tags
`kubernetes` / `homelab` / `infrastructure`, link to the corresponding
Application via the `argocd/app-name` annotation) is the same across every
registered repo — see [Repository Patterns](../repos.md) for the full
structure of a `gitops.<app>`.

## Persistence

The Backstage instance uses Postgres via the **CloudNativePG** operator
(`gitops.cnpg`), which provisions a dedicated database
(`kustomize/backstage/`) with a `PodMonitor` for observability.

## Deployment

Like any other of the cluster's own apps, Backstage is delivered via GitOps
— see [GitOps pattern](pattern.md).
