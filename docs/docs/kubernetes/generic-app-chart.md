# Chart genérico de apps (`gitops.generic-app-chart`)

Chart Helm compartilhado (`generic-app`) para workloads próprias do homelab. Cada app
consumidora (`gitops.<app>`, padel api/ui) o referencia como dependência no seu
`Chart.yaml` e fornece só o seu `values.yaml`.

## Recursos que o chart renderiza

| Recurso | Gate | Observações |
|---|---|---|
| `Deployment` | sempre | probes, resources, nodeSelector (x86/ARM64), securityContext |
| `ServiceAccount` | `serviceAccount.create` | default `true` |
| `Service` | sempre | `service.port` → `service.targetPort` |
| `HTTPRoute` (Gateway API) | `httpRoute.enabled` | alvo: NGINX Gateway Fabric |
| `ConfigMap` | `configMap.enabled` | ver abaixo |
| `ExternalSecret` (app) | `externalSecrets.enabled` | via `ClusterSecretStore/aws-ssm` |
| `ExternalSecret` (pull ECR) | `ecrPullSecret.enabled` | ver abaixo |

## Ciclo de vida do Namespace

Desde a versão **0.2.0** o chart **não cria Namespace**. A criação é responsabilidade do
ArgoCD, via `syncOptions: [CreateNamespace=true]` na Application. `namespace.name` existe
apenas para direcionar os recursos (vazio = namespace do release). As chaves antigas
`namespace.create/labels/annotations/istioInjection` foram removidas.

## ConfigMap

```yaml
configMap:
  enabled: true
  data:
    LOG_LEVEL: debug
    config.yaml: |
      server:
        port: 8080
  injectAsEnv: true    # todas as keys viram env vars (envFrom)
  mountPath: /etc/app  # monta as keys como arquivos readOnly (volume "generic-app-config")
```

- Nome default: fullname do release (`configMap.name` sobrescreve).
- Mudanças em `data` fazem rollout automático do Deployment (annotation
  `checksum/config` no pod template).
- Montagens parciais (uma key só, subPath): usar `volumes`/`volumeMounts` manuais.

## Pull secret ECR (`ecrPullSecret`)

```yaml
ecrPullSecret:
  enabled: true
```

Cria um `ExternalSecret` (sync-wave `-1`) que materializa um Secret
`kubernetes.io/dockerconfigjson` a partir do `ClusterGenerator/ecr-token` — **gerenciado no
`gitops.core-addons`** (`kustomize/external-secrets/authorization-token.yaml`), não
replicado neste chart. O Secret gerado (`<fullname>-ecr-pull` por default) é adicionado
automaticamente ao `imagePullSecrets` do pod. Cada app que habilita a flag tem o seu
próprio Secret no seu namespace — sem disputa de ownership entre Applications no ArgoCD.

## Versionamento e consumers

Os consumers pinam a versão do chart em `Chart.yaml`/`Chart.lock`. Uma mudança neste repo
**só chega às apps quando cada consumer der bump + `helm dependency update`**. Consumers
atuais: `padel-movement` api/ui e `gitops.template`.
