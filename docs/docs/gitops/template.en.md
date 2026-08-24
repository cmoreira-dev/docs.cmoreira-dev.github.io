# New app template (`gitops.template`)

Scaffold for creating a new `gitops.<app>` repository for one of the org's
own applications (not a third-party addon — for that, see the
`helm/<addon>` pattern in the [addon catalog](addons.md)).

## What's already there

```
gitops.template/
├── argocd/
│   └── .gitkeep            ← intentionally empty, see below
├── helm/
│   ├── Chart.yaml           ← depends on gitops.generic-app-chart
│   └── values.yaml
├── catalog-info.yaml
└── renovate.json
```

`helm/` already declares the
[generic app chart](../kubernetes/generic-app-chart.md) as a dependency —
covers `Deployment` + `Service` + `HTTPRoute` (Gateway API) +
`ExternalSecret` without writing any of those resources by hand.

## Using the template

1. Copy the template into the new `gitops.<app>` repo.
2. In `helm/Chart.yaml`, replace `<app-name>` with the real app name.
3. Fill in `helm/values.yaml` — `image.repository`/`tag`,
   `service.targetPort`, `httpRoute.hostnames` (which of the two Gateways,
   see [Networking & Ingress](../architecture/networking.md)),
   `externalSecrets` if the app has any secret. Full field reference lives
   in the [generic chart](../kubernetes/generic-app-chart.md)'s
   `values.yaml`.
4. Run `helm dependency update helm/` before committing, to resolve the
   subchart (generates `Chart.lock` + `charts/`, both committed).
5. Leave `argocd/` empty (just `.gitkeep`) —
   [`ApplicationSet/gitops-repos`](pattern.md#layer-1-automatic-discovery)
   generates the `Application` automatically for any `gitops.*` repo. Only
   populate `argocd/` manually if the app has more than one deployable
   component (like `gitops.teupadel.com` and `gitops.local-sara`, which
   have api + ui in the same repo).

## Resolving the chart dependency

Two source patterns for the generic chart dependency coexist today among
consumers:

- **OCI** (`oci://ghcr.io/cmoreira-dev/charts`) — used by
  `gitops.teupadel.com` and `gitops.local-sara`, resolves with no extra
  plugin on ArgoCD.
- **Git** (`git+https://...`) — this template's default, which depends on
  the `helm-git` plugin in the ArgoCD repo-server to resolve at sync time.

When copying the template, it's worth confirming which of the two the
`Chart.yaml` is using before the first real sync.
