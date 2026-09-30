# teupadel: backlog e estado

!!! info "Fonte de verdade / atualizar aqui"
    Esta página é a única fonte de pendências e estado de follow-ups do teupadel.com. Substitui o
    `PENDENCIAS.md` da pasta do produto (e os planos antigos de contas/e-mail, hardening,
    processor/GPU e telemetria, já executados).

!!! tip "Como manter esta página"
    Atualize o estado dos itens **no mesmo trabalho que os altera** (mesmo PR ou mesma sessão), e date
    cada atualização (`AAAA-MM-DD`). Item concluído: mova para "Fechado" com a data e onde vive agora
    (repo, PR, página de docs). Item novo: acrescente na secção certa, com número seguinte.

Atualizado em **2026-09-30** (roadmap v1 em [teupadel-roadmap](teupadel-roadmap.md)). Estado: login (magic link + Google), bolas com ledger, indicações, análise
assíncrona com histórico, anti-abuso (Fase 6), Turnstile, worker de bounces do SES, bucket S3 da fila e
telemetria estão **em produção**. Ver [teupadel.com](../apps/teupadel.md).

## Produto

- **1.** **Fila durável de análises (lógica).** _Revisto em 2026-09-30: o upload passa a ser por URL pré-assinada direto para o S3 (roadmap, Fase 1), em vez de streaming pela API._ A infra (bucket `cmoreira-dev-teupadel-analysis-uploads` e
   permissões do usuário `teupadel-api`) já existe. Falta, na API:
    - migração com status `queued`, `heartbeat_at`, `attempts`, `video_key`;
    - upload em streaming (multipart) para o S3, o ponto mais delicado; tira também o vídeo de 100 MB da
      memória do pod (hoje exige 1Gi);
    - worker com `SELECT ... FOR UPDATE SKIP LOCKED`, heartbeat a cada 30 s, requeue se o heartbeat parar
      (> 2 min), máx. 2 tentativas e depois `failed` + estorno da bola;
    - apagar o vídeo do S3 no fim do job; `ANALYSIS_UPLOADS_BUCKET` no gitops.

    Estimativa: 7 a 10 h.
