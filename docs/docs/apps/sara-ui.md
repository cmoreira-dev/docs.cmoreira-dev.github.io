# Sara: UI (`ui.ia.local-sara`)

!!! info "Fonte de verdade: código do repo; atualizar aqui em qualquer mudança funcional"
    Fonte: `sara/ui.ia.local-sara` (`src/`, `Dockerfile`, `next.config.js`) e
    `sara/gitops.local-sara`. Visão geral em [Sara](sara.md); contrato da API em
    [Sara: API](sara-api.md); pendências em [Sara: backlog](sara-backlog.md).

Player web do Sara: busca uma música, mostra cifra e letra com auto-scroll na BPM
escolhida, monta playlist e toca em sequência sem tocar no mouse para rolar. Alternativa
limpa ao Cifra Club para o que ele faz mal: acompanhar tocando (anúncios, vazamento de
memória que degrada a tela ao longo da sessão, sem scroll mãos-livres).

Sem lógica de negócio, sem scraping e sem chamadas diretas ao Cifra Club: tudo isso vive
em [`api.ia.local-sara`](sara-api.md). A UI só renderiza o que a API devolve.

A interface é **somente em inglês, de propósito**: Sara não fala português, embora o conteúdo
venha de um site brasileiro. Não há camada de i18n (ao contrário de `ui.ia.teupadel.com` com
`next-intl`); adicionar uma seria escopo extra para um app-presente de duas pessoas.

## Arquitetura

```
Browser
  -> Next.js (App Router, client components, servidor standalone)
      -> NEXT_PUBLIC_API_URL (api.ia.local-sara, FastAPI)   [chamado do browser]
          -> cifraclub.com.br (scraping server-side, nunca do browser)
```

A UI chama a API **direto do browser** (CORS habilitado na API), sem proxy via Route Handler.
Em teupadel o proxy existe para esconder um segredo (`ANTHROPIC_API_KEY`); a API do Sara não
tem segredos, então o salto extra não traria nada. Se isso mudar, espelhar o Route Handler do
teupadel em vez de `rewrites()` (que falha silenciosamente com `output: 'standalone'`).

!!! warning "Correção de doc antiga"
    O `docs/index.md` do repo diz que a UI alcança a API "pelo Service in-cluster". Falso:
    como o browser faz a chamada, usa a URL pública (`NEXT_PUBLIC_API_URL`, default
    `https://local.cmoreira.dev/sara/api`). Não há chamada UI para API dentro do cluster.

## Estrutura do repo

```
ui.ia.local-sara/
├── src/
│   ├── app/
│   │   ├── layout.jsx        # fontes (next/font/google: Inter, JetBrains Mono), PlaylistProvider, <html lang="en">
│   │   ├── page.jsx          # home: hero "Play with Sara" + busca + resultados
│   │   ├── globals.css       # design tokens como CSS vars
│   │   ├── icon.svg          # favicon (convenção App Router)
│   │   ├── health/route.js   # GET /health -> 200 "ok" (sob o basePath: /sara/health)
│   │   └── player/[artist]/[song]/page.jsx   # tela de prática
│   ├── components/           # Header, Hero, SearchBar, SongResultsList, PlaylistDrawer,
│   │                         # ChordLyricViewer, PlayerControls, ErrorState (+ .module.css)
│   ├── context/PlaylistContext.jsx   # playlist em localStorage
│   ├── hooks/useAutoScroll.js        # motor de auto-scroll por BPM
│   └── api/client.js                 # wrapper de fetch sobre a API
├── next.config.js            # output: 'standalone', basePath
├── Dockerfile, package.json, jsconfig.json
├── public/                   # só .gitkeep
└── catalog-info.yaml, docs/index.md, mkdocs.yml   # Backstage / TechDocs
```

## Fluxo de UX

1. **Home**: barra de busca única. Se o texto contém `cifraclub.com.br` (regex), chama
   `POST /songs/from-url` e vai direto ao player; senão chama `GET /search?q=` e lista as
   músicas do artista (`SongResultsList`), com selo "Best match" na
   `matched_song_slug`, botões "+ Playlist" e "Play". Erro da API aparece em `ErrorState`
   (`detail` da resposta). Placeholder sugere "artista + música" ou colar link.
2. **Player** (`/player/<artist>/<song>`): título, artista, tags `Key`/`Capo` (se existirem),
   "+ Playlist", link "Source" para o Cifra Club. Viewer monoespaçado com acordes destacados,
   seções sem colchetes. Controles: Prev / Play-Pause / Next, slider de BPM
   (40 a 220, passo 2, padrão 90) e toggle Full/Simplified (só se `has_simplified`).
   Trocar Full/Simplified refaz o fetch e volta o scroll ao topo.
