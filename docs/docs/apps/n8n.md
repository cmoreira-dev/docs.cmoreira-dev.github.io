# n8n

!!! note "Fonte da verdade"
    `gitops.n8n` (chart, Application, docs do repo), `gitops.cnpg` (`kustomize/n8n`, o banco) e
    `gitops.core-addons` (rota no cloudflared). Autenticação em [Autenticação](../architecture/auth.md);
    backups em [Backups do Postgres](../gitops/backups.md).

## O que é

Orquestrador de automações em **instância única** (sem queue mode), em `https://n8n.cmoreira.dev`.
Primeiro uso: receber mensagens do WhatsApp e encaminhá-las a um fluxo (o workflow em si está fora do
escopo desta página).

## Arquitetura

```
Internet → Cloudflare → cloudflared → NGINX Gateway Fabric (nginx-gateway-cmoreira-dev)
        → /webhook/whatsapp → Service n8n:5678
        → demais caminhos   → n8n-oauth2-proxy (Entra ID) → Service n8n:5678
n8n → Service n8n-rw (Cluster CNPG "n8n", 1 instância, TLS verificado com o CA do CNPG)
```

- Namespace `n8n`. Deployment com 1 réplica e estratégia `Recreate`; imagem fixada (`n8nio/n8n:2.41.6`,
  nunca `latest`, atualizada pelo Renovate).
- **n8n e Postgres ficam fixados no `srv-k8s-master`** (x86, SSD). Os dois escrevem bastante e nunca
  devem ir para os Raspberry Pi (desgaste do cartão SD). Por isso o nodeSelector e a toleration do
  control-plane.
- `~/.n8n` é um `emptyDir`: o estado vive no Postgres e a chave de criptografia vem do ambiente.
  Anexos binários (arquivos) ficam no pod e se perdem em restart; se os workflows passarem a tratar
  mídia, é preciso adicionar um PVC.
- Recursos: n8n 256Mi pedido / 1Gi limite; Postgres 256Mi / 512Mi. Probes em `/healthz` (liveness) e
  `/healthz/readiness`.

## Configuração (variáveis do n8n)

| Variável | Valor | Motivo |
|---|---|---|
| `DB_TYPE`, `DB_POSTGRESDB_*` | `postgresdb`, host `n8n-rw`, TLS com o CA do CNPG | banco no CNPG |
| `N8N_HOST`, `N8N_PROTOCOL`, `WEBHOOK_URL` | `n8n.cmoreira.dev`, `https`, `https://n8n.cmoreira.dev/` | URLs de webhook corretas atrás do túnel |
| `N8N_PROXY_HOPS` | `3` | cloudflared → Gateway → oauth2-proxy → n8n |
| `GENERIC_TIMEZONE`, `TZ` | `Europe/Lisbon` | |
| `N8N_DIAGNOSTICS_ENABLED`, `N8N_VERSION_NOTIFICATIONS_ENABLED`, `N8N_TEMPLATES_ENABLED`, `N8N_PERSONALIZATION_ENABLED` | `false` | sem telemetria nem chamadas externas |
| `EXECUTIONS_DATA_PRUNE`, `EXECUTIONS_DATA_MAX_AGE`, `EXECUTIONS_DATA_PRUNE_MAX_COUNT` | `true`, `168` (**horas**, 7 dias), `5000` | menos escrita em disco |
| `N8N_ENCRYPTION_KEY` | do SSM `/homelab/n8n/encryption-key` via `ExternalSecret` | ver abaixo |

!!! danger "A chave de criptografia"
    **Perder a `N8N_ENCRYPTION_KEY` significa perder todas as credenciais salvas no n8n**: o banco fica
    ilegível para elas, e nem o backup resolve. Mantenha uma cópia do parâmetro SSM **fora da AWS**.

## Exposição e acesso

- A `HTTPRoute` usa o Gateway existente `nginx-gateway-cmoreira-dev` (listener `*.cmoreira.dev`); não
  precisou de Gateway novo.
- A rota no cloudflared está no ConfigMap de `gitops.core-addons` (`kustomize/cloudflared`). O cloudflared
  não recarrega a configuração sozinho: depois de mudar o ConfigMap, reinicie o Deployment.
- O DNS (`CNAME n8n` apontando para o túnel) é manual, no portal da Cloudflare.
- O login é o Entra ID via oauth2-proxy; só `/webhook/whatsapp` fica público. **Nunca** liberar `/`,
  `/rest/*` nem `/webhook-test/*`. O nó do WhatsApp deve ser um **Webhook com o caminho fixo `whatsapp`**
  (o nó WhatsApp Trigger usa um caminho com UUID, que não casaria com a regra pública). Como o caminho é
  público, o workflow deve validar o `X-Hub-Signature-256` da Meta.

## Operação

- **Persistência testada** (2026-10-03): reiniciar o pod do n8n e o do Postgres preserva o owner e as
  configurações; falta repetir com workflows e credenciais reais.
- **Backup:** diário às 03:00 UTC (ver [Backups](../gitops/backups.md)).
- **Métricas:** `N8N_METRICS=true` com scrape explícito (allowlist) no `gitops.monitoring` está proposto,
  não implementado.
- **Atualizar:** o Renovate abre o PR com a nova tag; revise as notas da versão do n8n antes de mergear.

## Riscos

- Instância única em um nó só: o n8n fica fora do ar enquanto o nó reinicia.
- Anexos binários se perdem em restart (ver `emptyDir`).
- O caminho público do webhook é alcançável por qualquer pessoa; a proteção é a validação da assinatura
  no workflow.
