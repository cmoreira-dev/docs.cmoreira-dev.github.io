# Template de novo app (`gitops.template`)

Scaffold para criar um novo repositório `gitops.<app>` para uma aplicação
própria (não um addon de terceiros — para isso, ver o padrão `helm/<addon>` do
[catálogo de addons](addons.md)).

## O que já vem pronto

```
gitops.template/
├── argocd/
│   └── .gitkeep            ← vazio de propósito, ver abaixo
├── helm/
│   ├── Chart.yaml           ← depende de gitops.generic-app-chart
│   └── values.yaml
├── catalog-info.yaml
└── renovate.json
```

`helm/` já declara o [chart genérico de apps](../kubernetes/generic-app-chart.md)
como dependência — cobre `Deployment` + `Service` + `HTTPRoute` (Gateway API) +
`ExternalSecret` sem precisar escrever nenhum desses recursos à mão.

## Usando o template

1. Copiar o template para o novo repo `gitops.<app>`.
2. Em `helm/Chart.yaml`, trocar `<app-name>` pelo nome real do app.
3. Preencher `helm/values.yaml` — `image.repository`/`tag`,
   `service.targetPort`, `httpRoute.hostnames` (qual das duas Gateways, ver
   [Rede & Ingress](../architecture/networking.md)), `externalSecrets` se a app
   tiver algum segredo. Referência completa dos campos no `values.yaml` do
   [chart genérico](../kubernetes/generic-app-chart.md).
4. Rodar `helm dependency update helm/` antes de commitar, para resolver o
   subchart (gera `Chart.lock` + `charts/`, ambos commitados).
5. Deixar `argocd/` vazio (só `.gitkeep`) — o
   [`ApplicationSet/gitops-repos`](pattern.md#camada-1-descoberta-automatica)
   gera a `Application` automaticamente para qualquer repo `gitops.*`. Só
   popular `argocd/` manualmente se o app tiver mais de um componente
   deployável (como `gitops.teupadel.com` e `gitops.local-sara`, que têm api +
   ui no mesmo repo).

## Resolução da dependência do chart

Dois padrões de origem para a dependência do chart genérico coexistem hoje
entre os consumidores:

- **OCI** (`oci://ghcr.io/cmoreira-dev/charts`) — usado por
  `gitops.teupadel.com` e `gitops.local-sara`, resolve sem plugin adicional no
  ArgoCD.
- **Git** (`git+https://...`) — o default deste template, que depende do
  plugin `helm-git` no repo-server do ArgoCD para resolver em sync.

Ao copiar o template, vale confirmar qual das duas formas o `Chart.yaml` está
usando antes do primeiro sync real.
