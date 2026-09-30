# teupadel: product roadmap v1

Status: proposal approved on **2026-09-30**. Web/PWA first, native app later. It combines the "digital
coach in your pocket" vision with the monetization model (bolas, 5/15/40 packs, Evolution plan). Where they
conflict, the monetization decisions win. Open items and execution status live in the
[backlog](teupadel-backlog.md).

## Principles

1. **The video is temporary.** It lives in an S3 bucket only for asynchronous processing (the analysis
   keeps running if the person closes the browser after uploading). The backend deletes the video when the
   analysis ends, on success or failure; a 1-day lifecycle rule deletes whatever is left. The report and
   the **pose keypoints** are kept, so scores can be recomputed when the engine evolves.
    - EU bucket, private, encrypted, presigned-URL upload, **no versioning** (otherwise "delete" leaves
      copies). Rejected or failed videos are deleted too.
    - User-facing text: "Your video is deleted right after the analysis (24 h at most). We only keep the
      report." The retention period must be in the privacy policy (GDPR/LGPD), and keypoints count as
      personal data.
2. **The score is deterministic.** It comes from comparing the pose with the reference library; the LLM only
   explains. Each analysis stores `reference_version` and `analysis_version`.
3. **Web/PWA before a native app.** Validate retention (analyses per customer per month) before Flutter/React
   Native and App Store rules.
4. **One currency: bolas.** The ledger stores `provider` (stripe | app_store | play).

## Authentication without SES

AWS refused to take SES out of the sandbox and the **magic link stays blocked** until a new approval (see
[E-mail and SES](teupadel-email-ses.md)). Google login works and counts as a verified e-mail for the welcome
bolas and anti-abuse.

- E-mail sending goes behind an interface (`EmailSender`, already in `email_sender.py`), with SES and a
  second configurable provider (Brevo, Scaleway TEM, Postmark or Resend).
- New SES request in 2 to 3 weeks (transactional only, low volume, bounces via SNS).
- Sign in with Apple once there is an iOS app. Without e-mail there are no e-mail receipts; it does not
  block, since Stripe is in test mode.

## Scores: per movement and overall

Goal: each analysis produces a **movement score** and feeds an **overall score**, so "My account" can show
an evolution chart per movement and overall.

### From distance to score

`dtw_compare.py` already returns, per phase and per feature, `deviation_avg` and `dtw_distance` against the
reference. The score is computed in layers, all deterministic:

1. **Feature to 0-100.** Each feature has a tolerance `tol` (degrees, or canvas fraction for
   position/velocity): `score = 100 · max(0, 1 − deviation / (k · tol))`, with `k` set during calibration.
2. **Phase score.** Weighted mean of the phase's features (weights in a per-movement, per-phase table).
3. **Movement (analysis) score.** Weighted mean of the phases (preparation, contact, follow-through). With
   several strokes in the video, mean of the valid strokes.
4. **Overall score.** Per analysis, the movement score is also the overall score at that moment. On the
   "Overall" chart, each point is the **weighted mean of the latest score of each movement** over the last
   30 days, so the overall does not rise or fall just because the user switched movement.

Features without data (knee out of frame, for example) drop out of the mean and the weights renormalize; if
more than half of a phase's weight is missing, the phase gets no score (`null`) instead of an invented one.

### Stored structure

`reports.result` gets a structured block (the model's text is not part of the calculation):

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

For the chart, `movement`, `score`, `reference_version`, `analysis_version` and `finished_at` also go into
`reports` columns (additive migration), with an index on `(user_id, movement, finished_at)`. Keypoints
already live in `reports.result.pose_frames`, so scores can be recomputed later without reprocessing video.

### Chart in "My account"

- `GET /me/progress` returns the per-movement series (`movements`), `overall` and `version_breaks`; the UI filters.
- One line per movement plus the "Overall" line. A change of `reference_version` or `analysis_version` shows
  as a break in the chart, **or** old scores are recomputed from keypoints. Decision
  (2026-09-30): start with the visible break, with the message "We refined the model".
- A trend only shows with 2 or more analyses of the movement; before that, show the score alone.

### Calibration (depends on the coach)

Scores only mean something once the library has good reference videos. Today it is 5 YouTube clips (1 per
movement/phase), **indicative and uncalibrated**; their license also needs review before scores are
published. The final library will be filmed with a coach. Until then:

- show the score as **"beta"** (and explain the breaks as "we refined the model"), and store `reference_version` so it can be recomputed later;
- set `tol`, `k` and per-movement weights with the coach, using many good and bad executions of the same
  movement;
- define what a 100 is (a reference execution) and what a 0 is, so the chart does not move with noise.

## Phases

| # | Phase | Depends on | Status |
|---|---|---|---|
| 0 | Measure cost per analysis | | in progress |
| 1 | Analyses API contract + structured scores + keypoints | 0 | to do |
| 2 | PWA: install, guided camera, upload, status | 1 | can start now (against a mocked contract) |
| 3 | Accounts (Google) + history | | Google login done; history with scores missing |
| 4 | Web Push "analysis ready" | 2 | to do |
| 5 | Bolas wallet | 3 | to do |
| 6 | Evolution chart per movement and overall | 1, 3 | to do |
| 7 | Region, prices and Stripe | 5 | to do |
| 8 | Evolution plan + "next focus" | 6, 7 | later |
| 9 | Native iOS/Android app | retention validated | later |

Reference library with a coach: no date; it blocks score calibration (not the contract).

## Phase 1: API contract

- `POST /analyses` returns `{analysis_id, status: "uploading"}` with a presigned upload URL (multipart,
  resumable). Replaces the API-streamed upload planned in backlog #1.
- `GET /analyses/{id}` returns `status`: uploading, queued, processing, completed, rejected, failed.
- `completed`: `movement`, `scores` (see above), `positives[]`, `improvements[]`, `reference_version`,
  `analysis_version`.
- `rejected`: human-readable reason (out of frame, body cut off, low light) and the bola refunded.
- Movements supported today: serve, forehand, backhand, volley, smash. Bandeja and víbora come when there is
  a reference. Each movement has its expected camera angle in a table.
- Parts may be mocked; the contract may not.

## Phase 2: PWA

- `manifest.webmanifest` and a service worker that caches only the shell (never video or reports).
- Install: button on Android/Chrome; on iOS, instructions "Share, Add to Home Screen" (required for push).
- **Guided camera:** `getUserMedia` (rear, 720p, 30 fps) and `MediaRecorder`, with a silhouette and the
  movement's target zone. On-device validation with MediaPipe Pose Landmarker (full body, distance,
  centered), phone level via DeviceOrientation (asks permission on iOS) and a maximum duration per movement
  (8 s). Alternative: `<input type="file" accept="video/*" capture="environment">`.
- Formats: Safari records MP4/H.264 and Chrome Android records WebM; the backend normalizes with ffmpeg.
- Direct upload to S3 (presigned, multipart), with progress and resume. With the screen locked the upload
  stops (especially on iOS): warn "keep it open until it is sent, then you can close it".
- Status by polling or SSE on `GET /analyses/{id}`.

## Phase 4: Web Push

- VAPID, no Apple/Google accounts. On iOS only with the app installed.
- Without an account, the subscription is tied to the analysis and deleted after the notification.
- Ask for permission only after the first upload, never on open. Only valuable notifications: analysis
  ready, failure with bola refunded; progress ("+8 on forehand") later.

## Key metrics

- Analyses per customer per month (main).
- Rejected video rate (goal: drop from the estimated 15% with the guided camera).
- PWA installs and push acceptance.
- Monetization implementation-plan funnel.
