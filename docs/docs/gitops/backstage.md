# Backstage (`backstage.homelab`)

Portal interno de desenvolvedor, para dar visibilidade única sobre os
componentes e serviços da org (catálogo de software), sobre o padrão
[Backstage](https://backstage.io/).

## Registro de catálogo

Vários repositórios já carregam um `catalog-info.yaml`, registrando-se no
catálogo do Backstage:

- Addons: `gitops.cnpg`, `gitops.echoserver`,
  `gitops.headlamp`, `gitops.template`
- Infra: `homelab-bootsrap-k3s`
- Apps: `gitops.local-sara`, `gitops.teupadel.com`

O padrão de anotação (`type: website`, `lifecycle: lab`, tags `kubernetes` /
`homelab` / `infrastructure`, link para a Application correspondente via
anotação `argocd/app-name`) é o mesmo entre todos os repos registrados — ver
[Padrões de Repositório](../repos.md) para a estrutura completa de um
`gitops.<app>`.

## Persistência

A instância do Backstage usa Postgres via o operador **CloudNativePG**
(`gitops.cnpg`), que provisiona um banco dedicado (`kustomize/backstage/`) com
um `PodMonitor` para observabilidade.

## Deploy

Como qualquer outra app própria do cluster, o Backstage é entregue via GitOps —
ver [Padrão GitOps](pattern.md).
