# Sara: UI (`ui.ia.local-sara`)

!!! info "Source of truth: the repo code; update here on any functional change"
    Source: `sara/ui.ia.local-sara` (`src/`, `Dockerfile`, `next.config.js`) and
    `sara/gitops.local-sara`. Overview in [Sara](sara.md); API contract in
    [Sara: API](sara-api.md); open items in [Sara: backlog](sara-backlog.md).

Sara's web player: search a song, see chords and lyrics auto-scroll at the BPM you choose,
build a playlist and play it through without touching the mouse to scroll. A clean
alternative to Cifra Club for what it does badly: following along while playing (ads, a
memory leak that degrades the page over a session, no hands-free scroll).

No business logic, no scraping and no direct calls to Cifra Club: all of that lives in
[`api.ia.local-sara`](sara-api.md). The UI only renders what the API returns.

The interface is **English-only, deliberately**: Sara does not speak Portuguese, even though
the content comes from a Brazilian site. There is no i18n layer (unlike `ui.ia.teupadel.com`
with `next-intl`); adding one would be scope creep for a two-person gift app.

## Architecture

```
Browser
  -> Next.js (App Router, client components, standalone server)
      -> NEXT_PUBLIC_API_URL (api.ia.local-sara, FastAPI)   [called from the browser]
          -> cifraclub.com.br (scraped server-side, never from the browser)
```

The UI calls the API **directly from the browser** (CORS enabled on the API), without a Route
Handler proxy. In teupadel the proxy exists to hide a secret (`ANTHROPIC_API_KEY`); Sara's API
has no secrets, so the extra hop would buy nothing. If that changes, mirror teupadel's Route
Handler instead of `rewrites()` (which silently breaks under `output: 'standalone'`).

!!! warning "Correction to old doc"
    The repo's `docs/index.md` says the UI reaches the API "over the in-cluster Service". False:
    since the browser makes the call, it uses the public URL (`NEXT_PUBLIC_API_URL`, default
    `https://local.cmoreira.dev/sara/api`). There is no in-cluster UI-to-API call.

## Repo structure

```
ui.ia.local-sara/
├── src/
│   ├── app/
│   │   ├── layout.jsx        # fonts (next/font/google: Inter, JetBrains Mono), PlaylistProvider, <html lang="en">
│   │   ├── page.jsx          # home: "Play with Sara" hero + search + results
│   │   ├── globals.css       # design tokens as CSS vars
│   │   ├── icon.svg          # favicon (App Router convention)
│   │   ├── health/route.js   # GET /health -> 200 "ok" (under the basePath: /sara/health)
│   │   └── player/[artist]/[song]/page.jsx   # practice screen
│   ├── components/           # Header, Hero, SearchBar, SongResultsList, PlaylistDrawer,
│   │                         # ChordLyricViewer, PlayerControls, ErrorState (+ .module.css)
│   ├── context/PlaylistContext.jsx   # localStorage playlist
│   ├── hooks/useAutoScroll.js        # BPM-driven auto-scroll engine
│   └── api/client.js                 # fetch wrapper over the API
├── next.config.js            # output: 'standalone', basePath
├── Dockerfile, package.json, jsconfig.json
├── public/                   # only .gitkeep
└── catalog-info.yaml, docs/index.md, mkdocs.yml   # Backstage / TechDocs
```

## UX flow

1. **Home**: a single search bar. If the text contains `cifraclub.com.br` (regex), it calls
   `POST /songs/from-url` and goes straight to the player; otherwise it calls
   `GET /search?q=` and lists the artist's songs (`SongResultsList`), with a "Best match"
   badge on `matched_song_slug`, and "+ Playlist" and "Play" buttons. API errors show in
   `ErrorState` (the response `detail`). The placeholder suggests "artist + song" or pasting a
   link.
2. **Player** (`/player/<artist>/<song>`): title, artist, `Key`/`Capo` tags (when present),
   "+ Playlist", a "Source" link to Cifra Club. Monospace viewer with highlighted chords,
   sections without brackets. Controls: Prev / Play-Pause / Next, BPM slider (40 to 220, step
   2, default 90) and a Full/Simplified toggle (only if `has_simplified`). Switching
   Full/Simplified refetches and resets the scroll to the top.
