# teupadel: roadmap de produto v1

Estado: proposta aprovada em **2026-09-30**. Web/PWA primeiro, app nativo depois. Junta a visão
"treinador digital no bolso" com o modelo de monetização (bolas, pacotes 5/15/40, Plano Evolução). Onde
houver conflito, valem as decisões de monetização. Pendências e estado de execução ficam no
[backlog](teupadel-backlog.md).

## Princípios

1. **O vídeo é temporário.** Vive num bucket S3 só para o processamento assíncrono (a análise continua se
   a pessoa fechar o browser depois do upload). O backend apaga o vídeo quando a análise termina, com
   sucesso ou falha; uma regra de ciclo de vida de 1 dia apaga o que sobrar. Ficam o relatório e os
   **keypoints da pose**, que permitem recalcular as notas quando o motor evoluir.
    - Bucket na UE, privado, cifrado, upload por URL pré-assinada, **sem versionamento** (senão o "apagar"
      deixa cópias). Vídeos rejeitados ou com erro também são apagados.
    - Texto ao usuário: "O teu vídeo é apagado logo após a análise (no máximo 24 h). Guardamos só o
      relatório." O prazo tem de estar na política de privacidade (RGPD/LGPD), e os keypoints contam
      como dado pessoal.
2. **A nota é determinística.** Vem da comparação da pose com a biblioteca de referência; o LLM só explica.
   Cada análise guarda `reference_version` e `analysis_version`.
3. **Web/PWA antes de app nativo.** Validar retenção (análises por cliente por mês) antes de Flutter/React
   Native e das regras da App Store.
4. **Uma moeda: bolas.** O ledger guarda `provider` (stripe | app_store | play).

## Autenticação sem SES

A AWS recusou tirar o SES do sandbox e o **magic link fica bloqueado** até nova aprovação (ver
[E-mail e SES](teupadel-email-ses.md)). O login com Google funciona e conta como e-mail verificado para as
bolas de boas-vindas e o antiabuso.

- O envio de e-mail passa a ficar atrás de uma interface (`EmailSender`, já existe em `email_sender.py`),
  com SES e um segundo provedor configurável (Brevo, Scaleway TEM, Postmark ou Resend).
- Novo pedido ao SES em 2 a 3 semanas (só transacional, volume baixo, bounces por SNS).
- Sign in with Apple quando houver app iOS. Sem e-mail não há recibos por e-mail; não bloqueia, o Stripe
  está em modo de teste.

## Notas: por golpe e geral

Objetivo: cada análise produz uma **nota do golpe** e contribui para uma **nota geral**, para o "Minha
conta" mostrar o gráfico de evolução por golpe e o geral.

### Da distância à nota

O `dtw_compare.py` já devolve, por fase e por feature, `deviation_avg` e `dtw_distance` contra a
referência. A nota é calculada em camadas, todas determinísticas:

1. **Feature → 0 a 100.** Cada feature tem uma tolerância `tol` (em graus, ou fração do canvas para
   posição/velocidade): `nota = 100 · max(0, 1 − desvio / (k · tol))`, com `k` a fixar na calibração.
2. **Fase → nota.** Média ponderada das features da fase (pesos numa tabela por golpe e fase).
3. **Golpe (análise) → nota do golpe.** Média ponderada das fases (preparação, contato, finalização). Com
   vários golpes no vídeo, média dos golpes válidos.
4. **Nota geral.** Por análise, a nota do golpe é também a nota geral daquele momento. No gráfico "Geral",
   cada ponto é a **média ponderada da última nota de cada golpe** nos últimos 30 dias. Assim o geral
   não sobe nem desce só por trocar de golpe.

Features sem dado (joelho fora de quadro, por exemplo) saem da média e os pesos renormalizam; se faltar
mais de metade do peso da fase, a fase fica sem nota (`null`) em vez de inventar um valor.

### Estrutura guardada

`reports.result` passa a ter um bloco estruturado (o texto do modelo não entra no cálculo):

```json
{
  "scores": {
    "overall": 72,
    "movement": "forehand",
    "movement_score": 72,
    "phases": {"preparation": 80, "contact": 65, "follow_through": null},
    "metrics": {"elbow_angle_right_deg": {"score": 70, "deviation": 11.2}}
  },
  "reference_version": "2026-09-26-yt5",
  "analysis_version": "1"
}
```

Para o gráfico, `movement`, `score`, `reference_version`, `analysis_version` e `finished_at` também vão
para colunas de `reports` (migração aditiva), com índice `(user_id, movement, finished_at)`. Os
keypoints já ficam em `reports.result.pose_frames`, o que permite recalcular notas mais tarde sem reprocessar vídeo.

### Gráfico em "Minha conta"

