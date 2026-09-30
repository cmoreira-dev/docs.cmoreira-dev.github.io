# teupadel UI (`ui.ia.teupadel.com`)

!!! info "Source of truth / update here"
    This page is the source of truth for routes, components, the consumed API contract,
    theme/brand and deployment of the UI. The repo `CLAUDE.md` only holds what must always be
    loaded (commands, prohibitions, couplings) and points here. Update (PT + EN) on any
    functional change. Product overview: [teupadel.com](teupadel.md).

Public frontend of teupadel.com: marketing site plus the analysis tool. Next.js 16
(App Router, SSR) with React 19, `next-intl` (en / pt-pt / pt-br) and Node 24, served by
standalone `next start`. It has no business logic and never calls Anthropic: everything goes
through `api.ia.teupadel.com`, reached server-side only. The name is always spelled
**teupadel**, lowercase.

## Architecture

```
Browser --HTTPS--> Cloudflare Tunnel --> NGINX Gateway Fabric --> teupadel-ui (Next.js, :3000)
                                                                     |  Route Handlers (runtime proxy)
                                                                     v
                                                     teupadel-api (FastAPI, ClusterIP, ns teupadel)
```

The browser never talks to the API: it calls `/analyse`, `/api/*`, `/telemetry` and `/waitlist`
on the UI itself, and the Route Handlers forward to `${PADEL_API_URL}` (internal Service
`http://teupadel-api.teupadel.svc.cluster.local`), passing the session cookie where relevant.

## Routes

All pages exist in 3 languages with a prefix (`/en`, `/pt-pt`, `/pt-br`,
`localePrefix: 'always'`). **Slugs always stay in English**, even with translated content.

| Route | Component | What it is |
|---|---|---|
| `/` | `Home.jsx` | Marketing homepage (hero, how it works, sample, strokes, recording guide, trust, FAQ, CTA) |
| `/analysis` | `App.jsx` + `UploadZone`/`LoadingState`/`ReportView`/`NoAnalysis`/`ErrorState` | The tool: upload, loading, report / could not analyse / error |
| `/sample-report` | `SampleReport.jsx` (uses `ReportView`) | Fixed sample report, no real video |
| `/pricing` | `Pricing.jsx` + `WaitlistDialog.jsx` | Bola plans (regional prices in `src/data/pricingPlans.js`), waitlist CTAs |
| `/pricing/no-bolas` | `NoBolas.jsx` | Landing for users out of balance |
| `/how-it-works`, `/how-to-record`, `/faq` | `HowItWorks`, `HowToRecord`, `Faq` | Content pages |
| `/strokes/[stroke]` | `StrokePage.jsx` + `src/data/strokes.jsx` | 6 pages: `serve`, `forehand`, `backhand`, `volley`, `smash`, `ready-position` |
| `/privacy`, `/cookies`, `/terms` | `Privacy`, `Cookies`, `Terms` (on `LegalLayout`) | Legal; `Terms` is a draft without legal review (`Terms.draftNote`) |
| `/login` | `Login.jsx` + `Turnstile.jsx` | Unified login (magic link + Google); `?ref=<code>` (referral) and `?redirect=` |
| `/auth/verify` | `AuthVerify.jsx` | Consumes the magic-link token (`/api/auth/verify`) |
| `/account`, `/account/delete` | `Account.jsx`, `ProgressChart.jsx`, `AccountDelete.jsx` | Account, bola balance, progress chart (beta), history, deletion (GDPR) |
| `/reports/[id]` | `SavedReport.jsx` (uses `ReportView`) | Report from history, without a video player |

`/` redirects (`next-intl` middleware) to the locale from `Accept-Language`, falling back to
`pt-pt`. Outside `[locale]` live the Route Handlers and `/health`, `/robots.txt`, `/sitemap.xml`.
`src/data/strokes.jsx` defines the valid slugs and maps them to `Strokes.<slug>.*` in
`messages/<locale>.json`; only `smash` has reviewed technical content, the others carry a
`placeholderNote` (pending validation with a coach). `HowToRecord.placementPlaceholder` and
`Terms.draftNote` are the other pending content flagged on the page.

### Route Handlers (proxies)

| Handler | Forwards to | Notes |
|---|---|---|
| `POST /analyse` | `/analyse` | Streams the multipart; 300 s timeout; forwards `baggage`/`traceparent`/`cookie` |
| `POST /telemetry` | `/telemetry/events` | `sendBeacon`; any failure returns 204 |
| `POST /waitlist` | `/waitlist` | 10 s timeout |
| `POST /api/auth/magic-link`, `GET /api/auth/verify`, `POST /api/auth/logout` | `/auth/*` | Copy `Set-Cookie` back |
| `GET /api/auth/google`, `GET /api/auth/callback/google` | `/auth/google`, `/auth/callback/google` | Pass on the 3xx redirect and the OAuth state cookie |
| `GET`/`DELETE /api/me` | `/me` | Session, balance, `referral_code`; `DELETE` removes the account |
| `GET /api/reports`, `GET`/`DELETE /api/reports/[id]` | `/reports*` | History |
| `GET /api/me/progress` | `/me/progress` | Score series per movement and overall ("My account" chart) |
| `GET /health` | (local) | Kubernetes probe |

