# Generic app chart (`gitops.generic-app-chart`)

Shared Helm chart (`generic-app`) for the homelab's own workloads. Every
consumer app (`gitops.<app>`, teupadel/sara api/ui) references it as a
dependency in its `Chart.yaml` and only supplies its own `values.yaml`.

## Resources rendered by the chart

| Resource | Gated by | Notes |
|---|---|---|
| `Deployment` | always | probes, resources, nodeSelector (x86/ARM64), securityContext |
| `ServiceAccount` | `serviceAccount.create` | defaults to `true` |
| `Service` | always | `service.port` → `service.targetPort` |
| `HTTPRoute` (Gateway API) | `httpRoute.enabled` | targets NGINX Gateway Fabric |
| `ConfigMap` | `configMap.enabled` | see below |
| `ExternalSecret` (app) | `externalSecrets.enabled` | via `ClusterSecretStore/aws-ssm` |
| `ExternalSecret` (ECR pull) | `ecrPullSecret.enabled` | see below |

## Namespace lifecycle

As of version **0.2.0** the chart **does not create the Namespace**.
Creation is ArgoCD's responsibility, via `syncOptions: [CreateNamespace=true]`
on the Application. `namespace.name` only exists to target resources
(empty = the release's own namespace). The old keys
`namespace.create/labels/annotations/istioInjection` have been removed.

## ConfigMap

```yaml
configMap:
  enabled: true
  data:
    LOG_LEVEL: debug
    config.yaml: |
      server:
        port: 8080
  injectAsEnv: true    # every key becomes an env var (envFrom)
  mountPath: /etc/app  # mounts the keys as read-only files (volume "generic-app-config")
```

- Default name: the release's fullname (`configMap.name` overrides it).
- Changes to `data` trigger an automatic Deployment rollout (`checksum/config`
  annotation on the pod template).
- Partial mounts (a single key, subPath): use manual `volumes`/`volumeMounts`.

## ECR pull secret (`ecrPullSecret`)

```yaml
ecrPullSecret:
  enabled: true
```

Creates an `ExternalSecret` (sync-wave `-1`) that materializes a
`kubernetes.io/dockerconfigjson` Secret from the
`ClusterGenerator/ecr-token` — **managed in `gitops.core-addons`**
(`kustomize/external-secrets/authorization-token.yaml`), not replicated in
this chart. The generated Secret (`<fullname>-ecr-pull` by default) is
automatically added to the pod's `imagePullSecrets`. Every app that enables
the flag gets its own Secret in its own namespace — no ownership contention
between ArgoCD Applications.

## Versioning and consumers

Consumers pin the chart version in `Chart.yaml`/`Chart.lock`. A change in
this repo **only reaches the apps once each consumer bumps the version and
runs `helm dependency update`**. Current consumers: `teupadel` and `sara`
api/ui, and `gitops.template`.