3. **Playlist** (`PlaylistDrawer`, button in `Header` with a counter): Play per item, Remove,
   "Play all" (index 0) and Clear. No reordering (MVP).

## Auto-scroll (`useAutoScroll`)

It exists because of the original complaint (ads, memory leak, no hands-free scroll). Design
constraints:

- **One `requestAnimationFrame` loop, always cancelled**: in the effect cleanup on every
  dependency change and on unmount, no dangling `setInterval`/rAF. Keep this property when
  touching the hook: it is the point of the app.
- Speed derived from BPM, not a fixed px/frame: `pxPerSecond = 16 * (bpm / 60)`. The `16`
  (`PX_PER_BEAT`) is a tuned constant, not derived from time signature: Cifra Club has no
  per-line timing, so this is "faster BPM scrolls faster", not a metronome. Real beat sync
  would need timing data the site does not expose.
- Position is accumulated in JS (`positionRef`) and only the result is written to
  `scrollTop`: the DOM `scrollTop` is an integer and the per-frame delta is a fraction of a
  pixel (~0.4 px at 90 BPM and 60 fps), which would round to zero every frame.
- `bpm` and `onReachEnd` are read through refs inside the loop, so changing BPM mid-scroll
  does not tear down and restart the loop (avoids jitter); only `playing` restarts it.
- Reaching the bottom of the container triggers `onReachEnd`; the player advances to the
  next playlist song with autoplay (the "one click through a set" requirement). With no next
  song, it pauses.

## Playlist (client only)

`PlaylistContext` persists to `localStorage` under the key `sara.playlist`, with no backend or
auth (two-person tool; persistence infra would be overhead). Items:
`{artistSlug, songSlug, artist, title}`, no duplicates (key `artistSlug/songSlug`). Syncing
across devices would need a real store: only at that point.

The player reads `?playlistIndex=N` to know it is playing song N of the playlist (Prev/Next
and auto-advance). A song opened directly (from search) has no `playlistIndex`: it plays
standalone and Prev/Next are disabled. `?autoplay=1` makes auto-advance start already
scrolling; without it, opening a song always starts paused.

## Design system

Visual language adapted from `sara/DESIGN-together.ai.md` (dark hero, white body, mono-caps
eyebrow, single black CTA pill). Tokens become custom properties in `src/app/globals.css`
(`--color-*`, `--space-*`, `--radius-*`). Font substitutes per that doc's own guidance:
`Inter` (sans) and `JetBrains Mono` (eyebrow/label), via `next/font/google` in `layout.jsx`
(self-hosted at build, no external CDN, no CLS).

- CSS Modules per component; no Tailwind and no component library (same convention as
  `ui.ia.teupadel.com`).
