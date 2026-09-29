# Sara: backlog e estado

!!! info "Fonte de verdade: código do repo; atualizar aqui em qualquer mudança funcional"
    Estado e pendências do produto Sara (repos `api.ia.local-sara`, `ui.ia.local-sara`,
    `gitops.local-sara`). Componentes: [API](sara-api.md), [UI](sara-ui.md); visão geral em
    [Sara](sara.md).

!!! tip "Como manter esta página"
    Atualize os itens **no mesmo trabalho** que os altera (fechar, mover, criar). Toda
    atualização leva data (`AAAA-MM-DD`). Item concluído vai para "Concluído" com a data,
    não é apagado em silêncio. Item que virar decisão permanente migra para a página do
    componente.

Levantamento inicial em 2026-09-29, a partir dos `CLAUDE.md` antigos dos repos, do código e
dos values do gitops.

## Aberto

| # | Item | Repo | Detalhe | Atualizado |
|---|---|---|---|---|
| 1 | Replicar as contas do Sara e incluir `/sara/*` na policy IAM do External Secrets | `iac.homelab-live-infra` / `gitops.core-addons` | Hoje o Sara não usa `ExternalSecret`. Quando houver segredo (ex.: contas/credenciais), a policy IAM do External Secrets Operator precisa ganhar o prefixo `/sara/*` (hoje escopada por path: `homelab/*` e `teupadel/*`, ver [Segredos](../architecture/secrets.md)) e o `ExternalSecret` do `gitops.local-sara` deve referenciar `/homelab/sara/...` ou `/sara/...` conforme a convenção adotada. Origem: notas do repo raiz. | 2026-09-29 |
| 2 | Validar Next 16 / React 19 / Node 24 em produção | `ui.ia.local-sara` | Bumps do Renovate mergeados em `origin/main` em 2026-09-29 (só `Dockerfile` e `package.json` mudaram). Confirmar build standalone, runtime non-root (`USER 1000`, porta 3000), auto-scroll e navegação no cluster. | 2026-09-29 |
| 3 | `docker-compose.yml` da UI com porta desatualizada | `sara/docker-compose.yml` | Mapeia `3000:80`, mas o container agora escuta na 3000 (`PORT=3000`, non-root): deve ser `3000:3000`. Confirmar rodando o compose. | 2026-09-29 |
| 4 | Limpar branches Renovate remanescentes | `ui.ia.local-sara` | `renovate/major-nextjs-monorepo`, `major-react-monorepo` e `node-24.x` ainda existem no remoto e suas pontas são baseadas na `main` antiga (sem o Dockerfile non-root: `PORT=80`, sem `USER`). Os PRs já foram mergeados (merge preservou o non-root), mas apagar as branches evita reaproveitá-las por engano. | 2026-09-29 |
| 5 | Checkouts locais atrás de `origin/main` | `ui.ia.local-sara`, `api.ia.local-sara`, `gitops.local-sara` | UI (Next 16/React 19/Node 24), API (Python 3.14) e gitops (tags de imagem) têm commits remotos não puxados; `git pull` antes de trabalhar. | 2026-09-29 |
| 6 | Sem lockfile na UI | `ui.ia.local-sara` | Sem `package-lock.json`; Dockerfile usa `npm install`: builds não reprodutíveis. Avaliar commitar o lockfile e usar `npm ci`. | 2026-09-29 |
| 7 | `requirements.txt` sem pins exatos | `api.ia.local-sara` | Só `>=`; build pode mudar com versões novas. Renovate não tem o que bumpar. | 2026-09-29 |
| 8 | Cache do scraper sem limite | `api.ia.local-sara` | `_cache` só expira na leitura e nunca remove entradas; cresce em processos longos, por réplica. Aceitável para duas pessoas; revisar se o uso crescer. | 2026-09-29 |
| 9 | Busca por palavra única ambígua | `api.ia.local-sara` | "yellow" resolve o artista homônimo. Correção real exige índice/API de busca (Google Custom Search `site:cifraclub.com.br` ou sitemap). Limitação aceita; contorno é colar o link. | 2026-09-29 |
| 10 | Sincronizar playlist entre dispositivos | `ui.ia.local-sara` | Hoje só `localStorage`. Só faz sentido se Sara e o amigo precisarem; exigiria store server-side e auth. | 2026-09-29 |
| 11 | Sync de batida real no auto-scroll | `ui.ia.local-sara` | `PX_PER_BEAT = 16` é constante ajustada; o Cifra Club não expõe timing por linha. Fora de escopo. | 2026-09-29 |
| 12 | Confirmar probes no pod | `gitops.local-sara` | `values.yaml` de API e UI não definem liveness/readiness; confirmar o default do `generic-app` 0.7.0 (o `HEALTHCHECK` do Dockerfile não vale no Kubernetes). | 2026-09-29 |
| 13 | Avanço automático da playlist retoma o scroll? | `ui.ia.local-sara` | O loop termina ao chegar no fim e `playing` continua `true`; o `useAutoScroll` só reinicia se `playing` mudar. Se o Next remontar a página ao trocar `[artist]/[song]`, funciona; confirmar no browser. Também: `simplified` pode persistir entre músicas sem versão simplificada (confirmar). | 2026-09-29 |
| 14 | Docs de repo (`docs/index.md`) com afirmação errada | `ui.ia.local-sara` | Diz que a UI alcança a API pelo Service in-cluster; na verdade o browser usa a URL pública. Corrigir (arquivos fora do escopo desta reorganização). | 2026-09-29 |
| 15 | `README.md` da API com comentário de teste | `api.ia.local-sara` | Termina com comentário HTML "trivial change ... take 3" (resíduo do teste de write-back do image-updater). Remover. | 2026-09-29 |

## Concluído

| Item | Onde | Data |
|---|---|---|
| Repos no GitHub, CI para o ECR (`sara/api`, `sara/ui`) e Applications do Argo CD no ar em `local.cmoreira.dev/sara` (os `CLAUDE.md` antigos diziam "repo local ainda não enviado ao GitHub") | os três repos | verificado em 2026-09-29 |
| UI non-root (uid 1000, porta 3000) e API com `USER 10001` numérico para o `restricted` do `generic-app` 0.7.0 | `ui.ia.local-sara`, `api.ia.local-sara` | anterior a 2026-09-29 |
| Onboarding no Backstage (`catalog-info.yaml`, TechDocs, `openapi.yaml` commitado) | os três repos | anterior a 2026-09-29 |
| Bumps Renovate: Next 16, React 19, Node 24 (UI), Python 3.14 (API) mergeados em `origin/main` | `ui.ia.local-sara`, `api.ia.local-sara` | 2026-09-29 (validação no item 2) |

!!! warning "Confirmar"
    - O item 1 vem das notas do repo raiz; o caminho exato do parâmetro SSM e o nome da
      policy IAM não foram verificados aqui (não há `ExternalSecret` do Sara no gitops).
    - "Contas do Sara a replicar": escopo (quais contas/credenciais e para onde) não está
      documentado em nenhum arquivo lido; confirmar com o dono.