- **29.** **Notas por golpe e geral + gráfico de evolução** em "Minha conta" (desenho no
    [roadmap](teupadel-roadmap.md#notas-por-golpe-e-geral)): migração aditiva com `movement`, `score`,
    `reference_version`, `analysis_version`; `GET /me/progress`. _(aberto, 2026-09-30)_
- **30.** **PWA (Fase 2 do roadmap):** manifest, service worker só da casca, câmera guiada com MediaPipe,
    upload pré-assinado com retomada. _(aberto, 2026-09-30)_
- **31.** **Biblioteca de referência com professor:** hoje 5 clips de YouTube, sem calibração e com licença por
    rever. Bloqueia a calibração das notas (tolerâncias e pesos). Sem data. _(aberto, 2026-09-30)_
- **32.** **`EmailSender` com segundo provedor** (Brevo, Scaleway TEM, Postmark ou Resend), enquanto o magic link
    estiver bloqueado pelo SES (ver 6 e 7). _(aberto, 2026-09-30)_
- **2.** **Pagamento e planos.** `/pricing` e a waitlist existem; não há checkout nem compra de bolas.
   Instrumentar os eventos `payment_*` (helpers prontos) quando houver pagamento.
- **3.** **UX:**
    - o vídeo escolhido perde-se quando o visitante é mandado para o login;
    - o ecrã de carregamento não diz que se pode sair e ver o relatório no histórico;
    - o design ainda tem de confirmar o widget do Turnstile.
- **26.** **Turnstile em `/login` sob a nova CSP: testar no browser** (DevTools, sem erros de CSP) e percorrer
    `/analysis` de ponta a ponta: Next 16, React 19 e Node 24 foram para produção sem teste manual.
    _(aberto, 2026-09-29)_

## Legal e privacidade

- **4.** **Retenção dos relatórios:** decidir o prazo (`REPORT_RETENTION_DAYS`, hoje desligado).
- **5.** **Política de Privacidade e Termos** continuam como rascunho, sem revisão jurídica. O texto tem de
   citar os relatórios guardados, o S3 temporário (quando existir a fila), o banner de cookies e os
   subcontratantes (SES, Google, Cloudflare).

## E-mail (SES)

Runbook em [E-mail e SES](teupadel-email-ses.md).

- **6.** **Magic link nunca exercitado em produção:** precisa de um endereço verificado no SES ou de um
   `@teupadel.com`.
- **7.** **Sair do sandbox:** a AWS negou em 2026-09-29 (case 179054498900338). Reabrir em 2 a 3 semanas, agora
   com o worker de bounces validado; descrever o uso (só transacional, volume baixo, endereços de quem
   pede login, tratamento de bounces e reclamações). Texto pronto no apêndice do runbook.
- **8.** **Depois da aprovação:** validar SPF, DKIM e DMARC num e-mail real (Gmail e Outlook); configurar o envio
   como `contato@` (SMTP em `/teupadel/ses/gmail-smtp`); subir o DMARC de `none` para `quarantine` após
   2 a 4 semanas de relatórios limpos.

## Observabilidade

- **9.** **Alertas** (o repo `gitops.monitoring` não tem nenhuma regra; sem acesso ao Grafana Cloud daqui): Alloy
   não pronto / ausência de logs (`absent_over_time({namespace="teupadel", container="teupadel-api"}[10m])`);
   taxa de bounce > 2 %.
- **10.** **Automação dos dashboards:** a importação é manual. A cópia em
    `gitops.monitoring/dashboards/teupadel-telemetria.json` não tem as linhas "Autenticação" e "Bolas e
    indicações" (a versão completa é `dashboard-teupadel-telemetria.json` na pasta do produto). Importar
    também `dashboards/gpu.json`.
- **11.** **Alloy, médio prazo:** trocar `loki.source.kubernetes` (um stream por pod contra a API do Kubernetes)
    por `loki.source.file` a ler `/var/log/pods`. O `alloy-worker` no nó da GPU degrada ao fim de dias e
    perde logs; apagar o pod resolve, e o PR #26 adiciona liveness.
- **12.** **Telemetria fase 2:** Web Vitals (LCP/CLS/INP) no frontend; funil por sessão via TraceQL; ferramenta
    dedicada (Umami/Plausible/PostHog) só se surgirem perguntas de retenção/coorte.

## Plataforma

- **13.** **`gitops.generic-app-chart`:** não reinicia pods quando um `Secret` muda (foi preciso `rollout
    restart` à mão no Turnstile). Falta checksum das secrets nas annotations.
- **14.** **Backup do Postgres** com restauração real testada.
- **15.** **Testes:** ponta a ponta (Playwright) com o simulador do SES; teste completo em produção (cadastro,
    verificação, login pelos dois métodos, exclusão).
- **16.** **Nó da GPU "só GPU":** adiar até haver um 2.º worker amd64 (depois rebalancear e pôr taint).
- **27.** **Next 16: renomear `src/middleware.js` para `proxy` e investigar páginas dinâmicas.** O build avisa que
    `middleware` está obsoleto; as páginas passaram a ser renderizadas por pedido em vez de
    pré-renderizadas. _(aberto, 2026-09-29)_
- **28.** **Backstage: avisos de peer deps** (`jsdom` 30 vs `^27` do `@backstage/cli`). O fix do `yarn.lock` foi
    mergeado (backstage.homelab#32); os avisos ficam. _(aberto, 2026-09-29)_

## Segurança (achados da auditoria)

**Fechado em 2026-09-29:**

| # | Item | Estado |
|---|---|---|
| 17 | CSP na UI | UI#35, em produção; `script-src` mantém `'unsafe-inline'` (páginas pré-renderizadas). Ver [UI](../apps/teupadel-ui.md#seguranca) |
| 19 | JSON-LD escapado | feito |
| 20 (parte segura) | `npm audit fix`, `/_next/image` desligado, `X-Powered-By` removido | feito. Renovate mergeado hoje: Next 16, React 19, Node 24 (**sem teste manual**, ver 26) e os PRs das outras repos |
| 21 | HTTPS | Cloudflare "Always Use HTTPS" e HSTS ligados |
| 22 | `security-smoke.sh` | corrido em produção, tudo PASS |

**Ainda aberto:**

- **18.** **`NetworkPolicy` inertes até haver CNI que as aplique (Cilium/Calico).** Mergeadas (gitops#37) mas sem
    efeito: o CNI é flannel. Migrar de CNI é projeto próprio; depois restringir egress numa 2.ª passagem.
- **20.** **(restante) pose-estimation:** CUDA 13, onnxruntime novo, OpenCV 5 e NumPy 2 têm de voltar **juntos** e
    testados na GPU. A base foi revertida para CUDA 12.4 / onnxruntime 1.19.2 / OpenCV 4
    (pose-estimation#22). O PR do Renovate api.ia.pose-estimation#16 (NumPy 2) está mergeable, mas **não
    deve ser mergeado sozinho**.
- Ver também 26 (Turnstile sob a CSP), 27 (Next 16) e 28 (Backstage).
- Repetir `./security-smoke.sh` depois de cada deploy relevante.

## Go-live e Sara

- **23.** Tela de consentimento do Google em **"Em produção"**, com o cliente `teupadel-prod`.
- **24.** Testar a recepção de `contato@`, `suporte@` e `privacidade@`.
- **25.** **Sara:** replicar contas mais tarde; a policy do External Secrets (IaC `iam-external-secrets`) precisa
    do prefixo `/sara/*`.
