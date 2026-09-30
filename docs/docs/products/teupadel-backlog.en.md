# teupadel: backlog and status

!!! info "Source of truth / update here"
    This page is the single source for teupadel.com follow-ups and status. It replaces the product
    folder's `PENDENCIAS.md` (and the old accounts/e-mail, hardening, processor/GPU and telemetry
    plans, which have already been executed).

!!! tip "How to maintain this page"
    Update the status of items **in the same piece of work that changes them** (same PR or same session),
    and date every update (`YYYY-MM-DD`). Finished item: move it to "Closed" with the date and where it
    lives now (repo, PR, docs page). New item: add it to the right section with the next number.

Updated on **2026-09-30** (roadmap v1 in [teupadel-roadmap](teupadel-roadmap.md)). State: login (magic link + Google), bolas with ledger, referrals, async
analysis with history, anti-abuse (Phase 6), Turnstile, SES bounce worker, the queue's S3 bucket and
telemetry are **in production**. See [teupadel.com](../apps/teupadel.en.md).

## Product

- **1.** **Durable analysis queue (logic).** _Revised 2026-09-30: upload becomes a presigned-URL direct-to-S3 upload (roadmap, Phase 1) instead of streaming through the API._ The infra (bucket `cmoreira-dev-teupadel-analysis-uploads` and
  `teupadel-api` user permissions) already exists. Still missing in the API:
    - migration with status `queued`, `heartbeat_at`, `attempts`, `video_key`;
    - streaming (multipart) upload to S3, the trickiest part; it also takes the 100 MB video out of pod
      memory (currently requires 1Gi);
    - worker with `SELECT ... FOR UPDATE SKIP LOCKED`, heartbeat every 30 s, requeue if the heartbeat stops
      (> 2 min), max 2 attempts, then `failed` + bola refund;
    - delete the S3 video at job end; `ANALYSIS_UPLOADS_BUCKET` in gitops.

    **Done on 2026-09-30 (api.ia.teupadel.com, queue PR):** migration 0008, `POST /analyses` + presigned parts
    + `complete` + `GET /analyses/{id}`, worker with `SKIP LOCKED`, heartbeat, retry and refund, and video
    deletion. **Still missing:** turning it on (`ANALYSIS_UPLOADS_BUCKET` in gitops) and items 33 to 35.
- **33.** **CORS on the `cmoreira-dev-teupadel-analysis-uploads` bucket** (IaC `s3-analysis-uploads`): without
    `aws_s3_bucket_cors_configuration` (PUT and upload headers, origin `https://www.teupadel.com`) the browser
    cannot send the parts. Merging into `iac.homelab-live-infra` triggers the apply: only with confirmation.
    Then the UI CSP needs the S3 host in `connect-src`. _(open, 2026-09-30)_
- **34.** **Video bucket region:** it is in `us-east-1`, the roadmap asks for the EU (GDPR). Decide before
    turning it on: recreate in `eu-west-1` (the bucket is temporary, nothing to migrate). _(open, 2026-09-30)_
- **35.** **WebM:** Chrome Android records WebM and the processor only accepts mp4/mov/avi/mkv. Normalize with
    ffmpeg (processor or API) before the guided camera. _(open, 2026-09-30)_
