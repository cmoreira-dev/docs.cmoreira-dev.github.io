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

## Tema visual (restyle Apple)

O frontend do Backstage (`backstage.homelab`) usa um tema inspirado no design system da Apple,
extraído do documento `DESIGN-apple.md` (raiz do repo). O documento descreve páginas de marketing;
aplicamos só a linguagem visual (cores, tipografia, raios, elevação, botões) aos padrões reais do
Backstage, não o layout de marketing.

- **Tokens** em um único arquivo, `packages/app/src/theme/appleTheme.ts` (nenhum hex fora dele),
  registrado pelo `ThemeBlueprint` do sistema de frontend novo (`createFrontendModule`), como `light`
  em `pluginId: 'app'` (sobrescreve o tema padrão).
- **Tipografia Inter self-hosted** via `@fontsource/inter`, não Google Fonts: o app tem uma CSP
  endurecida (`app-config.yaml`) e um `<link>` externo exigiria abrir `style-src`/`font-src` e vazaria
  IPs para o Google. A pilha começa por `system-ui, -apple-system`, então dispositivos Apple usam a SF Pro.
- **Sem gradientes decorativos, sem sombras**: profundidade por troca de superfície; a barra lateral e
  o `pressed` (`scale(0.95)`) são feitos por overrides do tema, sem editar componentes.
- **Verificação até hoje é de build** (`tsc --noEmit`, `yarn workspace app lint/build`); não foi
  confirmado visualmente em navegador.