3. **Playlist** (`PlaylistDrawer`, botão no `Header` com contador): Play por item, Remove,
   "Play all" (índice 0) e Clear. Sem reordenação (MVP).

## Auto-scroll (`useAutoScroll`)

Existe por causa da queixa original (anúncios, vazamento de memória, sem scroll
mãos-livres). Restrições de projeto:

- **Um único loop `requestAnimationFrame`, sempre cancelado**: no cleanup do effect a cada
  mudança de dependência e no unmount, sem `setInterval`/rAF pendente. Preservar essa
  propriedade ao mexer no hook: é o ponto do app.
- Velocidade por BPM, não px/frame fixo: `pxPerSecond = 16 * (bpm / 60)`. O `16`
  (`PX_PER_BEAT`) é constante ajustada, não derivada de compasso: o Cifra Club não tem timing
  por linha, então é "BPM maior rola mais rápido", não metrônomo. Sync de batida real
  exigiria dados de timing que o site não expõe.
- A posição é acumulada em JS (`positionRef`) e só o resultado é escrito no `scrollTop`: o
  `scrollTop` do DOM é inteiro, e o delta por frame é fração de pixel (~0,4 px a 90 BPM e
  60 fps), que seria arredondado a zero a cada frame.
- `bpm` e `onReachEnd` são lidos via refs dentro do loop, então mudar BPM em movimento não
  derruba/reinicia o loop (evita tremor); só `playing` reinicia.
- Chegar ao fim do container dispara `onReachEnd`; o player avança para a próxima música
  da playlist com autoplay (o requisito "um clique para tocar o set"). Sem próxima, pausa.

## Playlist (só cliente)

`PlaylistContext` persiste em `localStorage` na chave `sara.playlist`, sem backend nem
autenticação (ferramenta de duas pessoas; infra de persistência seria overhead). Itens:
`{artistSlug, songSlug, artist, title}`, sem duplicatas (chave `artistSlug/songSlug`).
Sincronizar entre dispositivos exigiria um store real: só nesse momento.

O player lê `?playlistIndex=N` para saber que toca a música N da playlist (Prev/Next e
avanço automático). Música aberta direto (busca) não tem `playlistIndex`: toca avulsa e
Prev/Next ficam desabilitados. `?autoplay=1` faz o avanço automático começar já rolando;
sem ele, abrir uma música sempre começa pausado.

## Design system

Linguagem visual adaptada de `sara/DESIGN-together.ai.md` (hero escuro, corpo branco, eyebrow
em mono maiúsculo, CTA único em pílula preta). Tokens viram custom properties em
`src/app/globals.css` (`--color-*`, `--space-*`, `--radius-*`). Substitutos de fonte
conforme o próprio doc: `Inter` (sans) e `JetBrains Mono` (eyebrow/label), via
`next/font/google` em `layout.jsx` (self-hosted no build, sem CDN externo, sem CLS).

- CSS Modules por componente; sem Tailwind e sem biblioteca de componentes (mesma convenção
  de `ui.ia.teupadel.com`).
