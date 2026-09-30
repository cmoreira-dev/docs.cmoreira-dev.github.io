# teupadel UI (`ui.ia.teupadel.com`)

!!! info "Fonte de verdade / atualizar aqui"
    Esta página é a fonte de verdade sobre rotas, componentes, contrato da API consumida,
    tema/marca e deploy da UI. O `CLAUDE.md` do repo guarda só o que tem de estar sempre
    carregado (comandos, proibições, acoplamentos) e aponta para aqui. Atualizar (PT + EN)
    em qualquer mudança funcional. Visão geral do produto: [teupadel.com](teupadel.md).

Frontend público do teupadel.com: site institucional + a ferramenta de análise. Next.js 16
(App Router, SSR) com React 19, `next-intl` (en / pt-pt / pt-br) e Node 24, servido por
`next start` standalone. Não contém lógica de negócio nem chamadas à Anthropic: tudo passa
por `api.ia.teupadel.com`, alcançada só server-side. O nome escreve-se sempre **teupadel**,
em minúsculas.

## Arquitetura

```
Browser --HTTPS--> Cloudflare Tunnel --> NGINX Gateway Fabric --> teupadel-ui (Next.js, :3000)
                                                                     |  Route Handlers (proxy runtime)
                                                                     v
                                                     teupadel-api (FastAPI, ClusterIP, ns teupadel)
```

O browser nunca fala com a API: chama `/analyse`, `/api/*`, `/telemetry` e `/waitlist` na
própria UI, e os Route Handlers reencaminham para `${PADEL_API_URL}` (Service interno
`http://teupadel-api.teupadel.svc.cluster.local`), passando o cookie de sessão quando
aplicável.

## Rotas

Todas as páginas existem nos 3 idiomas com prefixo (`/en`, `/pt-pt`, `/pt-br`,
`localePrefix: 'always'`). **Os slugs ficam sempre em inglês**, mesmo com conteúdo traduzido.

| Rota | Componente | O que é |
|---|---|---|
| `/` | `Home.jsx` | Homepage institucional (hero, como funciona, exemplo, golpes, guia de gravação, confiança, FAQ, CTA) |
| `/analysis` | `App.jsx` + `UploadZone`/`LoadingState`/`ReportView`/`NoAnalysis`/`ErrorState` | A ferramenta: upload, loading, relatório / não deu para analisar / erro |
| `/sample-report` | `SampleReport.jsx` (usa `ReportView`) | Relatório de exemplo fixo, sem vídeo real |
| `/pricing` | `Pricing.jsx` + `WaitlistDialog.jsx` | Planos de bolas (preços por região em `src/data/pricingPlans.js`), CTAs de waitlist |
| `/pricing/no-bolas` | `NoBolas.jsx` | Destino de quem fica sem saldo |
| `/how-it-works`, `/how-to-record`, `/faq` | `HowItWorks`, `HowToRecord`, `Faq` | Páginas de conteúdo |
| `/strokes/[stroke]` | `StrokePage.jsx` + `src/data/strokes.jsx` | 6 páginas: `serve`, `forehand`, `backhand`, `volley`, `smash`, `ready-position` |
| `/privacy`, `/cookies`, `/terms` | `Privacy`, `Cookies`, `Terms` (sobre `LegalLayout`) | Legal; `Terms` é rascunho sem revisão jurídica (`Terms.draftNote`) |
| `/login` | `Login.jsx` + `Turnstile.jsx` | Login unificado (magic link + Google); `?ref=<código>` (indicação) e `?redirect=` |
| `/auth/verify` | `AuthVerify.jsx` | Consome o token do magic link (`/api/auth/verify`) |
| `/account`, `/account/delete` | `Account.jsx`, `ProgressChart.jsx`, `AccountDelete.jsx` | Conta, saldo de bolas, gráfico de evolução (beta), histórico, exclusão (RGPD) |
| `/reports/[id]` | `SavedReport.jsx` (usa `ReportView`) | Relatório do histórico, sem player de vídeo |

