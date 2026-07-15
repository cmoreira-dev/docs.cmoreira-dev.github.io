# Migração Istio → NGINX Gateway Fabric

Status: **planejamento** — nenhum corte de tráfego real foi feito. Nenhuma mudança neste
documento foi aplicada no cluster; ver seção [Estado atual](#estado-atual-levantamento) para
o que já existe.

## Contexto

O NGINX Gateway Fabric já está instalado no cluster (`nginx-gateway` namespace,
`GatewayClass nginx`, controller `gateway.nginx.org/nginx-gateway-controller`). O Istio hoje é
usado **apenas como ingress gateway** (`istio-system`) — não há mesh (sem sidecar injection em
nenhum namespace) nem políticas de auth/mTLS (`AuthorizationPolicy`, `RequestAuthentication`,
`PeerAuthentication` — nenhuma existe no cluster). Isso simplifica a migração: não existe
comportamento de auth "no Istio" para recriar, só roteamento HTTP.

A exposição externa é via Cloudflare Tunnel (`cloudflared` namespace), que hoje aponta
100% do tráfego para `http://istio-ingress.istio-system.svc.cluster.local:80`. TLS termina na
Cloudflare (`tls.mode: PASSTHROUGH` em todos os `Gateway` do Istio) — o novo Gateway do NGINX
também deve ficar em modo passthrough/HTTP puro, sem certificados a gerenciar no cluster.

## Estado atual (levantamento)

### Inventário — apps expostas via Istio

| Host | Namespace | VirtualService/Gateway | Backend | Tipo |
|---|---|---|---|---|
| `argocd.cmoreira.dev` | istio-system | `argocd-vs` / `argocd-gw` | `argocd-server.argocd.svc:443` | in-cluster |
| `headlamp.cmoreira.dev` | istio-system | `headlamp-vs` / `headlamp-gw` | `headlamp.headlamp.svc:80` | in-cluster |
| `local.cmoreira.dev/echo` | istio-system | `generic-vs` / `istio-gw` | `echoserver-service.echoserver.svc:80` | in-cluster |
| `local.cmoreira.dev/grafana` | istio-system | `generic-vs` / `istio-gw` | `grafana-service.grafana.svc:3000` | **órfã** — namespace/Service `grafana` não existe no cluster hoje (stack de monitoring atual só tem Alloy) |
| `n8n.cmoreira.dev` | istio-system | `n8n-vs` / `n8n-gw` + `DestinationRule` + `ServiceEntry` | `n8n.local:5678` (IP externo `192.168.1.99`, TLS disabled) | **externo ao cluster** |
| `proxmox.cmoreira.dev` | istio-system | `proxmox-vs` / `proxmox-gw` + `DestinationRule` + `ServiceEntry` | `proxmox.local:8006` (IP externo `192.168.1.2`, TLS simple + `insecureSkipVerify`) | **externo ao cluster** |

Fonte: `gitops.istio/kustomize/istio-config/*.yaml` (repo, sincronizado via Application
`istio-config`).

### NGINX Gateway Fabric

- Namespace: `nginx-gateway`. Pod rodando em `rasp-k8s-worker-1` (ARM64) — a imagem é
  multi-arch, ok.
- `GatewayClass/nginx` aceito (`gateway.nginx.org/nginx-gateway-controller`).
- **Nenhum `Gateway` (Gateway API) nem `HTTPRoute` criado ainda** — hoje o NGINX Gateway
  Fabric está instalado mas não expõe nada. Isso é o primeiro passo da Fase 0.
- CRDs experimentais do NGF disponíveis no cluster, incluindo `BackendTLSPolicy` — relevante
  para o caso do Proxmox (ver [Riscos](#riscos-e-rollback-por-fase)).

### Auth / mTLS

Nenhuma. Sidecar injection desabilitado em todos os namespaces
(`kubectl get ns -L istio-injection` sem nenhum valor setado). O OIDC do Headlamp (Entra ID via
Dex) é resolvido pelo próprio Headlamp (`headlamp.config.oidc`, secret `headlamp-oidc` via
ExternalSecret) — não depende do Istio e não muda com esta migração.

### Cloudflare Tunnel

`cloudflared-config` ConfigMap (`gitops.core-addons/kustomize/cloudflared/configmap.yaml`) —
uma entrada por host, todas apontando pro mesmo Service do Istio ingress:

```yaml
ingress:
  - hostname: local.cmoreira.dev
    service: http://istio-ingress.istio-system.svc.cluster.local:80
  - hostname: proxmox.cmoreira.dev
    service: http://istio-ingress.istio-system.svc.cluster.local:80
  - hostname: argocd.cmoreira.dev
    service: http://istio-ingress.istio-system.svc.cluster.local:80
  - hostname: headlamp.cmoreira.dev
    service: http://istio-ingress.istio-system.svc.cluster.local:80
  - hostname: n8n.cmoreira.dev
    service: http://istio-ingress.istio-system.svc.cluster.local:80
  - service: http_status:404
```

Migrar um host = trocar a linha `service:` daquele hostname para o Service do NGINX Gateway
Fabric (`nginx-gateway-fabric.nginx-gateway.svc.cluster.local:443` ou um `Gateway`
NodePort/LoadBalancer dedicado — a definir na Fase 0) e reload do `cloudflared`. Rollback é a
mesma troca ao contrário.

### ExternalSecrets

`ClusterSecretStore/aws-ssm` (AWS SSM Parameter Store, `us-east-1`) — já usado por Headlamp e
outros. Nada muda aqui com a migração de ingress.

### Achado colateral (fora do escopo desta migração)

O `ApplicationSet/gitops-repos` no cluster está com a regex
`^gitops\.(?!template).*` **quebrada** (Go RE2 não suporta negative lookahead — erro
`ApplicationGenerationFromParamsError`, status `Degraded`). Ele continua servindo as
Applications já geradas anteriormente, mas não está reconciliando novos repos `gitops.*`
correndo agora. Isso é anterior a esta migração e não bloqueia o trabalho abaixo, mas deve ser
corrigido separadamente (regex sem lookahead, ex. dois generators com include/exclude, ou
filtro por path).

## Mapeamento de equivalência Istio → Gateway API

| Conceito Istio | Equivalente Gateway API / NGINX Gateway Fabric |
|---|---|
| `Gateway` (istio, `tls.mode: PASSTHROUGH`, porta 80) | `Gateway` (Gateway API) com listener HTTP na porta 80, `parametersRef` para `GatewayClass/nginx` |
| `VirtualService` (host + `uri.prefix` match + `route.destination`) | `HTTPRoute` (`parentRefs` → Gateway, `hostnames`, `rules[].matches[].path`, `rules[].backendRefs`) |
| `DestinationRule` (TLS settings pro backend) | `BackendTLSPolicy` (Gateway API experimental, suportado pelo NGF) — **com ressalva**, ver risco do Proxmox abaixo |
| `ServiceEntry` (endpoint externo ao cluster) | Não existe equivalente nativo no Gateway API. Precisa de um `Service` sem selector + `EndpointSlice` manual (ou `ExternalName`) apontando pro IP externo, referenciado como `backendRef` do `HTTPRoute` |
| TLS termination | Sem mudança — continua na Cloudflare Tunnel (passthrough em ambos os stacks) |
| Auth (`AuthorizationPolicy`/`RequestAuthentication`) | N/A — não existia nada a migrar |

Traduzindo cada VirtualService específico:

- **argocd, headlamp**: 1:1 direto — `HTTPRoute` simples, `PathPrefix /`, um `backendRef` pro
  Service in-cluster existente. Sem timeouts/retries customizados hoje, então nada a
  preservar além do host e do backend.
- **local.cmoreira.dev (/echo)**: `HTTPRoute` com `matches[].path.type: PathPrefix, value: /echo`
  → `backendRef` pro `echoserver-service`. Direto.
- **local.cmoreira.dev (/grafana)**: **recomendação: não migrar**. O Service `grafana` não
  existe no cluster (verificado: `kubectl get svc -n grafana` → não encontrado; o
  `gitops.monitoring` atual só tem Alloy). É uma rota órfã, provavelmente de um setup antigo de
  Prometheus/Grafana já removido. Confirmar com o time antes de descartar, mas não faz sentido
  recriar uma rota para um backend que não existe.
- **n8n, proxmox**: precisam de um `Service` intermediário (sem selector, com
  `EndpointSlice` manual apontando pro IP externo — `192.168.1.99:5678` e `192.168.1.2:8006`
  respectivamente) para servir de `backendRef` do `HTTPRoute`, já que Gateway API não tem
  conceito de `ServiceEntry`. Ver riscos abaixo para o caso do Proxmox (TLS skip-verify).

## Estratégia de corte por fases

### Fase 0 — Validar o NGINX Gateway Fabric sem tráfego real

- Criar um `Gateway` (Gateway API) em `nginx-gateway` sobre o `GatewayClass/nginx`, listener
  HTTP porta 80 (ou 8080, para não colidir enquanto o Istio ainda escuta na 80 do seu próprio
  Service).
- Criar um `HTTPRoute` de teste (ex. host `nginx-test.cmoreira.dev` ou usando
  `echoserver` como backend, sem trocar o Cloudflare Tunnel) e validar com
  `kubectl port-forward` ou `curl` direto no ClusterIP do `nginx-gateway-fabric` Service.
- Critério de saída: `HTTPRoute` com status `Accepted: True` e resposta HTTP correta via
  port-forward.
- Nenhum impacto em produção — não toca no Cloudflare Tunnel nem no Istio.

### Fase 1 — Piloto de baixo risco (Headlamp)

- Por que Headlamp: app interna, sem dependências externas (n8n/proxmox), sem lógica de
  path-prefix compartilhado (diferente do `local.cmoreira.dev`), e já isolada num namespace
  próprio.
- Criar `HTTPRoute` para `headlamp.cmoreira.dev` apontando pro `Service` existente
  (`headlamp.headlamp.svc.cluster.local:80`) — **o Deployment do Headlamp não muda**, só a
  camada de ingress.
- Rodar em paralelo ao `headlamp-vs`/`headlamp-gw` do Istio (ambos ativos, tráfego real ainda
  vai pelo Istio via Cloudflare).
- Validar por IP/porta direta (port-forward do Service do NGINX Gateway Fabric, ou expor
  temporariamente via NodePort) que o `HTTPRoute` responde igual ao VirtualService atual.

### Fase 2 — Cutover do piloto

- Trocar a entrada `headlamp.cmoreira.dev` no `cloudflared-config` ConfigMap para apontar pro
  Service do NGINX Gateway Fabric.
- Reload do `cloudflared` (rollout restart do Deployment, ou reload de config se suportado).
- Rollback: reverter a entrada do ConfigMap para `istio-ingress.istio-system.svc.cluster.local:80`
  e reload — o `headlamp-vs`/`headlamp-gw` do Istio continua intacto até a Fase 4, então o
  rollback é imediato e sem downtime adicional.
- Critério de saída: `headlamp.cmoreira.dev` respondendo via NGINX Gateway Fabric por pelo
  menos 24-48h sem incidentes antes de seguir para a Fase 3.

### Fase 3 — Demais apps, uma a uma

Ordem sugerida (do mais simples ao mais arriscado):

1. `argocd.cmoreira.dev` — 1:1 direto, mas alto valor de negócio (é o próprio ArgoCD) → migrar
   com cautela, fora de horário crítico.
2. `local.cmoreira.dev/echo` — baixo risco, só testing.
3. (decidir sobre `/grafana` — provavelmente remover, não migrar).
4. `n8n.cmoreira.dev` — precisa do `Service` externo intermediário (sem TLS, mais simples).
5. `proxmox.cmoreira.dev` — precisa do `Service` externo intermediário **e** resolver o gap de
   TLS skip-verify (ver riscos). Migrar por último, só depois de validar `BackendTLSPolicy` ou
   decidir uma alternativa.

Mesma sequência de cada fase 1→2 (paralelo → validar → trocar Cloudflare Tunnel → observar →
rollback se necessário) para cada app.

### Fase 4 — Decomissionamento do Istio

Só depois que **todas** as apps da Fase 3 estiverem migradas e estáveis por pelo menos uma
semana:

- Remover as entradas do `cloudflared-config` que ainda referenciem o Istio (deve já estar
  vazio a essa altura).
- Deletar os `VirtualService`/`Gateway`/`DestinationRule`/`ServiceEntry` do
  `gitops.istio/kustomize/istio-config` (PR de remoção, não `kubectl delete` manual).
- Escalar `istio-ingress` e `istiod` para 0 réplicas primeiro (validação de "ninguém mais
  depende disso"), esperar alguns dias, só então remover a Application `istio` e
  `istio-config` do ArgoCD.
- Remover o Helm release (`gitops.istio/helm/istio-system`) via PR deletando o repo/Application.
- Limpar CRDs do Istio (`kubectl get crd | grep istio.io`) e o namespace `istio-system`.

## Riscos e rollback por fase

| Fase | Risco | Mitigação / Rollback |
|---|---|---|
| 0 | Nenhum — sem tráfego real envolvido | N/A |
| 1 | HTTPRoute mal configurado não afeta produção (Istio continua servindo o tráfego real) | Basta corrigir o HTTPRoute; sem impacto a usuários |
| 2 (piloto) | Cloudflare Tunnel aponta pro NGINX e o Service/HTTPRoute está errado → Headlamp fica inacessível | Reverter a linha do ConfigMap `cloudflared-config` pro Service do Istio + reload; Istio-side continua configurado, então é reversível em minutos |
| 3 (ArgoCD) | Se o HTTPRoute do ArgoCD tiver problema, perde-se acesso à UI do ArgoCD (mas não ao cluster — `kubectl` continua funcionando) | Mesmo rollback do ConfigMap; considerar migrar ArgoCD fora de horário de mudança ativa |
| 3 (n8n) | `Service` sem selector mal configurado (`EndpointSlice` com IP/porta errados) quebra o n8n | Testar via port-forward antes do cutover; rollback = reverter ConfigMap |
| 3 (proxmox) | **Gap real**: Gateway API não tem um "insecureSkipVerify" trivial equivalente ao `DestinationRule` do Istio. `BackendTLSPolicy` do NGF valida contra uma CA — se o Proxmox usa certificado self-signed (comum), pode não ser possível reproduzir o skip-verify sem workaround (ex. terminar TLS do lado do backend de outra forma, ou aceitar expor o Proxmox só via HTTP interno se a rede permitir, ou manter esse host específico no Istio por mais tempo) | **Não migrar o Proxmox até validar isso na prática** (Fase 0 estendida: testar `BackendTLSPolicy` contra o Proxmox real antes de comprometer a data de corte) |
| 4 | Remoção do Istio quebra algo não mapeado no inventário | Escalar para 0 réplicas antes de deletar de vez; manter os manifests em um branch por algumas semanas antes de apagar de vez |

## Checklist de decomissionamento do Istio

- [ ] Todas as apps do inventário migradas e estáveis (Fases 1–3 completas)
- [ ] `/grafana` route resolvida (migrada ou formalmente removida)
- [ ] Gap de TLS do Proxmox resolvido ou aceito com plano B documentado
- [ ] `cloudflared-config` sem nenhuma referência a `istio-ingress.istio-system.svc.cluster.local`
- [ ] `istio-ingress`/`istiod` escalados a 0 por >= 1 semana sem incidentes
- [ ] PR removendo `kustomize/istio-config/*` no `gitops.istio`
- [ ] PR removendo a Application `istio` (helm/istio-system) do ArgoCD
- [ ] CRDs `*.istio.io` removidos do cluster
- [ ] Namespace `istio-system` removido
- [ ] Este documento atualizado com data de conclusão e link pro PR final

## Próximos passos

Este documento cobre o planejamento. Os manifests da Fase 0/1 (Gateway + HTTPRoute do piloto
Headlamp) ainda não foram preparados — avisar se quiser que eu prepare esse PR como próximo
passo.
