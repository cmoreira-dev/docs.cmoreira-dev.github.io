# Sara: backlog and status

!!! info "Source of truth: the repo code; update here on any functional change"
    Status and open items for the Sara product (repos `api.ia.local-sara`,
    `ui.ia.local-sara`, `gitops.local-sara`). Components: [API](sara-api.md),
    [UI](sara-ui.md); overview in [Sara](sara.md).

!!! tip "How to maintain this page"
    Update items **in the same piece of work** that changes them (close, move, create). Every
    update carries a date (`YYYY-MM-DD`). A finished item moves to "Done" with its date, it is
    not silently deleted. An item that becomes a permanent decision migrates to the
    component page.

Initial survey on 2026-09-29, from the repos' old `CLAUDE.md` files, the code and the gitops
values.

## Open

| # | Item | Repo | Detail | Updated |
|---|---|---|---|---|
| 1 | Replicate Sara's accounts and add `/sara/*` to the External Secrets IAM policy | `iac.homelab-live-infra` / `gitops.core-addons` | Sara uses no `ExternalSecret` today. Once there is a secret (e.g. accounts/credentials), the External Secrets Operator IAM policy needs the `/sara/*` prefix (currently path-scoped: `homelab/*` and `teupadel/*`, see [Secrets](../architecture/secrets.md)) and the `gitops.local-sara` `ExternalSecret` should reference `/homelab/sara/...` or `/sara/...` per the chosen convention. Origin: root-repo notes. | 2026-09-29 |
| 2 | Validate Next 16 / React 19 / Node 24 in production | `ui.ia.local-sara` | Renovate bumps merged on `origin/main` on 2026-09-29 (only `Dockerfile` and `package.json` changed). Confirm the standalone build, non-root runtime (`USER 1000`, port 3000), auto-scroll and navigation in the cluster. **2026-09-29:** build under Next 16.3.7/React 19.3 validated locally (`npm ci`, `npm run build`, the standalone server answers `/sara` and `/sara/health` with 200; `npm audit` 0). Still to validate in production. | 2026-09-29 |
| 4 | Clean up remaining Renovate branches | `ui.ia.local-sara` | `renovate/major-nextjs-monorepo`, `major-react-monorepo` and `node-24.x` still exist on the remote and their tips are based on the old `main` (no non-root Dockerfile: `PORT=80`, no `USER`). The PRs were merged (the merge preserved non-root), but deleting the branches avoids reusing them by mistake. | 2026-09-29 |
| 5 | Local checkouts behind `origin/main` | `ui.ia.local-sara`, `api.ia.local-sara`, `gitops.local-sara` | UI (Next 16/React 19/Node 24), API (Python 3.14) and gitops (image tags) have unpulled remote commits; `git pull` before working. | 2026-09-29 |
| 6 | No lockfile in the UI | `ui.ia.local-sara` | No `package-lock.json`; the Dockerfile uses `npm install`: non-reproducible builds. Consider committing the lockfile and using `npm ci`. **2026-09-29:** lockfile added in PR `ui.ia.local-sara#10` (awaiting merge); using `npm ci` in the Dockerfile stays a future decision. | 2026-09-29 |
| 7 | `requirements.txt` without exact pins | `api.ia.local-sara` | Only `>=`; builds may change with new versions. Renovate has nothing to bump. | 2026-09-29 |
| 8 | Unbounded scraper cache | `api.ia.local-sara` | `_cache` only expires on read and never evicts; it grows in long-lived processes, per replica. Acceptable for two people; revisit if usage grows. | 2026-09-29 |
| 9 | Search by a single ambiguous word | `api.ia.local-sara` | "yellow" resolves to the homonymous artist. A real fix needs a search index/API (Google Custom Search `site:cifraclub.com.br` or the sitemap). Accepted limitation; workaround is pasting the link. | 2026-09-29 |
| 10 | Sync playlist across devices | `ui.ia.local-sara` | Currently `localStorage` only. Only makes sense if Sara and her friend need it; would require a server-side store and auth. | 2026-09-29 |
| 11 | Real beat sync in auto-scroll | `ui.ia.local-sara` | `PX_PER_BEAT = 16` is a tuned constant; Cifra Club exposes no per-line timing. Out of scope. | 2026-09-29 |
| 12 | Confirm pod probes | `gitops.local-sara` | The API and UI `values.yaml` define no liveness/readiness; confirm the `generic-app` 0.7.0 default (the Dockerfile `HEALTHCHECK` does not apply in Kubernetes). | 2026-09-29 |
| 13 | Does playlist auto-advance resume scrolling? | `ui.ia.local-sara` | The loop ends at the bottom and `playing` stays `true`; `useAutoScroll` only restarts if `playing` changes. If Next remounts the page when `[artist]/[song]` changes it works; confirm in the browser. Also: `simplified` may persist across songs without a simplified version (confirm). | 2026-09-29 |
| 14 | Repo docs (`docs/index.md`) with a wrong claim | `ui.ia.local-sara` | Says the UI reaches the API through the in-cluster Service; actually the browser uses the public URL. Fix (files outside the scope of this reorganization). **2026-09-29:** fixed in PR `ui.ia.local-sara#9` (awaiting merge). | 2026-09-29 |
| 15 | API `README.md` with a test comment | `api.ia.local-sara` | Ends with the HTML comment "trivial change ... take 3" (leftover from the image-updater write-back test). Remove. **2026-09-29:** removed in PR `api.ia.local-sara#9` (awaiting merge). | 2026-09-29 |

## Done

| Item | Where | Date |
|---|---|---|
| Repos on GitHub, CI to ECR (`sara/api`, `sara/ui`) and Argo CD Applications live at `local.cmoreira.dev/sara` (the old `CLAUDE.md` files said "local repo not yet pushed to GitHub") | all three repos | verified 2026-09-29 |
| UI non-root (uid 1000, port 3000) and API with numeric `USER 10001` for the `restricted` profile of `generic-app` 0.7.0 | `ui.ia.local-sara`, `api.ia.local-sara` | before 2026-09-29 |
| Backstage onboarding (`catalog-info.yaml`, TechDocs, committed `openapi.yaml`) | all three repos | before 2026-09-29 |
| Renovate bumps: Next 16, React 19, Node 24 (UI), Python 3.14 (API) merged on `origin/main` | `ui.ia.local-sara`, `api.ia.local-sara` | 2026-09-29 (validation in item 2) |
| `docker-compose.yml`: UI port fixed to `3000:3000` (local file, not in a repo) | `sara/docker-compose.yml` | 2026-09-29 |

!!! warning "To confirm"
    - Item 1 comes from the root-repo notes; the exact SSM parameter path and IAM policy name
      were not verified here (there is no Sara `ExternalSecret` in the gitops).
    - "Sara's accounts to replicate": scope (which accounts/credentials and where to) is not
      documented in any file read; confirm with the owner.