- Acordes em `var(--color-accent-magenta)`, peso 500: único uso do acento como preenchimento
  sólido, exceção deliberada e restrita; não espalhar acentos (regra "não introduza um
  quinto acento" do doc de design).
- O doc `DESIGN-together.ai.md` mora em `sara/` (raiz do produto), não dentro do repo da UI.

## Variáveis de ambiente (build-time)

| Variável | Obrigatória | Default (`ARG` do Dockerfile) | Descrição |
|---|---|---|---|
| `NEXT_PUBLIC_API_URL` | não | `https://local.cmoreira.dev/sara/api` | URL base da API, chamada direto do browser (`client.js` cai em `http://localhost:8080` se ausente) |
| `NEXT_BASE_PATH` | não | `/sara` | Vira `basePath` em `next.config.js` |

**Ambas são de build, não de runtime.** `NEXT_PUBLIC_*` é inlined no bundle em `npm run build`
e `basePath` é opção de build: um `environment:` no `docker run`/compose não as altera. O
Dockerfile as recebe como `ARG`.

**Os defaults dos `ARG` são os valores de produção**, porque o workflow reutilizável de CI
(`cmoreira-dev/.github` -> `build-push-ecr.yml`) faz `docker build .` simples, **sem
`--build-arg`**. O `docker-compose.yml` sobrescreve ambos para local
(`NEXT_PUBLIC_API_URL=http://localhost:8080`, `NEXT_BASE_PATH` vazio). Nova variável
build-time exige o mesmo tratamento nos dois lados, senão local e produção divergem em
silêncio.

## Convenções e stack

- Next.js `^16.0.0`, React `^19.0.0` (`package.json` de `origin/main`), Node 24
  (`node:24-slim` no Dockerfile). A checkout local usada para redigir esta página ainda
  estava em Next `^15`/React `^18`/Node 20 (atrás de `origin/main`). Componentes de função +
  hooks; sem Redux/Zustand.
- JavaScript puro (sem TypeScript), `fetch` nativo (sem axios), sem `next-intl`.
- Alias `@/*` -> `./src/*` (`jsconfig.json`).
- Sem `package-lock.json` no repo: o Dockerfile usa `npm install --prefer-offline`.
- Os bumps do Renovate para Next 16, React 19 e Node 24 já estão mergeados em `origin/main`
  (a validação do build/runtime sob essas versões está no [backlog](sara-backlog.md)).

## Testar localmente

```bash
npm install
NEXT_PUBLIC_API_URL=http://localhost:8080 npm run dev
# http://localhost:3000
```

Build de produção: `npm run build` e `NEXT_PUBLIC_API_URL=http://localhost:8080 npm start`.
Ou via Docker com `sara/docker-compose.yml` (junto da API). Sem testes automatizados; os
repos não têm CI de PR, só build-on-push na `main`: validar `docker build` localmente antes
de mergear.

## Imagem (Dockerfile)

Dois estágios `node:24-slim` (`origin/main`; `node:20-slim` na checkout local desatualizada). Builder recebe os `ARG`, roda `npm install` e `npm run build`.
Runner: `NODE_ENV=production`, `PORT=3000`, `HOSTNAME=0.0.0.0`, copia `.next/standalone`,
`.next/static` e `public` com `--chown=1000:1000`, `USER 1000` (uid do usuário `node`,
numérico para o `runAsNonRoot` verificar), `EXPOSE 3000`, `CMD ["node", "server.js"]`.

!!! warning "Non-root e porta 3000 são obrigatórios"
    O `generic-app` 0.7.0 aplica `securityContext` `restricted` (non-root, drop ALL
    capabilities, sem `NET_BIND_SERVICE`): a imagem tem de rodar non-root e em porta acima de
    1024, com `service.targetPort` (3000) igual em `helm/ui/values.yaml`. As branches
    remotas antigas do Renovate (`renovate/major-nextjs-monorepo`, `major-react-monorepo`,
    `node-24.x`), já mergeadas, ainda existem no remoto com base anterior à correção
    (`PORT=80`, sem `USER`): não reaproveitar; bumps novos devem preservar o Dockerfile
    non-root.

## Deploy (`local.cmoreira.dev/sara`)

`local.cmoreira.dev` é hostname compartilhado (Gateway `nginx-gateway-cmoreira-dev` em
`gitops.core-addons`, já tunelado via Cloudflare por host, não por path: um novo path não
exige mudança lá). O `HTTPRoute` da UI casa `PathPrefix /sara` e encaminha **sem** remover o
prefixo, exatamente por isso o `basePath` existe: o Next.js precisa saber que está montado em
`/sara` para prefixar links, assets e fetches RSC, e espera o path com `/sara` intacto (um
`URLRewrite` quebraria isso). A API fica em `/sara/api` e **remove** o prefixo. A Gateway API
resolve por longest-prefix-match, então `/sara/api/*` vence o catch-all `/sara` da UI.

| Item | Valor (`helm/ui/values.yaml`) |
|---|---|
| Imagem | ECR `<conta>.dkr.ecr.us-east-1.amazonaws.com/sara/ui` (workflow `build-push.yml` -> `build-push-ecr.yml`) |
| Tag | SHA curto, atualizada por commits `build: automatic update of sara-ui` (argocd-image-updater) |
| Service | porta 80 para `targetPort` 3000 |
| Rota | Gateway `nginx-gateway-cmoreira-dev` (ns `nginx-gateway`), host `local.cmoreira.dev`, `PathPrefix /sara`, sem filtro |
| Config/segredos | nenhum `configMap` nem `ExternalSecret` (config é build-time) |

Namespace `sara`, Application `sara-ui` (projeto `homelab`, sync automatizado). Detalhes do
gitops e do wrapper de `gitops.generic-app-chart` em [Sara: API](sara-api.md#deploy).

!!! warning "Confirmar"
    - `docker-compose.yml` mapeia a UI como `3000:80`, mas o container agora escuta na 3000
      (`PORT=3000`): provável mapeamento desatualizado (ver [backlog](sara-backlog.md)).
    - Probes de liveness/readiness da UI: `values.yaml` não define; confirmar o default do
      chart. `/sara/health` existe na app.
    - Next 16 / React 19 / Node 24 (mergeados hoje, 2026-09-29): não validei que o build
      standalone e o runtime non-root continuam ok nessas versões (a checkout local estava
      atrás, só li o diff: apenas `Dockerfile` e `package.json` mudaram).
