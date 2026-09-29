# teupadel.com — Processor de pose (api.ia.pose-estimation)

!!! warning "Fonte de verdade: código do repo; atualizar aqui em qualquer mudança funcional"
    Esta página descreve `api.ia.pose-estimation` (o `padel-processor`). Se o código e esta
    página divergirem, o código manda. Depois de qualquer mudança funcional, atualizar esta
    página **e** a versão EN (`teupadel-processor.en.md`) no mesmo trabalho. As regras curtas de
    trabalho ficam no `CLAUDE.md` do repo.

Serviço de pose estimation em **GPU**: recebe um vídeo, extrai a pose do jogador frame a frame
(YOLOv8n-pose via ONNX Runtime), calcula features biomecânicas, segmenta os golpes e compara cada
fase com uma biblioteca de referência por DTW. Só a [API](teupadel-api.md) lhe fala; a visão
geral do produto está em [teupadel.com](teupadel.md).

Corre no namespace `teupadel` (Service ClusterIP `teupadel-processor:8000`, sem HTTPRoute), no nó
`proxmox-k8s-gpu-worker-1` (RTX 3060 com time-slicing de 2 slots, partilhada com o Ollama). Migrou do
LXC `192.168.1.20` para o cluster em 2026-08.

## Contrato

### GET /health

`{"status": "ok"}`. O processamento corre em `asyncio.to_thread`, para o `/health` responder durante
uma análise (senão o readiness probe tiraria o pod do Service).

### POST /analyse

`multipart/form-data`: `file` (`mp4`/`mov`/`avi`/`mkv`, máx. 100 MB) e query `fps` (inteiro 1-10, default 2)
e `movement` opcional (`serve`/`forehand`/`backhand`/`volley`/`smash`). Erros: `413` (vídeo grande), `400`
(formato, `fps` fora de 1-10, `movement` inválido, `Content-Length` inválido), `500` (falha ao processar).

Se `movement` for dado, é usado diretamente na comparação (`movement_source: "user_provided"`). Sem ele,
o serviço tenta detetar o golpe (`movement_source: "auto_detected"` + `movement_warning`).

!!! warning "Deteção automática de golpe é experimental"
    `segmentation._classify_movement` foi validada contra os 5 vídeos de referência e errou a maior parte
    das vezes (7 repetições do mesmo forehand saíram como `volley`/`backhand`/`forehand` misturados). A
    deteção de *quando* há um swing (janela + impacto) é razoável; classificar *qual* golpe não é confiável.
    `classification_confidence` (0-1, `null` quando `user_provided`) gradua a mesma heurística, não a torna
    mais precisa; serve à API para esconder/hedgear um golpe.

Resposta (campos que a API consome; ver `README.md` do repo para o schema completo):

```json
{
  "total_frames": 90,
  "frames_with_pose": 74,
  "fps_extracted": 1.5,
  "processing_time_s": 12.4,
  "frames": [{ "frame": 0, "timestamp_s": 0.0, "detected": true,
               "landmarks": { "nose": { "x": 0.5, "y": 0.2, "visibility": 0.99 } },
               "features": { "elbow_angle_right_deg": 92.1 } }],
  "analysis_gif_base64": "R0lGODlh...",
  "stroke_analysis": [{
    "movement": "forehand", "movement_source": "user_provided", "movement_warning": null,
    "classification_confidence": null,
    "start_s": 1.5, "impact_s": 2.2, "end_s": 3.0,
    "deviations": { "preparation": { "elbow_angle_right_deg": {
      "deviation_avg": 12.4, "deviation_max": 30.1, "dtw_distance": 1.8,
      "n_points_client": 6, "n_points_reference": 78 } } }
  }]
}
```

- **Landmarks:** 17 pontos (YOLOv8n-pose, não 33 do MediaPipe): `nose`, `left/right_eye`, `left/right_ear`,
  `left/right_shoulder`, `left/right_elbow`, `left/right_wrist`, `left/right_hip`, `left/right_knee`,
  `left/right_ankle`. Coordenadas normalizadas [0,1] contra o canvas 640x640 com letterbox do modelo (não
  contra o frame original); o downscale `FRAME_MAX_DIM` é transparente para o output. Frame sem pose:
  `detected: false`, `landmarks: {}`, `features: null`.
