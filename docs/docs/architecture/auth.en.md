# Authentication (Entra ID)

!!! note "Source of truth"
    `gitops.n8n` and `gitops.ai-core-addons` (oauth2-proxy and LiteLLM SSO), `homelab-bootsrap-k3s`
    (`k8s-addons/argocd/values.yaml.j2`, ArgoCD Dex and RBAC), `gitops.core-addons` (Dex
    ExternalSecret). The Entra application registrations are created by hand, not in IaC yet.

## Overview

There is **one identity provider**: Microsoft Entra ID in the `cmoreira-dev` tenant. Every tool is an
OIDC client of it. There is no Cloudflare Access in front of the `*.cmoreira.dev` hosts: each host
protects itself, either with the app's native login or, where the app has no SSO, with an
**oauth2-proxy** in front.

| Tool | How it authenticates | Who gets in |
|---|---|---|
| ArgoCD | Dex (Microsoft connector), Entra group `Platform Engineering` | group members; admin by email in RBAC, everyone else read-only |
| Headlamp | Native OIDC against Entra ID | `cassio@cmoreira.dev` (cluster-admin) |
| Backstage | Native Microsoft provider | domains `cmoreira.dev` and `rapporthub.pt` |
| LiteLLM (UI) | Native Microsoft SSO; break-glass login (`/fallback/login`) behind oauth2-proxy | SSO users get `internal_user` |
| LiteLLM (API) | API keys (no human login) | whoever holds the key |
| n8n | oauth2-proxy (Entra ID) in front of the editor; n8n's own login stays as a second layer | emails on the allow list |
| teupadel.com | Google OAuth and magic link, in the app itself (customers) | end users |
| AWS | Federated login via Entra ID | see [Secrets & Security](secrets.md) |

## oauth2-proxy (n8n and LiteLLM)

n8n community edition has no SSO (OIDC and SAML are in the paid plans). The fix is one
[oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy) per app, in **reverse-proxy** mode,
between the Gateway and the app.

Why not a single proxy on the Gateway:

- The installed Gateway API (1.5.1, standard channel) has no `ExternalAuth` filter.
- NGINX Gateway Fabric's `OIDC` filter is NGINX Plus only, and the cluster runs open source NGINX.
- `SnippetsFilter` (nginx `auth_request`) is off in the controller; enabling it would change the
  ingress for the whole cluster.

Setup (`oauth2-proxy` chart 10.7.1, app 7.15.5):

- Provider `entra-id`, 2 stateless replicas (cookie sessions), allowed emails listed in `values.yaml`
  (`authenticatedEmailsFile`).
- Cookie on the `.cmoreira.dev` domain: one sign-in covers every protected subdomain.
- `session-cookie-minimal`: only the identity goes in the cookie. Without it the Entra session exceeds
  4 KB, the proxy splits it into several `Set-Cookie` headers and NGINX answers **502** on
  `/oauth2/callback` (`upstream sent too big header`).
- The `HTTPRoute` splits the traffic. For n8n, `/webhook/whatsapp` (PathPrefix) goes straight to n8n,
  because Meta cannot log in; everything else (`/`, `/rest/*`, `/webhook-test/*`, `/oauth2/*`) goes
  through the proxy. For LiteLLM only the break-glass login and the docs go through the proxy; the UI
  uses native SSO and the API goes direct, protected by key.
- Rollout in two PRs: first the proxy is deployed **receiving no traffic** (validated with
  `port-forward`), then a second PR switches the route. Reverting the route PR undoes the cutover.

## Application registrations in Entra

| Registration | Used by | Redirect URIs | Secret |
|---|---|---|---|
| `oauth2-proxy homelab` (`077cba4b-…`) | oauth2-proxy for n8n and LiteLLM; LiteLLM native SSO | `https://n8n.cmoreira.dev/oauth2/callback`, `https://llm.cmoreira.dev/oauth2/callback`, `https://llm.cmoreira.dev/sso/callback` | SSM `/homelab/oauth2-proxy/{client-id,client-secret,cookie-secret}` |
| `Argocd` (`ef1ad5d7-…`) | ArgoCD Dex | `https://argocd.cmoreira.dev/api/dex/callback` | SSM `/homelab/argocd/dex-microsoft-client-secret` |

The client secrets expire on **2027-10-03**. All of them reach the cluster through `ExternalSecret`;
none lives in a ConfigMap or in git.

### Rotating a client secret

1. In Entra, create a new credential on the registration (do not delete the old one).
2. Update the SSM parameter. ESO syncs within 1 hour (24 hours for the oauth2-proxy ones); restart the
   consumer to pick it up right away.
3. Test the login, and only then delete the old credential in Entra.

## ArgoCD: who gets in

- The Dex Microsoft connector only accepts members of the Entra group **`Platform Engineering`** (the
  name must match exactly, space included). Giving someone access therefore means **adding them to the
  group in Entra** (a manual step, outside GitOps).
- The role comes from `argocd-rbac-cm`: admin by email for `cassio@cmoreira.dev` and
  `cassio@rapporthub.pt`; any other group member falls back to `role:readonly` (`policy.default`).
- `argocd-cm` only holds the reference `$argocd-dex-microsoft:clientSecret`; the Secret comes from SSM
  through an `ExternalSecret` (label `app.kubernetes.io/part-of: argocd`, required for ArgoCD to
  resolve the reference).
- ArgoCD is installed by `helm upgrade` outside GitOps; the values live in `values.yaml.j2` in the
  bootstrap repo. The local `admin` account is still enabled as a last resort.
- Chart 10.x: `global.networkPolicy.create=false`, because flannel does not enforce NetworkPolicy.

## Pitfalls we hit

- **Group name mismatch** (`PlatformEngineering` in Dex, `Platform Engineering` in Entra): everyone got
  "not in any of the required groups".
- **Session cookie over 4 KB**: 502 on the oauth2-proxy callback (see above).
- **Double login on LiteLLM**: with the proxy in front of the UI, LiteLLM still asked for its
  username/password form. Native SSO solved it.
- **Slow LiteLLM sync**: the chart runs a database migration Job as a hook on every sync, so any change
  to the app takes a few minutes.
