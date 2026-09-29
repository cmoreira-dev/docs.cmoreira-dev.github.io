# teupadel.com — Pose processor (api.ia.pose-estimation)

!!! warning "Source of truth: the repo code; update this page on any functional change"
    This page describes `api.ia.pose-estimation` (the `padel-processor`). If the code and this page
    disagree, the code wins. After any functional change, update this page **and** the PT version
    (`teupadel-processor.md`) in the same piece of work. The short working rules live in the repo's
    `CLAUDE.md`.

**GPU** pose-estimation service: it receives a video, extracts the player's pose frame by frame
(YOLOv8n-pose via ONNX Runtime), computes biomechanical features, segments the strokes and compares each
phase against a reference library using DTW. Only the [API](teupadel-api.md) talks to it; the product
overview is in [teupadel.com](teupadel.md).

It runs in the `teupadel` namespace (ClusterIP Service `teupadel-processor:8000`, no HTTPRoute), on node
`proxmox-k8s-gpu-worker-1` (RTX 3060 with 2-slot time-slicing, shared with Ollama). It moved from LXC
`192.168.1.20` to the cluster in 2026-08.

## Contract

### GET /health

`{"status": "ok"}`. Processing runs in `asyncio.to_thread`, so `/health` answers during an analysis
(otherwise the readiness probe would take the pod out of the Service).

### POST /analyse

`multipart/form-data`: `file` (`mp4`/`mov`/`avi`/`mkv`, max 100 MB) and query `fps` (integer 1-10,
default 2) and optional `movement` (`serve`/`forehand`/`backhand`/`volley`/`smash`). Errors: `413` (video
too large), `400` (format, `fps` outside 1-10, invalid `movement`, invalid `Content-Length`), `500`
(processing failure).

If `movement` is given, it is used directly in the comparison (`movement_source: "user_provided"`).
Without it, the service tries to detect the stroke (`movement_source: "auto_detected"` +
`movement_warning`).

!!! warning "Automatic stroke detection is experimental"
    `segmentation._classify_movement` was validated against the 5 reference videos and got most of them
    wrong (7 repetitions of the same forehand came out as a mix of `volley`/`backhand`/`forehand`).
    Detecting *when* a swing happens (window + impact) is reasonable; classifying *which* stroke is not
    reliable. `classification_confidence` (0-1, `null` when `user_provided`) grades the same heuristic and
    does not make it more accurate; the API uses it to hide/hedge a stroke.

Response (fields the API consumes; see the repo `README.md` for the full schema):

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

- **Landmarks:** 17 points (YOLOv8n-pose, not MediaPipe's 33): `nose`, `left/right_eye`, `left/right_ear`,
  `left/right_shoulder`, `left/right_elbow`, `left/right_wrist`, `left/right_hip`, `left/right_knee`,
  `left/right_ankle`. Coordinates are normalized [0,1] against the model's 640x640 letterboxed canvas (not
  the original frame); the `FRAME_MAX_DIM` downscale is transparent to the output. A frame without a pose:
  `detected: false`, `landmarks: {}`, `features: null`.
- **`features`** (only on frames with a pose, additive, does not replace `landmarks`): elbow and knee
  angles, shoulder-hip rotation, wrist height relative to the shoulder, hip velocity/acceleration
  (computed between consecutive detected frames via `timestamp_s`).
- **`stroke_analysis`** is what the API's prompt consumes. Each stroke carries `deviations` per phase
  (`preparation`, `impact`, `follow_through`) and per feature, with mean/max deviation and DTW distance.
- **`analysis_gif_base64`:** the annotated GIF is still generated (best-effort), but the API does not use
  it: the UI draws the skeleton from `pose_frames`.
- **Pose selection:** `postprocess()` applies NMS (`nms.py`) and sorts candidates by (confidence, area);
  `select_primary_pose()` always takes the first. Multiple people are out of scope.

## Pipeline

1. `extract_frames`: `ffmpeg` (effective `fps`, downscale to `FRAME_MAX_DIM`). The effective `fps` drops on
   long videos so the total stays within `MAX_FRAMES`.
2. `yolo.inference`: ONNX Runtime, providers `["CUDAExecutionProvider", "CPUExecutionProvider"]` (GPU with
   CPU fallback; `get_session()` filters to those available).
3. `feature_extraction` (`features.py`).
4. `stroke_segmentation` (`segmentation.py`): swing windows from wrist velocity, with a threshold relative
   to the video itself; the peak is the impact proxy.
5. `dtw_compare.py` (`dtaidistance`): aligns feature by feature against the reference clip of the same
   stroke/phase and returns the deviations.

Each stage is an OTel span (`extract_frames`, `yolo.inference`, `feature_extraction`,
`stroke_segmentation`).