Shared helpers (`apiBase`, `forwardCookieHeader`, `copySetCookie`, `forwardUserAgent`) are in
`src/lib/authProxy.js`. **Every new proxy reads `PADEL_API_URL` on each request** (see "Variables").

## Consumed API

The analysis flow is **asynchronous** (`src/api/client.js`, `src/hooks/useAnalyse.js`):

1. `POST /analyse?fps=&lang=&movement=` with `FormData` (`file`: mp4/mov/avi/mkv, max 100 MB).
   `fps` is 1-10 and computed on the client; `lang` is the locale (`en`|`pt-pt`|`pt-br`); `movement` is
   optional (`serve`/`forehand`/`backhand`/`volley`/`smash`; empty = auto-detect).
2. The API answers **`202 {report_id, ...}`**. Errors: `401 login_required` (anonymous visitor),
   `402` (no bolas), `429` (in-flight job limit), `400` (format/language/movement), `5xx`.
3. The UI polls `GET /api/reports/{id}` every 3 s, up to 6 min. States: `done` and `no_analysis`
   return `result`; `failed` shows the generic message; `processing` keeps polling. After 6 min it
   suggests checking the history (the report keeps being generated server-side).
4. Redirects: `401` goes to `/<lang>/login?redirect=/analysis`; `402` goes to `/<lang>/pricing`.

`result` has the shape `ReportView` consumes: `metadata` (`total_frames`, `frames_with_pose`,
`fps_extracted`, `processing_time_s`), `report` (`resumo_geral`, `pontos_positivos[]`,
`pontos_a_melhorar[]` with `titulo`/`texto`/`golpe_index`/`tempo_s`, and `proximo_treino` with `passos`
and `momentos`), and `pose_frames` (per-frame landmarks, for the skeleton). When `analysis_possible` is
`false` there is no `report`: the UI shows `NoAnalysis.jsx` with fixed translated tips derived from
`reason` (`no_stroke_detected` | `low_pose_detection`) and `metadata`. The full contract (states, refunds,
endpoints) is in [teupadel.com](teupadel.md) and in the API `CLAUDE.md`.

Other calls: `GET /api/me` (header, balance), `GET /api/reports` (history), `DELETE /api/me`,
`POST /waitlist`, `POST /telemetry` (events). The API normalises the LLM JSON before serving it; the UI
must not trust optional fields without defaults.

## UX flow in `/analysis`

1. **Upload** (`UploadZone`): drag & drop or click; extension and size validation; video duration read on
   the client; warning if < 8 s (non-blocking); optional stroke chips. `fps` is automatic:
   `clamp(round(50 / duration), 2, 10)` (no slider). Upload is open; analysing requires an account.