- **`features`** (só em frames com pose, aditivo, não substitui `landmarks`): ângulos de cotovelo e joelho,
  rotação ombro-quadril, altura do pulso relativa ao ombro, velocidade/aceleração da anca (calculadas entre
  frames detetados consecutivos via `timestamp_s`).
- **`stroke_analysis`** é o que o prompt da API consome. Cada golpe traz `deviations` por fase
  (`preparation`, `impact`, `follow_through`) e por feature, com desvio médio/máximo e distância DTW.
- **`analysis_gif_base64`:** o GIF anotado ainda é gerado (best-effort), mas a API não o usa: a UI desenha o
  esqueleto a partir de `pose_frames`.
- **Seleção de pose:** `postprocess()` aplica NMS (`nms.py`) e ordena candidatos por (confiança, área);
  `select_primary_pose()` toma sempre o primeiro. Múltiplas pessoas está fora de âmbito.

## Pipeline

1. `extract_frames`: `ffmpeg` (`fps` efetivo, downscale a `FRAME_MAX_DIM`). O `fps` efetivo baixa em vídeos
   longos para o total ficar em `MAX_FRAMES`.
2. `yolo.inference`: ONNX Runtime, providers `["CUDAExecutionProvider", "CPUExecutionProvider"]` (GPU com
   fallback CPU; `get_session()` filtra pelos disponíveis).
3. `feature_extraction` (`features.py`).
4. `stroke_segmentation` (`segmentation.py`): janelas de swing por velocidade do pulso, com limiar relativo
   ao próprio vídeo; o pico é o proxy do impacto.
5. `dtw_compare.py` (`dtaidistance`): alinha feature a feature contra o clip de referência do mesmo
   golpe/fase e devolve os desvios.

Cada etapa é um span OTel (`extract_frames`, `yolo.inference`, `feature_extraction`, `stroke_segmentation`).

!!! note "Biblioteca de referência"
    `reference_library/data.json` tem **1 clip por golpe/fase** (5 golpes x 3 fases = 15 combinações),
    extraídos em 2026-09-26 de vídeos de coaching do YouTube (ver `source` de cada clip). Com 1 clip o desvio
    reflete a diferença contra esse clip, não uma média robusta: tratar os números como indicativos até haver
    3-5 clips por combinação, com jogadores diferentes. Adicionar clips:
    `python reference_library.py --video X --movement serve --phase preparation --clip-id serve_001`.

## Variáveis de ambiente

| Var | Default | Descrição |
|---|---|---|
| `MODEL_PATH` | `./yolov8n-pose.onnx` (imagem: `/app/yolov8n-pose.onnx`) | Modelo ONNX (~13 MB, embutido na imagem) |
| `FRAME_MAX_DIM` | `1280` | Teto da maior dimensão dos frames extraídos |
| `MAX_FRAMES` | `150` | Teto de frames processados |
| `OTEL_EXPORTER_OTLP_ENDPOINT` (+ `_PROTOCOL`, `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_TRACES_SAMPLER`, `OTEL_SDK_DISABLED`) | (desligado) | Telemetria; sem endpoint é no-op. Produção usa o Alloy (`:4318`), serviço `teupadel-processor`. |

## Repo

```
api.ia.pose-estimation/
├── main.py               FastAPI: GET /health, POST /analyse
├── processor.py          ffmpeg + ONNX Runtime + orquestração (segmentação/DTW), GIF
├── nms.py                NMS em Python puro (testável sem cv2/numpy/onnxruntime)
├── features.py           features biomecânicas por frame
├── segmentation.py       janelas de swing, impacto, classificação (experimental)
├── dtw_compare.py        comparação DTW contra a biblioteca de referência
├── reference_library.py  schema e CLI da biblioteca; reference_library/data.json
├── telemetry.py          OTel, baggage, formatter JSON
├── yolov8n-pose.onnx     modelo embutido na imagem
├── tests/                test_nms.py, test_segmentation.py
├── openapi.yaml · requirements.txt · requirements-dev.txt · Dockerfile
└── .github/workflows/    build-push.yml (ECR, linux/amd64)
```

