# Sara: API (`api.ia.local-sara`)

!!! info "Source of truth: the repo code; update here on any functional change"
    Source: `sara/api.ia.local-sara` (`main.py`, `scraper.py`, `Dockerfile`) and
    `sara/gitops.local-sara`. Product overview in [Sara](sara.md); UI in
    [Sara: UI](sara-ui.md); open items in [Sara: backlog](sara-backlog.md).

FastAPI backend that resolves artists and songs on [Cifra Club](https://www.cifraclub.com.br)
and returns structured chords/lyrics as JSON. API only: no HTML/CSS/JS, no database, no
secrets.

## Repo structure

```
api.ia.local-sara/
├── main.py            # FastAPI app + routes + CORS
├── scraper.py         # all Cifra Club HTTP and parsing
├── requirements.txt   # fastapi, uvicorn[standard], httpx, beautifulsoup4, lxml (no exact pins)
├── Dockerfile
├── openapi.yaml       # committed snapshot of the spec (Backstage catalog)
├── catalog-info.yaml  # Component + API entity (Backstage)
├── docs/index.md, mkdocs.yml   # repo TechDocs
└── .github/workflows/build-push.yml   # build and push to ECR
```

## Why scraping, and why this approach

Cifra Club has no public search API. Its on-site search (`/?q=...`) is client-side and calls
a private `/api/...` endpoint (Google Custom Search) that is `Disallow`ed in `robots.txt`
and key/referrer-locked: it **must not be called**. What is allowed and is server-rendered
HTML:

- Artist page `GET /<artist-slug>/`: lists the ~15 most popular songs as
  `<a href="/<artist-slug>/<song-slug>/">` links.
- Song page `GET /<artist-slug>/<song-slug>/`: chords and lyrics in
  `<pre data-chord-content="true">`, one `<div>` per block; chords as
  `<b data-chord-name="...">`.

### Artist resolution (`resolve_search`)

1. Slugify the query (lowercase, strip accents, hyphenate) and `GET /<slug>/`: covers
   "Coldplay" and "Legiao Urbana".
2. On 404, if the query has 2+ words, assume artist and song are mixed
   (e.g. "coldplay yellow", "yellow coldplay") and try every word-boundary split, in both
   orders, as an `/<artist>/<song>/` page, capped at `_MAX_SPLIT_ATTEMPTS` (8) requests.
   The first split that resolves wins: the artist's song list is returned with the matched
   song pinned first and `matched_song_slug` set.
3. Nothing resolved: `404`.

Accepted limitation: a single ambiguous word (e.g. "yellow") resolves to the real, different
artist (`/yellow/`), not Coldplay's song, since there is no second word to split against.
A real fix needs a search index/API (Google Custom Search on `site:cifraclub.com.br`, or
crawling the sitemap), judged too much external dependency for a two-person tool. The UI
nudges toward "artist + song" and the paste-a-link escape hatch (`POST /songs/from-url`),
which always works regardless of search.

### Parsing

- Selectors are never CSS classes (hashed, change on every Cifra Club deploy): song title in
  lists via the image `alt` (`Capa da música "..."`) with a partial `primaryLabel` class
  fallback; key via `button[data-anchor="--chord-tone"]`; capo and tuning via label text
  (`Capotraste`, `Afinação`) followed by the next `<p>`; `has_simplified` via a
  `/<artist>/<song>/simplificada.html` link.
- Each `<div>` in the chord block is a *couplet* (chord row + lyric row joined by `\n`).
  `scraper._lines_from_div` flattens it into tokens and splits on `\n`, recomputing
  `chords[].index` per line. Do not assume one div == one line.
- Line classification (`_classify_line`): `blank`, `section` (bracketed), `chord`
  (chord-only after masking the chords), otherwise `lyric`.
- Artist song lists skip the slugs `discografia` and `videoaulas`; only links matching
  `^/([a-z0-9-]+)/([a-z0-9-]+)/$` with the artist's slug are kept.

## Endpoints

Public base: `https://local.cmoreira.dev/sara/api` (the `/sara/api` prefix is stripped before
reaching the container; locally routes are at the root, port 8080).

### `GET /health`

```json
{"status": "ok"}
```

### `GET /search?q=<query>`

Resolution as described above. Same shape as `GET /artists/{artist_slug}` plus
`matched_song_slug` (present only when an artist+song split matched).
Errors: `400` if `q` is empty (after `strip`); `404` nothing resolves (message suggests
pasting a link); `502` Cifra Club unreachable; `422` if `q` is missing (FastAPI validation).

### `GET /artists/{artist_slug}`

```json
{
  "slug": "coldplay",
  "name": "Coldplay",
  "songs": [{"slug": "the-scientist", "title": "The Scientist"}]
}
```

Errors: `404`, `502`.

### `GET /artists/{artist_slug}/songs/{song_slug}?simplified=false`

```json
{
  "artist": "Coldplay",
  "artist_slug": "coldplay",
  "title": "The Scientist",
  "song_slug": "the-scientist",
  "key": "F",
  "capo": null,
  "tuning": "E A D G C F",
  "simplified": false,
  "has_simplified": true,
  "cifraclub_url": "https://www.cifraclub.com.br/coldplay/the-scientist/",
  "lines": [
    {"type": "lyric", "text": "Afinação: E A D G C F", "chords": []},
    {"type": "blank", "text": "", "chords": []},
    {"type": "section", "text": "[Primeira Parte]", "chords": []},
    {"type": "chord", "text": "Dm7             Bb9", "chords": [
      {"name": "Dm7", "index": 0}, {"name": "Bb9", "index": 16}
    ]},
    {"type": "lyric", "text": "    Come up to meet you", "chords": []}
  ]
}
```

- `?simplified=true` fetches the `simplificada.html` variant (fewer chord voicings); only
  meaningful when `has_simplified` is `true` on the base version.
- `lines[].text` preserves the original spacing (it is what lines chords up over the lyric in
  a monospace font); render in a monospace container. `chords[].index` is the character
  offset of the chord in `text` (redundant for plain rendering; useful for highlighting).
- `type`: `section` (`[Chorus]`), `chord` (chord-only), `lyric` (includes incidental metadata
  such as the tuning line, which Cifra Club does not mark differently), `blank`.
- `key`, `capo`, `tuning` may be `null`; `capo` is `null` when the site says
  "sem capotraste" (no capo).
- Errors: `404` song not found; `502` Cifra Club unreachable or unexpected page shape
  (markup changed).

### `POST /songs/from-url`

Body: `{"url": "https://www.cifraclub.com.br/<artist>/<song>/"}` (or the `simplificada.html`
variant). Response: same shape as the song endpoint. Fallback path when slug guessing fails
or the user already has a link. Only the first two path segments and the `simplificada*`
suffix are used; the URL host is never used to make the request (always
`https://www.cifraclub.com.br`).
Errors: `400` URL without `cifraclub.com.br` in the host or without artist/song segments;
`404`; `502`.

## Environment variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `ALLOWED_ORIGINS` | no | `*` | Comma-separated CORS origins. Production (gitops): `https://local.cmoreira.dev`. Compose: `http://localhost:3000` |

The port is **not** configurable by env: the `Dockerfile` hardcodes `--port 8080` (the old
table listed `PORT`, which the code never read). CORS allows `GET` and `POST`, any header.
Browser requests to `local.cmoreira.dev/sara/api` from `local.cmoreira.dev` are same-origin,
so CORS is not load-bearing in production (kept for clarity).

## Conventions and decisions

- Python + FastAPI + Uvicorn; async `httpx` (one request per page, 10 s timeout, follows
  redirects); BeautifulSoup + `lxml`. No Selenium/Playwright: every page is server-rendered
  HTML.
- Desktop-browser `User-Agent` (Chrome on Windows) on every request, since Cifra Club's CDN
  (Akamai) behaves better with browser-like clients.
- In-memory cache (`_cache` in `scraper.py`), 1 h TTL (`CACHE_TTL_S = 3600`), key = path,
  per process/replica, disposable on purpose (no Redis/DB). It only expires on read (entries
  are never evicted): unbounded growth in long-lived processes. Failures (404/errors) are
  not cached.
- No HTML/CSS/JS in this repo.

## Test locally

```bash
pip install -r requirements.txt
uvicorn main:app --reload --port 8080

curl http://localhost:8080/health
curl "http://localhost:8080/search?q=coldplay"
curl http://localhost:8080/artists/coldplay
curl http://localhost:8080/artists/coldplay/songs/the-scientist
curl -X POST http://localhost:8080/songs/from-url -H "Content-Type: application/json" \
  -d '{"url": "https://www.cifraclub.com.br/coldplay/the-scientist/"}'
```

Or via Docker: `sara/docker-compose.yml` (product folder root) runs API + UI. There are no
automated tests.

## Image (Dockerfile)

- Base `python:3-slim` (the local checkout uses `3.11-slim`; remote `main` is already on
  `3.14-slim` after a Renovate PR).
- Installs `curl` (used by `HEALTHCHECK`), copies `main.py` and `scraper.py`.
- Non-root: `useradd -u 10001 appuser` and `USER 10001` (numeric so Kubernetes
  `runAsNonRoot` can verify it).
- `EXPOSE 8080`; `HEALTHCHECK CMD curl -f http://localhost:8080/health`.
- Port 8080 (unprivileged) is required by the `restricted` `securityContext` of
  `generic-app` (see below).

## Deploy

Repo `gitops.local-sara` (wrapper charts around `gitops.generic-app-chart`, dependency
`generic-app` `0.7.0` at `oci://ghcr.io/cmoreira-dev/charts`). Two hand-written Argo CD
Applications in `argocd/` (`sara-api`, `sara-ui`), project `homelab`, destination namespace
`sara` (`CreateNamespace`, `ServerSideApply`, automated sync with `prune` and `selfHeal`).
Unlike the `gitops.template` default (one `helm/` per repo, empty `argocd/` generated by the
ApplicationSet), this repo ships two independent components, so `argocd/` is populated by
hand.

| Item | Value (`helm/api/values.yaml`) |
|---|---|
| Image | ECR `<account>.dkr.ecr.us-east-1.amazonaws.com/sara/api` (workflow `build-push.yml` calls `build-push-ecr.yml` from the `cmoreira-dev/.github` repo) |
| Tag | Short commit SHA, updated by automatic commits `build: automatic update of sara-api` (argocd-image-updater with git write-back) |
| Service | port 80 to `targetPort` 8080 |
| Route | `HTTPRoute` on Gateway `nginx-gateway-cmoreira-dev` (ns `nginx-gateway`), host `local.cmoreira.dev`, `PathPrefix /sara/api` with `URLRewrite` `ReplacePrefixMatch: /` |
| Config | `configMap` injected as env: `ALLOWED_ORIGINS=https://local.cmoreira.dev` |
| Secrets | none (no `ExternalSecret`): public scraper with no keys |

`/sara/api` beats the UI's `/sara` catch-all on the same host by Gateway API
longest-prefix-match, with no extra configuration. The rewrite exists because FastAPI is
unaware of the prefix (no `root_path`). See [Networking & Ingress](../architecture/networking.md)
and [Build & Registry](../cicd/build-registry.md).

!!! warning "Container security: `restricted`"
    The `generic-app` default `securityContext` (since 0.4.0) is `restricted` (non-root, no
    capabilities). The image must run non-root with a numeric UID on a port above 1024, and
    the gitops `targetPort` must match the real port.

!!! note "Coupling with the UI"
    A contract change (fields, `lines[].type`, errors) breaks the [UI](sara-ui.md)
    (`src/api/client.js` and viewer): change both sides together. A port, env or route change
    requires adjusting the values in `gitops.local-sara` in the same piece of work.

!!! note "Backstage catalog"
    `openapi.yaml` is a committed snapshot of `GET /openapi.json` (Backstage's processor does
    not read arbitrary internal hosts, and a file does not depend on the service being up).
    Regenerate with `curl <service>/openapi.json` after changing any endpoint. The generated
    spec does not describe response schemas (`schema: {}`), only input and `422`.
    `catalog-info.yaml` registers `sara-api` as a Component and an API (`system: sara`,
    `owner: platform-team`).

!!! warning "To confirm"
    - Liveness/readiness probes on the pod: `values.yaml` defines none; confirm the default of
      the `generic-app` 0.7.0 chart (the Dockerfile `HEALTHCHECK` is not used by Kubernetes).
    - Python version in production: the repo's local checkout is behind `origin/main`
      (local `3.11` image vs remote `3.14`); confirm the running tag.