2. **Loading** (`LoadingState`): progressive messages, cancel button (aborts polling).
3. **Report** (`ReportView`): the user's local video (`URL.createObjectURL`, never re-sent) with a
   **live skeleton overlay** (`PoseCanvasOverlay`, from `pose_frames`) and moment thumbnails
   (`skeleton/MomentThumbnails`); summary, strengths and improvements (each with "Watch in video · stroke N,
   mm:ss", which seeks), next training, PDF export (text). On `/reports/[id]` the video does not exist, so
   there is no player or seek.
4. **Could not analyse** (`NoAnalysis`) and **error** (`ErrorState`).

After each analysis the UI calls `router.refresh()` so the header balance reflects the debit or refund.
`PoseIllustration.jsx` and `AnalyzingIllustration.jsx` are static illustrations, not real data.

## Repo structure

```
ui.ia.teupadel.com/
├── public/brand/           logo, symbol, seal, racket (3 variants), favicon-180
├── messages/               en.json, pt-pt.json, pt-br.json (namespaced per component)
├── src/
│   ├── middleware.js       next-intl (matcher excludes analyse/telemetry/waitlist/health/api)
│   ├── i18n/               routing.js, navigation.js, request.js
│   ├── fonts/              self-hosted woff2 (+ README with licences)
│   ├── data/               strokes.jsx, pricingPlans.js
│   ├── app/
│   │   ├── [locale]/       pages (route table), layout.jsx, not-found, opengraph-image
│   │   ├── analyse/ telemetry/ waitlist/ health/   Route Handlers
│   │   ├── api/            auth/*, me, reports, reports/[id]
│   │   └── robots.js, sitemap.js, icon.svg
│   ├── App.jsx             tool state
│   ├── api/client.js       upload + polling
│   ├── hooks/useAnalyse.js
│   ├── lib/                telemetry, consent, authProxy, jsonLd, region, videoDuration, ...
│   └── components/         pages, Turnstile, skeleton/ (overlay), Header/Footer/AccountMenu, ...
├── next.config.js          standalone, CSP, headers, images.unoptimized, poweredByHeader off
├── Dockerfile
└── package.json
```

## Environment variables

| Variable | Where | Description |
|---|---|---|
| `PADEL_API_URL` | ConfigMap (`gitops.teupadel.com/helm/ui/values.yaml`) | Internal API URL; default `http://localhost:8080`. Read server-side on every request; no `NEXT_PUBLIC_` prefix |
| `TURNSTILE_SITE_KEY` | ExternalSecret (SSM `/teupadel/turnstile/site-key`) | Public Turnstile key, read at runtime in `login/page.jsx`; without it (dev) the widget does not render |

!!! warning "Do not use `rewrites()` for `PADEL_API_URL`"
    Under `output: 'standalone'` the `rewrites()` destination is resolved and frozen at build time.
    Confirmed on 2026-08-13: a build with `PADEL_API_URL=A`, run with `B`, kept using `A`. Route Handlers
    read `process.env` at runtime, so the variable changes with the ConfigMap, no rebuild.

## Theme and brand

Single light theme, tokens in `src/index.css` (Portuguese names), shared classes `tp-btn`, `tp-tag`,
`tp-card`, `tp-rotulo`. Fixed dark bands through `--noite`/`--on-noite`. No `data-theme` and no toggle
(`color-scheme: light`). Fonts are self-hosted with `next/font/local` from `src/fonts/*.woff2`
(Unbounded 500-800, Figtree 400/500/700, JetBrains Mono 500/600; OFL licence), so the build does not
depend on Google Fonts. Palette details, assets and guidelines in [Brand](../products/teupadel-brand.md).

## Build, image and deploy

- **Stack**: Next.js 16, React 19, `next-intl` 4, Node 24 (`package.json` and `Dockerfile` are the truth).
- **Dockerfile**: two `node:24-slim` stages; the runner copies `.next/standalone`, `.next/static` and
  `public`, runs `node server.js` as **uid 1000** on **port 3000**, with `HOSTNAME=0.0.0.0` (K8s injects
  `HOSTNAME=<pod>`; without the override the server only listens on the pod IP). No Nginx.
- **Validation**: `docker build` (CI only builds on push to `main`, there is no PR CI). Locally,
  `npm run dev` and `npm run build` also work.
- **Registry**: ECR `teupadel/ui`, published by the [Build & Registry](../cicd/build-registry.md) pipeline.
- **GitOps**: `gitops.teupadel.com/helm/ui` (a `generic-app` wrapper), namespace `teupadel`, HTTPRoute
  `www.teupadel.com` on Gateway `nginx-gateway-teupadel-com`. Argo CD auto-sync; argocd-image-updater
  bumps the tag. Probes on `/health`; restricted `securityContext` (non-root, drop ALL, seccomp).
- **Rendering**: after Next 16 all pages show as dynamic in the build output; not investigated (see the
  [backlog](../products/teupadel-backlog.md)).
- **Middleware**: `src/middleware.js` is still in use; Next 16 warns it was superseded by `proxy` (pending).

## Security

Review of 2026-09-29:

- **CSP** (`next.config.js`): `default-src 'self'`; scripts, frame and `connect-src` only add
  `challenges.cloudflare.com` (Turnstile); `object-src 'none'`, `frame-ancestors 'none'`,
  `upgrade-insecure-requests` in production. `script-src` keeps `'unsafe-inline'` because pages are
  prerendered and there is no per-request nonce; `'unsafe-eval'` only in dev.
- Other headers: `X-Content-Type-Options`, `X-Frame-Options: DENY`, `Referrer-Policy`,
  `Permissions-Policy` (camera/microphone/geolocation off), HSTS in production; `X-Powered-By` removed.
- **`/_next/image` disabled** (`images.unoptimized`); JSON-LD escaped.
- **Cloudflare**: "Always Use HTTPS" and HSTS enabled.
- **NetworkPolicies** merged in `gitops.teupadel.com` (#37) but **not enforced**: the CNI is plain
  flannel, which does not apply them. They need Cilium or Calico.
- **Smoke test**: `security-smoke.sh` (root of `teupadel.com/`) sends unauthenticated requests and
  expects rejection; run against production on 2026-09-29, all PASS. Repeat after each relevant deploy.
- Pending: confirm in the browser that the `/login` Turnstile loads without CSP errors.
  See the [backlog](../products/teupadel-backlog.md).


## Scores and progress chart (beta)

- `ProgressChart.jsx` (in `/account`) draws, in custom SVG, one series at a time: **Overall** or one movement
  (tabs). The pure logic is in `src/lib/progress.js`.
- The line **does not cross** a change of `reference_version`/`analysis_version`: there is a dashed mark and
  the note "we refined the model". With fewer than 2 analyses of a movement it shows only the score.
- `ReportView` shows `result.scores.score` with the **beta** tag in the report summary.
- Texts live in the `Progress` namespace of the 3 locales. Contract: [API, Scores](teupadel-api.md#scores-beta).