!!! note "Reference library"
    `reference_library/data.json` has **1 clip per stroke/phase** (5 strokes x 3 phases = 15
    combinations), extracted on 2026-09-26 from YouTube coaching videos (see each clip's `source`). With one
    clip the deviation reflects the difference against that clip, not a robust average: treat the numbers as
    indicative until there are 3-5 clips per combination, with different players. To add clips:
    `python reference_library.py --video X --movement serve --phase preparation --clip-id serve_001`.

## Environment variables

| Var | Default | Description |
|---|---|---|
| `MODEL_PATH` | `./yolov8n-pose.onnx` (image: `/app/yolov8n-pose.onnx`) | ONNX model (~13 MB, embedded in the image) |
| `FRAME_MAX_DIM` | `1280` | Cap on the largest dimension of extracted frames |
| `MAX_FRAMES` | `150` | Cap on processed frames |
| `OTEL_EXPORTER_OTLP_ENDPOINT` (+ `_PROTOCOL`, `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_TRACES_SAMPLER`, `OTEL_SDK_DISABLED`) | (off) | Telemetry; no-op without an endpoint. Production uses Alloy (`:4318`), service `teupadel-processor`. |

## Repo

```
api.ia.pose-estimation/
├── main.py               FastAPI: GET /health, POST /analyse
├── processor.py          ffmpeg + ONNX Runtime + orchestration (segmentation/DTW), GIF
├── nms.py                pure-Python NMS (testable without cv2/numpy/onnxruntime)
├── features.py           per-frame biomechanical features
├── segmentation.py       swing windows, impact, classification (experimental)
├── dtw_compare.py        DTW comparison against the reference library
├── reference_library.py  library schema and CLI; reference_library/data.json
├── telemetry.py          OTel, baggage, JSON formatter
├── yolov8n-pose.onnx     model embedded in the image
├── tests/                test_nms.py, test_segmentation.py
├── openapi.yaml · requirements.txt · requirements-dev.txt · Dockerfile
└── .github/workflows/    build-push.yml (ECR, linux/amd64)
```

## Dependencies that move together

!!! danger "onnxruntime-gpu, the nvidia/cuda tag, numpy and opencv-python-headless move TOGETHER"
    Never merge isolated Renovate PRs for these dependencies. Validate first with `pip install --dry-run`
    for cp310/manylinux and a GPU smoke test.

- `onnxruntime-gpu==1.19.2` matches CUDA 12.4 / cuDNN 9 in the base image
  `nvidia/cuda:12.4.1-cudnn-runtime-ubuntu22.04`; Ubuntu 22.04 ships Python 3.10.
- `numpy>=1.26,<2.0` and `opencv-python-headless>=4.9.0,<5`: OpenCV 5 requires NumPy 2.
- On 2026-09-29 Renovate PRs bumped them independently and broke the build (`onnxruntime-gpu` 1.30.0 does
  not exist for py3.10; OpenCV 5 needs NumPy 2).

## Deploy

- **Build:** a push to `main` (or a `v*` tag) triggers `build-push.yml` (the org's reusable workflow), which
  publishes to ECR `teupadel/processor` via OIDC (`iac.homelab-live-infra`, `github_repos`) with
  `platforms: linux/amd64`: **never arm64**, CUDA/onnxruntime-gpu are amd64. See
  [Build & Registry](../cicd/build-registry.md). There is no PR CI and no tests for `processor.py` /
  `dtw_compare.py` (they need real cv2/onnxruntime/dtaidistance).
- **GitOps:** `gitops.teupadel.com/helm/processor` (wrapper of `generic-app`), Application
  `teupadel-processor` with auto-sync; argocd-image-updater writes the newest ECR tag (7 hex) to
  `values.yaml`.
- **Runtime:** `runtimeClassName: nvidia`, `nodeSelector nvidia.com/gpu.present=true`, `nvidia.com/gpu: 1`
  (limits); requests 500m/1Gi, limits 3 CPU/4Gi. `strategy: Recreate`: there are only 2 GPU slots (1 belongs
  to Ollama), so a `RollingUpdate` would create a second pod that cannot schedule and the rollout would hang.
  Tolerant startup probe (up to 30 x 5 s: CUDA + model on first start), readiness with a 5 s timeout.
  Non-root UID 10001 (numeric, because of `runAsNonRoot`), port 8000, restricted `securityContext`.
- **Network:** no HTTPRoute; the `teupadel-processor` `NetworkPolicy` only admits the API (port 8000), but the
  current CNI (flannel) does not enforce NetworkPolicy. Do not expose it publicly: the service has no
  authentication.
- The API side points at it via `PROCESSOR_URL` (`http://teupadel-processor.teupadel.svc.cluster.local:8000`);
  the API waits up to 180 s per analysis.

## Testing

```bash
pip install -r requirements.txt          # onnxruntime-gpu; without a GPU it falls back to CPU
uvicorn main:app --reload --port 8000
curl -F "file=@swing.mp4" "http://localhost:8000/analyse?fps=2&movement=forehand"

pip install -r requirements-dev.txt
pytest tests/ -v                         # nms.py + segmentation.py; no GPU/cv2/onnxruntime
```

## Backlog and items to confirm

- Durable S3 queue (video persisted instead of memory-only in the API): planned, **not implemented**.
- Return `orig_w`/`orig_h` (or normalize in the processor) so the skeleton overlay is pixel-perfect on
  non-square videos; needs coordination with the API.
- More reference-library clips; multi-person detection/selection.

!!! note "To confirm"
    - Current state of GPU time-slicing (2 slots) and sharing with Ollama: taken from the GitOps `values.yaml`
      comments and the `nvidia-device-plugin` in `gitops.core-addons`, not verified on the cluster.
    - Benchmark quoted in GitOps (150 1080p frames in ~10 s on GPU, versus ~87 s on the LXC/CPU): not
      reproduced here.