- Chords in `var(--color-accent-magenta)`, weight 500: the only use of the accent as a solid
  fill, a deliberate scoped exception; do not spread accents (the design doc's "don't
  introduce a fifth accent" rule).
- `DESIGN-together.ai.md` lives in `sara/` (product root), not inside the UI repo.

## Environment variables (build-time)

| Variable | Required | Default (Dockerfile `ARG`) | Description |
|---|---|---|---|
| `NEXT_PUBLIC_API_URL` | no | `https://local.cmoreira.dev/sara/api` | API base URL, called directly from the browser (`client.js` falls back to `http://localhost:8080` if absent) |
| `NEXT_BASE_PATH` | no | `/sara` | Becomes `basePath` in `next.config.js` |

**Both are build-time, not runtime.** `NEXT_PUBLIC_*` is inlined into the bundle at
`npm run build` and `basePath` is a build option: an `environment:` entry at `docker run`/
compose time does not change them. The Dockerfile takes them as `ARG`s.

**The `ARG` defaults are the production values**, because the reusable CI workflow
(`cmoreira-dev/.github` -> `build-push-ecr.yml`) does a plain `docker build .` with **no
`--build-arg`**. `docker-compose.yml` overrides both for local
(`NEXT_PUBLIC_API_URL=http://localhost:8080`, empty `NEXT_BASE_PATH`). A new build-time
variable needs the same treatment on both sides, otherwise local and production silently
diverge.

## Conventions and stack

- Next.js `^16.0.0`, React `^19.0.0` (`package.json` on `origin/main`), Node 24
  (`node:24-slim` in the Dockerfile). The local checkout used to write this page was still on
  Next `^15`/React `^18`/Node 20 (behind `origin/main`). Function components + hooks; no
  Redux/Zustand.
- Plain JavaScript (no TypeScript), native `fetch` (no axios), no `next-intl`.
- Alias `@/*` -> `./src/*` (`jsconfig.json`).
- No `package-lock.json` in the repo: the Dockerfile uses `npm install --prefer-offline`.
- The Renovate bumps to Next 16, React 19 and Node 24 are already merged on `origin/main`
  (validation of build/runtime under those versions is in the [backlog](sara-backlog.md)).

## Test locally

```bash
npm install
NEXT_PUBLIC_API_URL=http://localhost:8080 npm run dev
# http://localhost:3000
```

Production build: `npm run build` and `NEXT_PUBLIC_API_URL=http://localhost:8080 npm start`.
Or via Docker with `sara/docker-compose.yml` (together with the API). No automated tests;
the repos have no PR CI, only build-on-push to `main`: validate `docker build` locally before
merging.

## Image (Dockerfile)

Two `node:24-slim` stages (`origin/main`; `node:20-slim` in the outdated local checkout).
The builder takes the `ARG`s, runs `npm install` and `npm run build`. Runner:
`NODE_ENV=production`, `PORT=3000`, `HOSTNAME=0.0.0.0`, copies `.next/standalone`,
`.next/static` and `public` with `--chown=1000:1000`, `USER 1000` (uid of the `node` user,
numeric so `runAsNonRoot` can verify it), `EXPOSE 3000`, `CMD ["node", "server.js"]`.

!!! warning "Non-root and port 3000 are mandatory"
    `generic-app` 0.7.0 applies a `restricted` `securityContext` (non-root, drop ALL
    capabilities, no `NET_BIND_SERVICE`): the image must run non-root and on a port above
    1024, with `service.targetPort` (3000) equal in `helm/ui/values.yaml`. The old remote
    Renovate branches (`renovate/major-nextjs-monorepo`, `major-react-monorepo`,
    `node-24.x`), already merged, still exist on the remote with a base predating the fix
    (`PORT=80`, no `USER`): do not reuse them; new bumps must preserve the non-root
    Dockerfile.

## Deploy (`local.cmoreira.dev/sara`)

`local.cmoreira.dev` is a shared hostname (Gateway `nginx-gateway-cmoreira-dev` in
`gitops.core-addons`, already tunneled via Cloudflare by host, not by path: a new path needs
no change there). The UI `HTTPRoute` matches `PathPrefix /sara` and forwards **without**
stripping the prefix, which is exactly why `basePath` exists: Next.js must know it is
mounted at `/sara` to prefix links, assets and RSC fetches, and expects the path with
`/sara` intact (a `URLRewrite` would break that). The API is at `/sara/api` and **does**
strip the prefix. Gateway API resolves by longest-prefix-match, so `/sara/api/*` beats the
UI's `/sara` catch-all.

| Item | Value (`helm/ui/values.yaml`) |
|---|---|
| Image | ECR `<account>.dkr.ecr.us-east-1.amazonaws.com/sara/ui` (workflow `build-push.yml` -> `build-push-ecr.yml`) |
| Tag | Short SHA, updated by `build: automatic update of sara-ui` commits (argocd-image-updater) |
| Service | port 80 to `targetPort` 3000 |
| Route | Gateway `nginx-gateway-cmoreira-dev` (ns `nginx-gateway`), host `local.cmoreira.dev`, `PathPrefix /sara`, no filter |
| Config/secrets | no `configMap` or `ExternalSecret` (config is build-time) |

Namespace `sara`, Application `sara-ui` (project `homelab`, automated sync). Gitops and
`gitops.generic-app-chart` wrapper details in [Sara: API](sara-api.md#deploy).

!!! warning "To confirm"
    - `docker-compose.yml` maps the UI as `3000:80`, but the container now listens on 3000
      (`PORT=3000`): likely a stale mapping (see [backlog](sara-backlog.md)).
    - UI liveness/readiness probes: `values.yaml` defines none; confirm the chart default.
      `/sara/health` exists in the app.
    - Next 16 / React 19 / Node 24 (merged today, 2026-09-29): I did not validate that the
      standalone build and the non-root runtime still work on those versions (the local
      checkout was behind; I only read the diff: only `Dockerfile` and `package.json`
      changed).