- `GET /me/progress` devolve as séries por golpe (`movements`), a `overall` e `version_breaks`; a UI filtra.
- Uma linha por golpe mais a linha "Geral". Mudança de `reference_version` ou `analysis_version` aparece
  como quebra no gráfico, **ou** as notas antigas são recalculadas a partir dos keypoints. Decisão
  (2026-09-30): começar com a quebra visível, com a mensagem "Refinámos o modelo".
- Um ponto só se mostra com 2 ou mais análises do golpe; antes disso, mostrar a nota sozinha.

### Calibração (depende do professor)

As notas só significam algo depois de a biblioteca ter vídeos de referência boas. Hoje são 5 clips de
YouTube (1 por combinação golpe/fase), **indicativos e sem calibração**; além disso, a licença desses clips
precisa de revisão antes de publicar as notas. A biblioteca definitiva será filmada com um professor.
Até lá:

- mostrar a nota como **"beta"** (e explicar as quebras como "refinámos o modelo"), e guardar `reference_version` para poder recalcular depois;
- fixar `tol`, `k` e os pesos por golpe com o professor, usando várias execuções boas e más do mesmo golpe;
- definir o que é uma nota 100 (execução de referência) e o que é 0, para o gráfico não variar por ruído.

## Fases

| # | Fase | Depende de | Estado |
|---|---|---|---|
| 0 | Medir custo por análise | | em curso |
| 1 | Contrato da API de análises + notas estruturadas + keypoints | 0 | a fazer |
| 2 | PWA: instalação, câmera guiada, upload, estado | 1 | pode começar já (contra contrato mockado) |
| 3 | Contas (Google) + histórico | | login Google feito; falta histórico com notas |
| 4 | Web Push "análise pronta" | 2 | a fazer |
| 5 | Carteira de bolas | 3 | a fazer |
| 6 | Gráfico de evolução por golpe e geral | 1, 3 | a fazer |
| 7 | Região, preços e Stripe | 5 | a fazer |
| 8 | Plano Evolução + "próximo foco" | 6, 7 | depois |
| 9 | App nativo iOS/Android | retenção validada | depois |

Biblioteca de referência com professor: sem data, bloqueia a calibração das notas (não o contrato).

## Fase 1: contrato da API

- `POST /analyses` devolve `{analysis_id, status: "uploading"}` com URL de upload pré-assinada (multipart
  e retomável). Substitui o upload em streaming pela API previsto no backlog #1.
- `GET /analyses/{id}` devolve `status`: uploading, queued, processing, completed, rejected, failed.
- `completed`: `movement`, `scores` (ver acima), `positives[]`, `improvements[]`, `reference_version`,
  `analysis_version`.
- `rejected`: motivo legível (fora de enquadramento, corpo cortado, pouca luz) e a bola devolvida.
- Golpes suportados hoje: serve, forehand, backhand, volley, smash. Bandeja e víbora entram quando houver
  referência. Cada golpe tem o ângulo de câmera esperado numa tabela.
- Partes podem ser mockadas; o contrato não.

## Fase 2: PWA

- `manifest.webmanifest` e service worker que guarda só a casca (nunca vídeo nem relatório).
- Instalação: botão no Android/Chrome; no iOS, instruções "Partilhar, Adicionar ao ecrã principal"
  (necessário para push).
- **Câmera guiada:** `getUserMedia` (traseira, 720p, 30 fps) e `MediaRecorder`, com silhueta e zona do
  golpe. Validação no aparelho com MediaPipe Pose Landmarker (corpo inteiro, distância, centrado), nível do
  telefone com DeviceOrientation (pede permissão no iOS) e duração máxima por golpe (8 s).
  Alternativa: `<input type="file" accept="video/*" capture="environment">`.
- Formatos: Safari grava MP4/H.264 e Chrome Android grava WebM; o backend normaliza com ffmpeg.
- Upload direto para o S3 (pré-assinada, multipart), com progresso e retomada. Com o ecrã bloqueado o envio
  para (sobretudo no iOS): avisar "mantém aberto até enviar, depois podes fechar".
- Estado por polling ou SSE em `GET /analyses/{id}`.

## Fase 4: Web Push

- VAPID, sem contas Apple/Google. No iOS só com o app instalado.
- Sem conta, a subscrição fica ligada à análise e é apagada depois do aviso.
- Pedir permissão só depois do primeiro envio, nunca ao abrir. Só avisos com valor: análise pronta, falha
  com bola devolvida; progresso ("+8 no forehand") mais tarde.

## Métricas-chave

- Análises por cliente por mês (principal).
- Taxa de vídeos rejeitados (meta: descer dos 15% estimados com a câmera guiada).
- Instalações da PWA e aceitação de push.
- Funil do plano de implementação da monetização.