`/` redireciona (middleware `next-intl`) para o locale de `Accept-Language`, com fallback
`pt-pt`. Fora do `[locale]` ficam os Route Handlers e `/health`, `/robots.txt`, `/sitemap.xml`.
`src/data/strokes.jsx` define os slugs válidos e mapeia para `Strokes.<slug>.*` em
`messages/<locale>.json`; só `smash` tem conteúdo técnico revisado, os outros levam
`placeholderNote` (pendente de validação com treinador). `HowToRecord.placementPlaceholder`
e `Terms.draftNote` são os outros conteúdos pendentes sinalizados na página.

### Route Handlers (proxies)

| Handler | Encaminha para | Notas |
|---|---|---|
| `POST /analyse` | `/analyse` | Stream do multipart; timeout 300 s; reencaminha `baggage`/`traceparent`/`cookie` |
| `POST /telemetry` | `/telemetry/events` | `sendBeacon`; qualquer falha devolve 204 |
| `POST /waitlist` | `/waitlist` | Timeout 10 s |
| `POST /api/auth/magic-link`, `GET /api/auth/verify`, `POST /api/auth/logout` | `/auth/*` | Copiam `Set-Cookie` de volta |
| `GET /api/auth/google`, `GET /api/auth/callback/google` | `/auth/google`, `/auth/callback/google` | Repassam o redirect 3xx e o cookie de state do OAuth |
| `GET`/`DELETE /api/me` | `/me` | Sessão, saldo, `referral_code`; `DELETE` apaga a conta |
| `GET /api/reports`, `GET`/`DELETE /api/reports/[id]` | `/reports*` | Histórico |
| `GET /api/me/progress` | `/me/progress` | Séries de notas por golpe e geral (gráfico de "Minha conta") |
| `GET /health` | (local) | Probe do Kubernetes |

Os helpers comuns (`apiBase`, `forwardCookieHeader`, `copySetCookie`, `forwardUserAgent`) estão
em `src/lib/authProxy.js`. **Todo proxy novo lê `PADEL_API_URL` a cada pedido** (ver "Variáveis").

## API consumida

O fluxo de análise é **assíncrono** (`src/api/client.js`, `src/hooks/useAnalyse.js`):

1. `POST /analyse?fps=&lang=&movement=` com `FormData` (`file`: mp4/mov/avi/mkv, máx. 100 MB).
   `fps` é 1-10 e calculado no cliente; `lang` é o locale (`en`|`pt-pt`|`pt-br`); `movement` é opcional
   (`serve`/`forehand`/`backhand`/`volley`/`smash`; vazio = deteção automática).
2. A API responde **`202 {report_id, ...}`**. Erros: `401 login_required` (visitante anónimo),
   `402` (sem bolas), `429` (limite de jobs em curso), `400` (formato/idioma/movement), `5xx`.
3. A UI faz polling de `GET /api/reports/{id}` a cada 3 s, até 6 min. Estados: `done` e `no_analysis`
   devolvem `result`; `failed` mostra a mensagem genérica; `processing` continua. Passados 6 min, sugere
   consultar o histórico (o relatório continua a ser gerado no servidor).
4. Redirecionamentos: `401` leva a `/<lang>/login?redirect=/analysis`; `402` leva a `/<lang>/pricing`.