- **29.** **Per-movement and overall scores + evolution chart** in "My account" (design in the
    [roadmap](teupadel-roadmap.md#scores-per-movement-and-overall)): additive migration with `movement`,
    `score`, `reference_version`, `analysis_version`; `GET /me/progress`.
    _(API done 2026-09-30, api.ia.teupadel.com#38; UI: chart + beta tag in PR; calibration with the coach still missing, see 31)_
- **30.** **PWA (roadmap Phase 2):** manifest, shell-only service worker and install button in PR
    (ui#40, 2026-09-30). Still missing: guided camera with MediaPipe (CSP and `Permissions-Policy:
    camera=(self)`) and presigned upload with resume (depends on the API contract, Phase 1). _(partial, 2026-09-30)_
- **31.** **Reference library with a coach:** today 5 YouTube clips, uncalibrated, license to review. Blocks
    score calibration (tolerances and weights). No date. _(open, 2026-09-30)_
- **32.** **`EmailSender` with a second provider** (Brevo, Scaleway TEM, Postmark or Resend) while the magic
    link is blocked by SES (see 6 and 7). _(open, 2026-09-30)_
- **2.** **Payment and plans.** `/pricing` and the waitlist exist; there is no checkout or bola purchase.
  Instrument the `payment_*` events (helpers ready) when payment exists.
- **3.** **UX:**
    - the chosen video is lost when the visitor is sent to login;
    - the loading screen does not say the user can leave and find the report in the history;
    - design still has to confirm the Turnstile widget.
- **26.** **Turnstile on `/login` under the new CSP: test in the browser** (DevTools, no CSP errors) and walk
  through `/analysis` end to end: Next 16, React 19 and Node 24 went to production without manual testing.
  _(open, 2026-09-29)_

## Legal and privacy

- **4.** **Report retention:** decide the period (`REPORT_RETENTION_DAYS`, currently off).
- **5.** **Privacy Policy and Terms** are still drafts, without legal review. The text must mention stored
  reports, the temporary S3 (once the queue exists), the cookie banner and the subprocessors (SES, Google,
  Cloudflare).

## E-mail (SES)

Runbook in [E-mail and SES](teupadel-email-ses.en.md).

- **6.** **Magic link never exercised in production:** needs an SES-verified address or a `@teupadel.com` one.
- **7.** **Leave the sandbox:** AWS denied it on 2026-09-29 (case 179054498900338). Reopen in 2 to 3 weeks,
  now with the bounce worker validated; describe the usage (transactional only, low volume, addresses of
  people requesting login, bounce and complaint handling). Ready-made text in the runbook appendix.
- **8.** **After approval:** validate SPF, DKIM and DMARC on a real e-mail (Gmail and Outlook); configure
  sending as `contato@` (SMTP in `/teupadel/ses/gmail-smtp`); raise DMARC from `none` to `quarantine` after
  2 to 4 weeks of clean reports.

## Observability

- **9.** **Alerts** (`gitops.monitoring` has no rules; no Grafana Cloud access from here): Alloy not ready /
  missing logs (`absent_over_time({namespace="teupadel", container="teupadel-api"}[10m])`); bounce rate > 2 %.
- **10.** **Dashboard automation:** import is manual. The copy in
  `gitops.monitoring/dashboards/teupadel-telemetria.json` lacks the "Authentication" and "Bolas and
  referrals" rows (the full version is `dashboard-teupadel-telemetria.json` in the product folder). Also
  import `dashboards/gpu.json`.
- **11.** **Alloy, medium term:** replace `loki.source.kubernetes` (one stream per pod against the Kubernetes
  API) with `loki.source.file` reading `/var/log/pods`. `alloy-worker` on the GPU node degrades over days
  and drops logs; deleting the pod fixes it, and PR #26 adds liveness.
- **12.** **Telemetry phase 2:** Web Vitals (LCP/CLS/INP) in the frontend; per-session funnel via TraceQL;
  dedicated tool (Umami/Plausible/PostHog) only if retention/cohort questions come up.

## Platform

- **13.** **`gitops.generic-app-chart`:** does not restart pods when a `Secret` changes (a manual `rollout
  restart` was needed for Turnstile). Needs secret checksums in the annotations.
- **14.** **Postgres backup** with a really tested restore.
- **15.** **Tests:** end to end (Playwright) with the SES simulator; full production test (sign-up,
  verification, login with both methods, deletion).
- **16.** **"GPU-only" GPU node:** postpone until there is a 2nd amd64 worker (then rebalance and taint).
- **27.** **Next 16: rename `src/middleware.js` to `proxy` and investigate dynamic pages.** The build warns
  that `middleware` is deprecated; pages are now rendered per request instead of prerendered.
  _(open, 2026-09-29)_
- **28.** **Backstage: peer dependency warnings** (`jsdom` 30 vs `^27` from `@backstage/cli`). The
  `yarn.lock` fix was merged (backstage.homelab#32); the warnings remain. _(open, 2026-09-29)_

## Security (audit findings)

**Closed on 2026-09-29:**

| # | Item | Status |
|---|---|---|
| 17 | CSP on the UI | UI#35, in production; `script-src` keeps `'unsafe-inline'` (prerendered pages). See [UI](../apps/teupadel-ui.en.md#security) |
| 19 | JSON-LD escaped | done |
| 20 (safe part) | `npm audit fix`, `/_next/image` off, `X-Powered-By` removed | done. Renovate merged today: Next 16, React 19, Node 24 (**no manual test**, see 26) and the PRs of the other repos |
| 21 | HTTPS | Cloudflare "Always Use HTTPS" and HSTS enabled |
| 22 | `security-smoke.sh` | run in production, all PASS |

**Still open:**

- **18.** **`NetworkPolicy` inert until a CNI enforces them (Cilium/Calico).** Merged (gitops#37) but without
  effect: the CNI is flannel. Changing CNI is its own project; afterwards restrict egress in a 2nd pass.
- **20.** **(remaining) pose-estimation:** CUDA 13, new onnxruntime, OpenCV 5 and NumPy 2 must return
  **together** and GPU-tested. The base was reverted to CUDA 12.4 / onnxruntime 1.19.2 / OpenCV 4
  (pose-estimation#22). Renovate PR api.ia.pose-estimation#16 (NumPy 2) is mergeable but **must not merge
  alone**.
- See also 26 (Turnstile under the CSP), 27 (Next 16) and 28 (Backstage).
- Re-run `./security-smoke.sh` after every relevant deploy.

## Go-live and Sara

- **23.** Google consent screen set to **"In production"**, with the `teupadel-prod` client.
- **24.** Test receiving `contato@`, `suporte@` and `privacidade@`.
- **25.** **Sara:** replicate accounts later; the External Secrets policy (IaC `iam-external-secrets`) needs
  the `/sara/*` prefix.
