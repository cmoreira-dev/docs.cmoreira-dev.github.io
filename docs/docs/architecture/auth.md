# Autenticação (Entra ID)

!!! note "Fonte da verdade"
    `gitops.n8n` e `gitops.ai-core-addons` (oauth2-proxy e SSO do LiteLLM), `homelab-bootsrap-k3s`
    (`k8s-addons/argocd/values.yaml.j2`, Dex e RBAC do ArgoCD), `gitops.core-addons`
    (ExternalSecret do Dex). Os registros de aplicação no Entra são criados à mão, ainda sem IaC.

## Visão geral

Existe **um provedor de identidade**: o Microsoft Entra ID do tenant `cmoreira-dev`. Cada ferramenta é
um cliente OIDC dele. Não há Cloudflare Access na frente dos hosts `*.cmoreira.dev`: cada host se
protege sozinho, com login nativo no app ou, onde o app não tem SSO, com um **oauth2-proxy** na frente.

| Ferramenta | Como autentica | Quem entra |
|---|---|---|
| ArgoCD | Dex (conector Microsoft), grupo Entra `Platform Engineering` | membros do grupo; admin por e-mail no RBAC, o resto só leitura |
| Headlamp | OIDC nativo contra o Entra ID | `cassio@cmoreira.dev` (cluster-admin) |
| Backstage | Provider Microsoft nativo | domínios `cmoreira.dev` e `rapporthub.pt` |
| LiteLLM (UI) | SSO Microsoft nativo; login de emergência (`/fallback/login`) atrás do oauth2-proxy | usuários SSO entram como `internal_user` |
| LiteLLM (API) | chaves de API (sem login humano) | quem tem a chave |
| n8n | oauth2-proxy (Entra ID) na frente do editor; o login próprio do n8n continua como segunda camada | e-mails na lista permitida |
| teupadel.com | Google OAuth e link mágico, do próprio app (clientes) | usuários finais |
| AWS | Login federado via Entra ID | ver [Secrets & Segurança](secrets.md) |

## oauth2-proxy (n8n e LiteLLM)

O n8n community não tem SSO (OIDC e SAML são dos planos pagos). A solução foi um
[oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy) por app, em modo **reverse-proxy**, entre o
Gateway e o app.

Por que não um proxy único no Gateway:

- A Gateway API instalada (1.5.1, canal standard) não tem o filtro `ExternalAuth`.
- O filtro `OIDC` do NGINX Gateway Fabric é exclusivo do NGINX Plus, e o cluster roda o NGINX open source.
- O `SnippetsFilter` (nginx `auth_request`) está desligado no controlador; ligá-lo mexeria no ingress de
  todo o cluster.

Configuração (chart `oauth2-proxy` 10.7.1, app 7.15.5):

- Provider `entra-id`, 2 réplicas sem estado (sessão em cookie), lista de e-mails permitidos em
  `values.yaml` (`authenticatedEmailsFile`).
- Cookie no domínio `.cmoreira.dev`: um login vale para todos os subdomínios protegidos.
- `session-cookie-minimal`: só a identidade vai no cookie. Sem isso a sessão do Entra passa de 4 KB, o
  proxy divide em vários `Set-Cookie` e o NGINX responde **502** em `/oauth2/callback`
  (`upstream sent too big header`).
- A `HTTPRoute` divide o tráfego. No n8n, `/webhook/whatsapp` (PathPrefix) vai direto ao n8n, porque a
  Meta não faz login; todo o resto (`/`, `/rest/*`, `/webhook-test/*`, `/oauth2/*`) passa pelo proxy.
  No LiteLLM só o login de emergência e a documentação passam pelo proxy; a UI usa SSO nativo e a API
  segue direto, protegida por chave.
- Rollout em dois PRs: primeiro o proxy sobe **sem receber tráfego** (validado por `port-forward`), depois
  um PR troca a rota. Reverter o PR da rota desfaz o cutover.

## Registros de aplicação no Entra

| Registro | Usado por | Redirect URIs | Segredo |
|---|---|---|---|
| `oauth2-proxy homelab` (`077cba4b-…`) | oauth2-proxy do n8n e do LiteLLM; SSO nativo do LiteLLM | `https://n8n.cmoreira.dev/oauth2/callback`, `https://llm.cmoreira.dev/oauth2/callback`, `https://llm.cmoreira.dev/sso/callback` | SSM `/homelab/oauth2-proxy/{client-id,client-secret,cookie-secret}` |
| `Argocd` (`ef1ad5d7-…`) | Dex do ArgoCD | `https://argocd.cmoreira.dev/api/dex/callback` | SSM `/homelab/argocd/dex-microsoft-client-secret` |

Os client secrets expiram em **2027-10-03**. Todos chegam ao cluster por `ExternalSecret`; nenhum fica em
ConfigMap ou no git.

### Rotacionar um client secret

1. No Entra, crie uma nova credencial no registro (sem apagar a antiga).
2. Atualize o parâmetro no SSM. O ESO sincroniza em até 1 hora (24 horas para os do oauth2-proxy);
   reinicie o consumidor para pegar na hora.
3. Teste o login e só então apague a credencial antiga no Entra.

## ArgoCD: quem entra

- O conector Microsoft do Dex só aceita membros do grupo Entra **`Platform Engineering`** (o nome precisa
  bater exatamente, inclusive o espaço). Dar acesso a alguém é, portanto, **adicionar a pessoa ao
  grupo no Entra** (passo manual, fora do GitOps).
- O papel vem do `argocd-rbac-cm`: admin por e-mail para `cassio@cmoreira.dev` e `cassio@rapporthub.pt`;
  qualquer outro membro do grupo cai em `role:readonly` (`policy.default`).
- O `argocd-cm` guarda só a referência `$argocd-dex-microsoft:clientSecret`; o Secret vem do SSM por
  `ExternalSecret` (rótulo `app.kubernetes.io/part-of: argocd`, obrigatório para o ArgoCD resolver a
  referência).
- O ArgoCD é instalado por `helm upgrade` fora do GitOps; os valores moram em `values.yaml.j2` no repo de
  bootstrap. A conta local `admin` continua habilitada como último recurso.
- Chart 10.x: `global.networkPolicy.create=false`, porque o flannel não aplica NetworkPolicy.

## Armadilhas já vividas

- **Grupo com nome diferente** (`PlatformEngineering` no Dex, `Platform Engineering` no Entra): todos
  recebiam "not in any of the required groups".
- **Cookie de sessão acima de 4 KB**: 502 no callback do oauth2-proxy (ver acima).
- **Login duplo no LiteLLM**: com o proxy na frente da UI, o LiteLLM ainda pedia o formulário de
  usuário/senha. O SSO nativo resolveu.
- **Sync lento do LiteLLM**: o chart roda um Job de migração do banco como hook em todo sync, então
  qualquer mudança no app leva alguns minutos.
