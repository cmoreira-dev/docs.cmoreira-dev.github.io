# Argo CD

O ArgoCD é o único componente do cluster que aplica manifests diretamente —
todo o resto do fluxo de deploy é git push + reconciliação automática. Ver
[Padrão GitOps](gitops/pattern.md) para o mecanismo completo, ponta a ponta.

## Instalação

Instalado via Helm (chart oficial `argo/argo-cd`) pelos playbooks Ansible no
bootstrap do cluster — ver [Bootstrap do cluster](iac/bootstrap.md). SSO
configurado via Dex, conectado ao Microsoft Entra ID — não há usuário/senha
local de uso corrente.

Exposto em `argocd.cmoreira.dev`, atrás da mesma cadeia Cloudflare Tunnel →
Gateway API descrita em [Rede & Ingress](architecture/networking.md).

## `AppProject/homelab`

Um único `AppProject`, criado no bootstrap, com escopo amplo:

- Qualquer repositório da org `cmoreira-dev/*` como source permitido
- Qualquer destino/namespace no cluster
- Qualquer tipo de recurso Kubernetes

Isso mantém o projeto simples num homelab de operador único — não há
segregação de projeto por time/ambiente, porque não há múltiplos times nem
múltiplos ambientes no cluster.

## `ApplicationSet/gitops-repos`

O gerador de descoberta automática — detalhado em
[Padrão GitOps → Camada 1](gitops/pattern.md#camada-1-descoberta-automatica).
Resumo:

- Gerador `scmProvider` contra a org GitHub, filtro `^gitops\..*`
- Poll a cada 300s por novos repositórios `gitops.*`
- Uma `Application` por repositório encontrado, apontada para `argocd/` daquele
  repo

## Política de sincronização

Toda `Application` gerada usa:

```yaml
syncPolicy:
  automated:
    prune: true      # remove recursos que saíram do Git
    selfHeal: true    # reverte qualquer mudança feita fora do Git
  syncOptions:
    - ServerSideApply=true
    - CreateNamespace=true
```

Não há sync manual no fluxo normal — qualquer divergência entre cluster e Git
é corrigida automaticamente pelo `selfHeal`.

## Atualização de imagem

O `argocd-image-updater` (addon em `gitops.core-addons`) observa o ECR e
escreve de volta nos repos `gitops.*` quando uma nova tag de imagem chega,
usando credenciais próprias via `ExternalSecret` — ver
[Secrets & Segurança](architecture/secrets.md).
