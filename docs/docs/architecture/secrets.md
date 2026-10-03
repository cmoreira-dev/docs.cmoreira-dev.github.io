# Secrets & Segurança

Duas regras seguidas em toda a infra: **nenhuma credencial de longa duração
fica gravada em disco ou em Git**, e **cada consumidor de segredo pede
exatamente o que precisa**, no menor escopo possível.

## CI → AWS: OIDC, sem access keys

Os workflows de build/push das apps (`api.*`, `ui.*`) não têm nenhuma AWS
access key configurada como secret do GitHub. Em vez disso:

1. `infra-as-code/iac-aws-ecr-pipeline` provisiona um **OIDC identity provider**
   do GitHub Actions na conta AWS, e uma **IAM Role** assumível apenas por
   workflows rodando na branch `main` dos repos autorizados.
2. O workflow reutilizável de build/push (`cmoreira-dev/.github`) troca o token
   OIDC do job por credenciais temporárias dessa role, via
   `aws-actions/configure-aws-credentials`.
3. A credencial expira com o job — nada para rotacionar, nada para vazar.

Isso está descrito com mais detalhe em [Build & Registry](../cicd/build-registry.md).

## Cluster → AWS: External Secrets Operator

Dentro do cluster, nenhuma app injeta o próprio segredo em texto puro em manifest
algum. O padrão, uniforme em todos os apps e addons:

```mermaid
flowchart LR
    ssm["AWS SSM<br/>Parameter Store"] -->|"ClusterSecretStore<br/>(aws-ssm)"| eso["External Secrets<br/>Operator"]
    eso -->|materializa| secret["Secret nativo<br/>do Kubernetes"]
    secret -->|envFrom / volume| pod["Pod"]
```

- Um `ClusterSecretStore` chamado `aws-ssm` (definido em `gitops.core-addons`) é
  o único ponto de conexão entre o cluster e o SSM Parameter Store.
- Cada app declara um `ExternalSecret` (via o campo `externalSecrets` do
  [chart genérico](../kubernetes/generic-app-chart.md)) apontando para o(s)
  parâmetro(s) SSM que precisa — por exemplo, `api.teupadel.com` referencia
  `/homelab/teupadel/anthropic-api-key`.
- A policy IAM do usuário `external-secrets-operator` limita por **prefixo de
  path**: hoje `parameter/homelab/*` e `parameter/teupadel/*` (segredos do
  produto: Google OAuth, chave de sessão, SES). Um path fora da policy faz o
  `ExternalSecret` falhar (`SecretSyncedError`, `AccessDenied`) e o app perde
  essas variáveis em silêncio. A policy é gerenciada em
  `iac.homelab-live-infra` (`aws/cmoreira-dev/us-east-1/iam-external-secrets`,
  `ssm_path_prefixes`).
- O Secret gerado vive só no namespace da app; não há um Secret compartilhado
  entre apps.
- Apps sem segredo real (como `api.ia.local-sara`, um scraper público sem chave
  de API) simplesmente não habilitam `externalSecrets` — não existe um Secret
  vazio "por padrão".

## Pull de imagens privadas do ECR

O ECR é privado; cada app que precisa puxar imagem de lá habilita
`ecrPullSecret` no chart genérico, que gera um
`Secret/kubernetes.io/dockerconfigjson` a partir de um `ClusterGenerator`
(`ecr-token`, gerenciado em `gitops.core-addons`) — o token é obtido
dinamicamente da AWS, não é uma credencial estática copiada para o cluster.

## Atualização automática de tag de imagem

O `argocd-image-updater` (addon em `gitops.core-addons`) tem credenciais próprias
para ler o ECR e escrever de volta nos repos `gitops.*` — também via
`ExternalSecret`, nunca copiadas manualmente.

## Autenticação humana: Microsoft Entra ID

Os segredos acima são todos máquina-a-máquina. Para acesso **humano**, o
padrão é diferente: as ferramentas com privilégio amplo sobre a infra são
federadas via **Microsoft Entra ID** — sem senha local para gerenciar ou
rotacionar.

| Ferramenta | Mecanismo |
|---|---|
| ArgoCD | SSO via Dex, conectado ao Entra ID (grupo `Platform Engineering`) |
| Headlamp | OIDC direto contra o Entra ID (`headlamp.config.oidc`) |
| Backstage | Provider Microsoft nativo |
| LiteLLM (UI) | SSO Microsoft nativo |
| n8n | oauth2-proxy (Entra ID) na frente do editor |
| Conta AWS | Login federado via Entra ID |

Esse padrão vale para as duas nuvens (AWS e o próprio Entra ID/Azure) e para as
ferramentas operacionais (ArgoCD, Headlamp, Backstage, LiteLLM, n8n). Detalhes,
registros no Entra e rotação de segredos em [Autenticação](auth.md). As
aplicações de produto (`teupadel.com`, Sara) não usam o Entra ID: o teupadel tem
login próprio para clientes, e o Sara é um scraper público sem login.

## Resumo por tipo de credencial

| Credencial | Onde mora a fonte | Como chega ao consumidor |
|---|---|---|
| Deploy AWS (CI de app) | IAM Role (OIDC) | Assumida por job, expira com o job |
| Chaves de API de app (ex.: Anthropic) | AWS SSM Parameter Store | `ExternalSecret` → Secret no namespace da app |
| Pull de imagem ECR | Token dinâmico via `ClusterGenerator` | `ExternalSecret` → `dockerconfigjson` |
| Login no ArgoCD | Microsoft Entra ID (Dex) | Login federado, sem senha local |
| Login no Headlamp | Microsoft Entra ID (OIDC) | Login federado, sem senha local |
| Login no console/CLI AWS | Microsoft Entra ID | Login federado, sem access key de longa duração |
