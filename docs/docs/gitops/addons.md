# Catálogo de addons

Addons de terceiros seguem o padrão descrito em
[Padrões de Repositório](../repos.md#estrutura-interna-esperada-de-um-repo-gitopsapp):
`helm/<addon>/Chart.yaml` com `dependencies:` direto no chart upstream, sem
passar pelo chart genérico (esse é reservado para apps próprias).

| Repo | Addon(s) | O que faz |
|---|---|---|
| `gitops.core-addons` | cert-manager, External Secrets Operator, NGINX Gateway Fabric (+ CRDs), cloudflared, nvidia-device-plugin, argocd-image-updater | Base do cluster — praticamente todo outro `gitops.*` depende de algo daqui (Gateway API, `ClusterSecretStore/aws-ssm`, TLS) |
| `gitops.ai-core-addons` | Ollama, LiteLLM | Serving de LLM local: Ollama roda inferência na GPU, LiteLLM expõe um proxy compatível com a API da OpenAI na frente dele (`llm.cmoreira.dev`) |
| `gitops.cnpg` | CloudNativePG (operador Postgres) | Operador + primeiro banco consumidor (Backstage) |
| `gitops.crossplane` | Crossplane | Control plane para provisionamento declarativo de recursos |
| `gitops.monitoring` | Grafana Alloy, metrics-server | Observabilidade (coleta/telemetria) e métricas para HPA |
| `gitops.headlamp` | Headlamp | Dashboard web para o cluster (`headlamp.cmoreira.dev`), login via Microsoft Entra ID |
| `gitops.echoserver` | echoserver | Endpoint de teste para validar roteamento (Gateway API/HTTPRoute) |

## `gitops.core-addons` em detalhe

O único repo `gitops.*` que também carrega um stack **Terraform** próprio
(`terraform/`) — provisiona o lado AWS (IAM/OIDC) que sustenta o
`argocd-image-updater` e o pull de imagens do ECR a partir do cluster,
complementando o que `iac-aws-ecr-pipeline` provisiona para o lado do CI.

Addons instalados:

- **cert-manager** — emite os certificados TLS usados pelas Gateways (ver
  [Rede & Ingress](../architecture/networking.md))
- **External Secrets Operator** — via `ClusterSecretStore/aws-ssm`; ver
  [Secrets & Segurança](../architecture/secrets.md)
- **NGINX Gateway Fabric** — implementação da Gateway API que roteia todo
  tráfego de entrada
- **cloudflared** — o lado do túnel Cloudflare dentro do cluster
- **nvidia-device-plugin** — expõe a GPU do worker dedicado como recurso
  agendável
- **argocd-image-updater** — atualiza automaticamente a tag de imagem nos
  repos `gitops.*` quando uma nova imagem chega ao ECR

## `gitops.ai-core-addons` em detalhe

Modelado explicitamente sobre o padrão do `core-addons`. Dois componentes:

- **Ollama** (`otwld/ollama-helm`) — pinado no worker de GPU via
  `nodeSelector`, roda um modelo local (`llama3.1:8b`)
- **LiteLLM** (`BerriAI/litellm`) — proxy compatível com a API da OpenAI na
  frente do Ollama, exposto em `llm.cmoreira.dev`

Pré-requisitos documentados no próprio repo: o `nvidia-device-plugin` do
`core-addons`, o label manual `nvidia.com/gpu.present=true` no nó de GPU (sem
descoberta automática de nós), e um parâmetro no SSM
(`/homelab/litellm/master-key`) para o LiteLLM subir.

## `gitops.headlamp` em detalhe

Login federado via Microsoft Entra ID (OIDC direto, não via Dex/ArgoCD) —
mesmo padrão de autenticação humana do ArgoCD, ver
[Secrets & Segurança → Autenticação humana](../architecture/secrets.md#autenticacao-humana-microsoft-entra-id).

## Registro no Backstage

Vários addons (`gitops.cnpg`, `gitops.crossplane`, `gitops.echoserver`,
`gitops.headlamp`) têm `catalog-info.yaml`, registrando o componente no
catálogo do [Backstage](backstage.md) com `type: website`,
`lifecycle: lab`, e um link para a Application correspondente no ArgoCD.
