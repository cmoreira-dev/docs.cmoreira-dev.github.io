# Sara: API (`api.ia.local-sara`)

!!! info "Fonte de verdade: código do repo; atualizar aqui em qualquer mudança funcional"
    Fonte: `sara/api.ia.local-sara` (`main.py`, `scraper.py`, `Dockerfile`) e
    `sara/gitops.local-sara`. Visão geral do produto em [Sara](sara.md); UI em
    [Sara: UI](sara-ui.md); pendências em [Sara: backlog](sara-backlog.md).

Backend FastAPI que resolve artistas e músicas no [Cifra Club](https://www.cifraclub.com.br)
e devolve cifra/letra estruturada em JSON. Somente API: nenhum HTML/CSS/JS, sem banco,
sem segredos.

## Estrutura do repo

```
api.ia.local-sara/
├── main.py            # app FastAPI + rotas + CORS
├── scraper.py         # todo o HTTP e parsing do Cifra Club
├── requirements.txt   # fastapi, uvicorn[standard], httpx, beautifulsoup4, lxml (sem pins exatos)
├── Dockerfile
├── openapi.yaml       # snapshot commitado da spec (catálogo Backstage)
├── catalog-info.yaml  # Component + API entity (Backstage)
├── docs/index.md, mkdocs.yml   # TechDocs do repo
└── .github/workflows/build-push.yml   # build e push para o ECR
```

## Por que scraping, e por que esta abordagem

O Cifra Club não tem API pública de busca. A busca do site (`/?q=...`) é client-side e chama
um endpoint privado `/api/...` (Google Custom Search), que está em `Disallow` no
`robots.txt` e é travado por chave/referrer: **não deve ser chamado**. O que é permitido e
é HTML server-rendered:

- Página de artista `GET /<artist-slug>/`: lista as ~15 músicas mais populares como links
  `<a href="/<artist-slug>/<song-slug>/">`.
- Página de música `GET /<artist-slug>/<song-slug>/`: cifra e letra em
  `<pre data-chord-content="true">`, um `<div>` por trecho; acordes como
  `<b data-chord-name="...">`.

### Resolução de artista (`resolve_search`)

1. Slugify da query (minúsculas, sem acentos, hífens) e `GET /<slug>/`: cobre "Coldplay" e
   "Legiao Urbana".
2. Se der 404 e a query tiver 2+ palavras, assume "artista + música" misturados
   (ex.: "coldplay yellow", "yellow coldplay") e tenta cada divisão em fronteira de palavra,
   nas duas ordens, como página `/<artista>/<música>/`, limitado a `_MAX_SPLIT_ATTEMPTS`
   (8) requisições. A primeira divisão que resolve vence: devolve a lista de músicas do
   artista com a música casada fixada primeiro e `matched_song_slug` preenchido.
3. Nada resolveu: `404`.

Limitação aceita: uma palavra única ambígua (ex.: "yellow") resolve o artista homônimo
real (`/yellow/`), não a música do Coldplay, pois não há segunda palavra para dividir.
Resolver de verdade exige índice/API de busca (Google Custom Search em
`site:cifraclub.com.br` ou crawl do sitemap), considerado dependência externa demais para
uma ferramenta de duas pessoas. A UI incentiva "artista + música" e o atalho de colar link
(`POST /songs/from-url`), que funciona independentemente da busca.

### Parsing

- Seletores nunca por classe CSS (hasheadas, mudam a cada deploy do Cifra Club): título da
  música em lista por `alt` da imagem (`Capa da música "..."`) com fallback para classe
  `primaryLabel` parcial; tom por `button[data-anchor="--chord-tone"]`; capo e afinação por
  texto de label (`Capotraste`, `Afinação`) seguido do próximo `<p>`; `has_simplified` por
  link `/<artista>/<música>/simplificada.html`.
- Cada `<div>` do bloco de cifra é um *couplet* (linha de acordes + linha de letra unidas
  por `\n`). `scraper._lines_from_div` achata em tokens e divide em `\n`, recalculando
  `chords[].index` por linha. Não assumir 1 div == 1 linha.
- Classificação de linha (`_classify_line`): `blank`, `section` (entre colchetes),
  `chord` (só acordes após mascarar os acordes), senão `lyric`.
- Artistas/músicas em listas ignoram slugs `discografia` e `videoaulas`; só links que casam
  `^/([a-z0-9-]+)/([a-z0-9-]+)/$` com o slug do artista.

## Endpoints

Base pública: `https://local.cmoreira.dev/sara/api` (o prefixo `/sara/api` é removido antes
de chegar no container; localmente as rotas são na raiz, porta 8080).

### `GET /health`

```json
{"status": "ok"}
```

### `GET /search?q=<query>`

Resolução descrita acima. Mesmo formato de `GET /artists/{artist_slug}` mais
`matched_song_slug` (só presente quando uma divisão artista+música casou).
Erros: `400` se `q` vazio (após `strip`); `404` nada resolve (a mensagem sugere colar link);
`502` Cifra Club inacessível; `422` se `q` ausente (validação do FastAPI).

### `GET /artists/{artist_slug}`

```json
{
  "slug": "coldplay",
  "name": "Coldplay",
  "songs": [{"slug": "the-scientist", "title": "The Scientist"}]
}
```

Erros: `404`, `502`.

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

- `?simplified=true` busca a variante `simplificada.html` (menos vozes de acorde); só faz
  sentido se `has_simplified` for `true` na versão base.
- `lines[].text` preserva o espaçamento original (é o que alinha acordes sobre a letra em
  fonte monoespaçada); renderizar em container monoespaçado. `chords[].index` é o offset
  de caractere do acorde em `text` (redundante para render simples; útil para destaque).
- `type`: `section` (`[Refrão]`), `chord` (só acordes), `lyric` (inclui metadados
  incidentais como a linha de afinação, que o Cifra Club não distingue), `blank`.
- `key`, `capo`, `tuning` podem ser `null`; `capo` é `null` quando o site diz
  "sem capotraste".
- Erros: `404` música não encontrada; `502` Cifra Club inacessível ou formato de página
  inesperado (markup mudou).

### `POST /songs/from-url`

Corpo: `{"url": "https://www.cifraclub.com.br/<artist>/<song>/"}` (ou variante
`simplificada.html`). Resposta: mesmo formato da música. É o caminho de fallback quando o
palpite de slug falha ou o usuário já tem o link. Só os dois primeiros segmentos do path e
o sufixo `simplificada*` são usados; o host da URL não é usado para requisitar (sempre
`https://www.cifraclub.com.br`).
Erros: `400` URL sem `cifraclub.com.br` no host ou sem segmentos artista/música; `404`;
`502`.

## Variáveis de ambiente

| Variável | Obrigatória | Default | Descrição |
|---|---|---|---|
| `ALLOWED_ORIGINS` | não | `*` | Origens CORS separadas por vírgula. Produção (gitops): `https://local.cmoreira.dev`. Compose: `http://localhost:3000` |

A porta **não** é configurável por env: `Dockerfile` fixa `--port 8080` (a tabela antiga
listava `PORT`, que o código nunca leu). CORS permite métodos `GET` e `POST`, qualquer header.
Requisições do browser para `local.cmoreira.dev/sara/api` a partir de `local.cmoreira.dev`
são same-origin, então o CORS não é crítico em produção (mantido por clareza).

## Convenções e decisões

- Python + FastAPI + Uvicorn; `httpx` assíncrono (uma requisição por página, timeout 10 s,
  segue redirects); BeautifulSoup + `lxml`. Sem Selenium/Playwright: todas as páginas são
  HTML server-rendered.
- `User-Agent` de browser desktop (Chrome no Windows) em toda requisição, pois o CDN
  (Akamai) do Cifra Club se comporta melhor com clientes tipo browser.
- Cache em memória (`_cache` em `scraper.py`), TTL 1 h (`CACHE_TTL_S = 3600`), chave = path,
  por processo/réplica, descartável de propósito (sem Redis/DB). O cache só expira na
  leitura (entradas nunca são removidas): crescimento ilimitado em processos longos.
  Falhas (404/erro) não são cacheadas.
- Sem HTML/CSS/JS neste repo.

## Testar localmente

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

Ou via Docker: `sara/docker-compose.yml` (raiz da pasta do produto) sobe API + UI.
Não há testes automatizados.

## Imagem (Dockerfile)

- Base `python:3-slim` (a checkout local usa `3.11-slim`; a `main` remota já está em
  `3.14-slim` após PR do Renovate).
- Instala `curl` (usado no `HEALTHCHECK`), copia `main.py` e `scraper.py`.
- Non-root: `useradd -u 10001 appuser` e `USER 10001` (numérico, para o
  `runAsNonRoot` do Kubernetes conseguir verificar).
- `EXPOSE 8080`; `HEALTHCHECK CMD curl -f http://localhost:8080/health`.
- Porta 8080 (não privilegiada) é exigência do `securityContext` `restricted` do
  `generic-app` (ver abaixo).

## Deploy

Repo `gitops.local-sara` (wrapper charts sobre `gitops.generic-app-chart`, dependência
`generic-app` `0.7.0` em `oci://ghcr.io/cmoreira-dev/charts`). Dois Applications do Argo CD
escritos à mão em `argocd/` (`sara-api`, `sara-ui`), projeto `homelab`, destino namespace
`sara` (`CreateNamespace`, `ServerSideApply`, sync automatizado com `prune` e `selfHeal`).
Diferente do padrão `gitops.template` (um `helm/` por repo, `argocd/` vazio gerado pelo
ApplicationSet), o repo entrega dois componentes independentes, por isso `argocd/` é
preenchido manualmente.

| Item | Valor (`helm/api/values.yaml`) |
|---|---|
| Imagem | ECR `<conta>.dkr.ecr.us-east-1.amazonaws.com/sara/api` (workflow `build-push.yml` chama `build-push-ecr.yml` do repo `cmoreira-dev/.github`) |
| Tag | SHA curto do commit, atualizada por commits automáticos `build: automatic update of sara-api` (argocd-image-updater com write-back git) |
| Service | porta 80 para `targetPort` 8080 |
| Rota | `HTTPRoute` no Gateway `nginx-gateway-cmoreira-dev` (ns `nginx-gateway`), host `local.cmoreira.dev`, `PathPrefix /sara/api` com `URLRewrite` `ReplacePrefixMatch: /` |
| Config | `configMap` injetado como env: `ALLOWED_ORIGINS=https://local.cmoreira.dev` |
| Segredos | nenhum (sem `ExternalSecret`): scraper público sem chaves |

O `/sara/api` vence o `/sara` (catch-all da UI) no mesmo host por longest-prefix-match da
Gateway API, sem configuração extra. O rewrite existe porque o FastAPI não conhece o
prefixo (sem `root_path`). Ver [Rede & Ingress](../architecture/networking.md) e
[Build & Registry](../cicd/build-registry.md).

!!! warning "Segurança do container: `restricted`"
    O default de `securityContext` do `generic-app` (desde 0.4.0) é `restricted`
    (non-root, sem capabilities). A imagem precisa rodar non-root com UID numérico e em
    porta acima de 1024, e o `targetPort` do gitops deve bater com a porta real.

!!! note "Acoplamento com a UI"
    Mudança de contrato (campos, `lines[].type`, erros) quebra a [UI](sara-ui.md)
    (`src/api/client.js` e viewer): alterar os dois lados juntos. Mudança de porta, env ou
    rota exige ajuste dos values em `gitops.local-sara` no mesmo trabalho.

!!! note "Catálogo Backstage"
    `openapi.yaml` é um snapshot commitado de `GET /openapi.json` (o processador do
    Backstage não lê hosts internos arbitrários, e um arquivo não depende do serviço estar
    de pé). Regenerar com `curl <serviço>/openapi.json` após mudar qualquer endpoint. A
    spec gerada não descreve os schemas de resposta (`schema: {}`), só entrada e `422`.
    O `catalog-info.yaml` registra `sara-api` como Component e API (`system: sara`,
    `owner: platform-team`).

!!! warning "Confirmar"
    - Probes de liveness/readiness no pod: `values.yaml` não define nenhuma; confirmar o
      default do chart `generic-app` 0.7.0 (o `HEALTHCHECK` do Dockerfile não é usado pelo
      Kubernetes).
    - Versão de Python em produção: a checkout local do repo está atrás de `origin/main`
      (imagem `3.11` local vs `3.14` remota); confirmar a tag em execução.
