# n8n

!!! note "Source of truth"
    `gitops.n8n` (chart, Application, repo docs), `gitops.cnpg` (`kustomize/n8n`, the database) and
    `gitops.core-addons` (the cloudflared route). Authentication in
    [Authentication](../architecture/auth.md); backups in [Postgres backups](../gitops/backups.md).

## What it is

Automation orchestrator as a **single instance** (no queue mode), at `https://n8n.cmoreira.dev`. First
use: receive WhatsApp messages and forward them to a flow (the workflow itself is out of scope for this
page).

## Architecture

```
Internet → Cloudflare → cloudflared → NGINX Gateway Fabric (nginx-gateway-cmoreira-dev)
        → /webhook/whatsapp → Service n8n:5678
        → every other path  → n8n-oauth2-proxy (Entra ID) → Service n8n:5678
n8n → Service n8n-rw (CNPG Cluster "n8n", 1 instance, TLS verified with the CNPG CA)
```

- Namespace `n8n`. Deployment with 1 replica and `Recreate` strategy; pinned image
  (`n8nio/n8n:2.41.6`, never `latest`, updated by Renovate).
- **n8n and Postgres are pinned to `srv-k8s-master`** (x86, SSD). Both write a lot and must never land
  on the Raspberry Pis (SD card wear). Hence the nodeSelector and the control-plane toleration.
- `~/.n8n` is an `emptyDir`: state lives in Postgres and the encryption key comes from the environment.
  Binary data (file attachments) stays in the pod and is lost on restart; if workflows start handling
  media, a PVC has to be added.
- Resources: n8n 256Mi request / 1Gi limit; Postgres 256Mi / 512Mi. Probes on `/healthz` (liveness) and
  `/healthz/readiness`.

## Configuration (n8n variables)

| Variable | Value | Why |
|---|---|---|
| `DB_TYPE`, `DB_POSTGRESDB_*` | `postgresdb`, host `n8n-rw`, TLS with the CNPG CA | database on CNPG |
| `N8N_HOST`, `N8N_PROTOCOL`, `WEBHOOK_URL` | `n8n.cmoreira.dev`, `https`, `https://n8n.cmoreira.dev/` | correct webhook URLs behind the tunnel |
| `N8N_PROXY_HOPS` | `3` | cloudflared → Gateway → oauth2-proxy → n8n |
| `GENERIC_TIMEZONE`, `TZ` | `Europe/Lisbon` | |
| `N8N_DIAGNOSTICS_ENABLED`, `N8N_VERSION_NOTIFICATIONS_ENABLED`, `N8N_TEMPLATES_ENABLED`, `N8N_PERSONALIZATION_ENABLED` | `false` | no telemetry or outbound calls |
| `EXECUTIONS_DATA_PRUNE`, `EXECUTIONS_DATA_MAX_AGE`, `EXECUTIONS_DATA_PRUNE_MAX_COUNT` | `true`, `168` (**hours**, 7 days), `5000` | less disk writes |
| `N8N_ENCRYPTION_KEY` | from SSM `/homelab/n8n/encryption-key` via `ExternalSecret` | see below |

!!! danger "The encryption key"
    **Losing the `N8N_ENCRYPTION_KEY` means losing every credential saved in n8n**: the database becomes
    unreadable for them, and not even a backup fixes that. Keep a copy of the SSM parameter **outside
    AWS**.

## Exposure and access

- The `HTTPRoute` uses the existing `nginx-gateway-cmoreira-dev` Gateway (`*.cmoreira.dev` listener); no
  new Gateway was needed.
- The cloudflared route is in the `gitops.core-addons` ConfigMap (`kustomize/cloudflared`). cloudflared
  does not reload its configuration by itself: after changing the ConfigMap, restart the Deployment.
- DNS (`CNAME n8n` pointing at the tunnel) is manual, in the Cloudflare portal.
- Login is Entra ID through oauth2-proxy; only `/webhook/whatsapp` is public. **Never** open `/`,
  `/rest/*` or `/webhook-test/*`. The WhatsApp node must be a **Webhook with the fixed path `whatsapp`**
  (the WhatsApp Trigger node uses a UUID path, which would not match the public rule). Because the path
  is public, the workflow should validate Meta's `X-Hub-Signature-256`.

## Operations

- **Persistence tested** (2026-10-03): restarting the n8n pod and the Postgres pod preserves the owner
  and the settings; still to repeat with real workflows and credentials.
- **Backup:** daily at 03:00 UTC (see [Backups](../gitops/backups.md)).
- **Metrics:** `N8N_METRICS=true` with an explicit (allowlist) scrape in `gitops.monitoring` is proposed,
  not implemented.
- **Updating:** Renovate opens the PR with the new tag; read the n8n release notes before merging.

## Risks

- Single instance on a single node: n8n is down while the node reboots.
- Binary data is lost on restart (see `emptyDir`).
- The public webhook path is reachable by anyone; the protection is the signature validation in the
  workflow.