## Dependências que se movem juntas

!!! danger "onnxruntime-gpu, tag nvidia/cuda, numpy e opencv-python-headless movem-se JUNTOS"
    Nunca mergear PRs isolados do Renovate para estas dependências. Validar antes com
    `pip install --dry-run` para cp310/manylinux e um smoke test em GPU.

- `onnxruntime-gpu==1.19.2` casa com CUDA 12.4 / cuDNN 9 da imagem base
  `nvidia/cuda:12.4.1-cudnn-runtime-ubuntu22.04`; Ubuntu 22.04 traz Python 3.10.
- `numpy>=1.26,<2.0` e `opencv-python-headless>=4.9.0,<5`: o OpenCV 5 exige NumPy 2.
- Em 2026-09-29 PRs do Renovate bumparam-nas isoladamente e quebraram o build (`onnxruntime-gpu` 1.30.0
  não existe para py3.10; OpenCV 5 precisa de NumPy 2).

## Deploy

- **Build:** push para `main` (ou tag `v*`) dispara `build-push.yml` (workflow reutilizável do org), que
  publica no ECR `teupadel/processor` via OIDC (`iac.homelab-live-infra`, `github_repos`) com
  `platforms: linux/amd64`: **nunca arm64**, CUDA/onnxruntime-gpu são amd64. Ver
  [Build & Registry](../cicd/build-registry.md). Não há CI de PR nem testes para `processor.py` /
  `dtw_compare.py` (precisam de cv2/onnxruntime/dtaidistance reais).
- **GitOps:** `gitops.teupadel.com/helm/processor` (wrapper do `generic-app`), Application `teupadel-processor`
  com auto-sync; o argocd-image-updater grava a tag (7 hex) mais recente do ECR no `values.yaml`.
- **Runtime:** `runtimeClassName: nvidia`, `nodeSelector nvidia.com/gpu.present=true`, `nvidia.com/gpu: 1`
  (limits); requests 500m/1Gi, limits 3 CPU/4Gi. `strategy: Recreate`: só há 2 slots de GPU (1 é do Ollama), um
  `RollingUpdate` criaria um segundo pod que não agendaria e o rollout ficaria preso. Probe de startup tolerante
  (até 30 x 5 s: CUDA + modelo no 1.º arranque), readiness com timeout de 5 s. Utilizador não-root UID 10001
  (numérico, por causa do `runAsNonRoot`), porta 8000, `securityContext` restricted.
- **Rede:** sem HTTPRoute; a `NetworkPolicy` `teupadel-processor` só admite a API (porta 8000), mas o CNI atual
  (flannel) não aplica NetworkPolicy. Não expor publicamente: o serviço não tem autenticação.
- O lado da API aponta para ele por `PROCESSOR_URL`
  (`http://teupadel-processor.teupadel.svc.cluster.local:8000`); a API espera até 180 s por análise.

## Testes

```bash
pip install -r requirements.txt          # onnxruntime-gpu; sem GPU cai para CPU
uvicorn main:app --reload --port 8000
curl -F "file=@swing.mp4" "http://localhost:8000/analyse?fps=2&movement=forehand"

pip install -r requirements-dev.txt
pytest tests/ -v                         # nms.py + segmentation.py; sem GPU/cv2/onnxruntime
```

## Pendências e itens a confirmar

- Fila durável em S3 (vídeo persistido em vez de só em memória na API): planeada, **não implementada**.
- Devolver `orig_w`/`orig_h` (ou normalizar no processor) para o overlay do esqueleto ficar pixel-perfect
  em vídeos não quadrados; exige coordenar com a API.
- Mais clips na biblioteca de referência; deteção/seleção de múltiplas pessoas.

!!! note "confirmar"
    - Estado atual do time-slicing da GPU (2 slots) e a partilha com o Ollama: vêm dos comentários do
      `values.yaml` do GitOps e do `nvidia-device-plugin` em `gitops.core-addons`, não verificados no cluster.
    - Benchmark citado no GitOps (150 frames 1080p em ~10 s na GPU, contra ~87 s no LXC/CPU): não reproduzido aqui.