O `result` tem a forma que `ReportView` consome: `metadata` (`total_frames`, `frames_with_pose`,
`fps_extracted`, `processing_time_s`), `report` (`resumo_geral`, `pontos_positivos[]`,
`pontos_a_melhorar[]` com `titulo`/`texto`/`golpe_index`/`tempo_s`, e `proximo_treino` com `passos` e
`momentos`), e `pose_frames` (landmarks por frame, para o esqueleto). Quando `analysis_possible` é
`false` não há `report`: a UI mostra `NoAnalysis.jsx` com dicas fixas traduzidas a partir de `reason`
(`no_stroke_detected` | `low_pose_detection`) e `metadata`. O contrato completo (estados, estornos,
endpoints) está em [teupadel.com](teupadel.md#analise-assincrona-e-historico-de-relatorios) e no
`CLAUDE.md` da API.

Outras chamadas: `GET /api/me` (cabeçalho, saldo), `GET /api/reports` (histórico), `DELETE /api/me`,
`POST /waitlist`, `POST /telemetry` (eventos). A API normaliza o JSON do LLM antes de o servir; a UI não
deve confiar em campos opcionais sem defaults.

## Fluxo de UX em `/analysis`

1. **Upload** (`UploadZone`): drag & drop ou clique; validação de extensão e tamanho; leitura da duração
   do vídeo no cliente; aviso se < 8 s (não bloqueia); chips opcionais de golpe. O `fps` é automático:
   `clamp(round(50 / duração), 2, 10)` (sem slider). O upload é livre; a análise exige conta.
2. **Loading** (`LoadingState`): mensagens progressivas, botão cancelar (aborta o polling).
3. **Relatório** (`ReportView`): vídeo local do utilizador (`URL.createObjectURL`, nunca reenviado) com
   **overlay do esqueleto ao vivo** (`PoseCanvasOverlay`, a partir de `pose_frames`) e miniaturas dos
   momentos (`skeleton/MomentThumbnails`); resumo, pontos fortes e a melhorar (cada um com "Ver no vídeo ·
   golpe N, mm:ss", que faz seek), próximo treino, exportar PDF (texto). Em `/reports/[id]` o vídeo não
   existe, logo não há player nem seek.
4. **Não deu para analisar** (`NoAnalysis`) e **erro** (`ErrorState`).

Depois de cada análise a UI faz `router.refresh()` para o saldo no cabeçalho refletir débito ou estorno.
`PoseIllustration.jsx` e `AnalyzingIllustration.jsx` são ilustrações estáticas, não dados reais.

## Estrutura do repo

```
ui.ia.teupadel.com/
├── public/brand/           logo, símbolo, selo, raquete (3 variantes), favicon-180
├── messages/               en.json, pt-pt.json, pt-br.json (namespaces por componente)
├── src/
│   ├── middleware.js       next-intl (matcher exclui analyse/telemetry/waitlist/health/api)
│   ├── i18n/               routing.js, navigation.js, request.js
│   ├── fonts/              woff2 self-hosted (+ README com licenças)
│   ├── data/               strokes.jsx, pricingPlans.js
│   ├── app/
│   │   ├── [locale]/       páginas (tabela de rotas), layout.jsx, not-found, opengraph-image
│   │   ├── analyse/ telemetry/ waitlist/ health/   Route Handlers
│   │   ├── api/            auth/*, me, reports, reports/[id]
│   │   └── robots.js, sitemap.js, icon.svg
│   ├── App.jsx             estado da ferramenta
│   ├── api/client.js       upload + polling
│   ├── hooks/useAnalyse.js
│   ├── lib/                telemetry, consent, authProxy, jsonLd, region, videoDuration, ...
│   └── components/         páginas, Turnstile, skeleton/ (overlay), Header/Footer/AccountMenu, ...
├── next.config.js          standalone, CSP, headers, images.unoptimized, poweredByHeader off
├── Dockerfile
└── package.json
```

## Variáveis de ambiente

| Variável | Onde | Descrição |
|---|---|---|
| `PADEL_API_URL` | ConfigMap (`gitops.teupadel.com/helm/ui/values.yaml`) | URL interna da API; default `http://localhost:8080`. Lida server-side a cada pedido; sem prefixo `NEXT_PUBLIC_` |
| `TURNSTILE_SITE_KEY` | ExternalSecret (SSM `/teupadel/turnstile/site-key`) | Chave pública do Turnstile, lida em runtime em `login/page.jsx`; sem ela (dev) o widget não renderiza |

!!! warning "Não usar `rewrites()` para `PADEL_API_URL`"
    Sob `output: 'standalone'` o destino de `rewrites()` é resolvido e congelado no build. Confirmado
    em 2026-08-13: build com `PADEL_API_URL=A`, executado com `B`, continuava a usar `A`. Os Route
    Handlers leem `process.env` em runtime, por isso a variável muda com o ConfigMap sem rebuild.

## Tema e marca

Tema único claro, tokens em `src/index.css` (nomes em português), classes partilhadas `tp-btn`, `tp-tag`,
`tp-card`, `tp-rotulo`. Faixas escuras fixas via `--noite`/`--on-noite`. Não há `data-theme` nem toggle
(`color-scheme: light`). Fontes self-hosted com `next/font/local` a partir de `src/fonts/*.woff2`
(Unbounded 500-800, Figtree 400/500/700, JetBrains Mono 500/600; licença OFL), para o build não depender
do Google Fonts. Detalhe de paleta, assets e diretrizes em [Marca](../products/teupadel-brand.md).

## Build, imagem e deploy

- **Stack**: Next.js 16, React 19, `next-intl` 4, Node 24 (a fonte é o `package.json` e o `Dockerfile`).
- **Dockerfile**: dois estágios `node:24-slim`; o runner copia `.next/standalone`, `.next/static` e
  `public`, corre `node server.js` como **uid 1000** na **porta 3000**, com `HOSTNAME=0.0.0.0` (o K8s
  injeta `HOSTNAME=<pod>`; sem o override o servidor só ouve no IP do pod). Sem Nginx.
- **Validação**: `docker build` (o CI só faz build no push para `main`, não há CI de PR). Localmente também
  funcionam `npm run dev` e `npm run build`.
- **Registry**: ECR `teupadel/ui`, publicado pelo pipeline de [Build & Registry](../cicd/build-registry.md).
- **GitOps**: `gitops.teupadel.com/helm/ui` (wrapper do `generic-app`), namespace `teupadel`, HTTPRoute
  `www.teupadel.com` na Gateway `nginx-gateway-teupadel-com`. Argo CD com auto-sync; o argocd-image-updater
  atualiza a tag. Probes em `/health`; `securityContext` restricted (non-root, drop ALL, seccomp).
- **Renderização**: depois do Next 16 todas as páginas aparecem como dinâmicas no output do build; não foi
  investigado (ver [backlog](../products/teupadel-backlog.md)).
- **Middleware**: `src/middleware.js` continua em uso; o Next 16 avisa que passou a `proxy` (pendente).

## Segurança

Revisão de 2026-09-29:

- **CSP** (`next.config.js`): `default-src 'self'`; scripts, frame e `connect-src` só acrescentam
  `challenges.cloudflare.com` (Turnstile); `object-src 'none'`, `frame-ancestors 'none'`,
  `upgrade-insecure-requests` em produção. `script-src` mantém `'unsafe-inline'` porque as páginas são
  pré-renderizadas e não há nonce por pedido; `'unsafe-eval'` só em dev.
- Outros headers: `X-Content-Type-Options`, `X-Frame-Options: DENY`, `Referrer-Policy`,
  `Permissions-Policy` (camera/microfone/geolocalização desligados), HSTS em produção; `X-Powered-By` removido.
- **`/_next/image` desligado** (`images.unoptimized`); JSON-LD escapado.
- **Cloudflare**: "Always Use HTTPS" e HSTS ligados.
- **NetworkPolicies** mergeadas em `gitops.teupadel.com` (#37) mas **sem efeito**: o CNI é flannel puro,
  que não as aplica. Precisam de Cilium ou Calico.
- **Smoke test**: `security-smoke.sh` (raiz de `teupadel.com/`) faz pedidos não autenticados e espera
  rejeição; corrido em produção em 2026-09-29, tudo PASS. Repetir depois de cada deploy relevante.
- Pendente: confirmar no browser que o Turnstile de `/login` carrega sem erros de CSP.
  Ver [backlog](../products/teupadel-backlog.md).


## Notas e gráfico de evolução (beta)

- `ProgressChart.jsx` (em `/account`) desenha, em SVG próprio, uma série de cada vez: **Geral** ou um golpe
  (abas). A lógica pura está em `src/lib/progress.js`.
- A linha **não atravessa** uma mudança de `reference_version`/`analysis_version`: há uma marca tracejada e a
  nota "refinámos o modelo". Com menos de 2 análises do golpe mostra só a nota.
- `ReportView` mostra `result.scores.score` com o selo **beta** no resumo do relatório.
- Textos no namespace `Progress` dos 3 idiomas. Contrato: [API, Notas](teupadel-api.md#notas-beta).
