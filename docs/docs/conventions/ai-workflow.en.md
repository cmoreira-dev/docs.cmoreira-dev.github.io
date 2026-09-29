# AI workflow

How Claude Code sessions work in this workspace: where each kind of knowledge lives, what is read
at the start, what is updated at the end, and what is delegated to local agents to save tokens.

!!! note "Source of truth"
    The short rules that apply to every session live in the root `CLAUDE.md` of
    `cmoreira-dev/`. This page explains the why and the upkeep. Changed the workflow? Update both.

## Where each thing lives

| Kind of knowledge | Where | Why |
|---|---|---|
| Rules that apply to **every** session (how to work, guardrails, routing to local agents) | root `CLAUDE.md` | Always loaded; every line costs tokens in every session |
| Repo essentials (commands, gotchas with the reason, conventions, cross-repo coupling) | repo `CLAUDE.md` | Loaded when working in that repo; overrides the root in its scope |
| Architecture, API contracts, env vars, routes, deploy, runbooks | this site (`docs/docs/`) | Does not need to be live every session; read per pillar when needed |
| Open items and in-flight status | backlog pages (`products/teupadel-backlog`, `apps/sara-backlog`) | Single source; survives across sessions and is versioned |
| User preferences and decisions across sessions | Claude Code memory (`~/.claude/projects/.../memory`) | Not derivable from code or docs |
| Loose notes in a product folder (`teupadel.com/`, `sara/`) | nowhere: these folders are not repositories | Files there are not versioned and get lost |

## Session protocol

1. **Identify the pillar** of the task (table in the root `CLAUDE.md`).
2. **Read the pillar's docs page** before changing contracts, routes, infrastructure or deploy.
3. **Read the repo's `CLAUDE.md` and `README.md`** (the repo's overrides the root's).
4. **Delegate cheap operational work** to local agents (see [Local agents](local-agents.md)).
5. **When done**: update the docs page (PT and EN) and the backlog in the same piece of work.
   If no page exists, create it or record the gap explicitly.

## Delivery flow

Feature branch → PR → merge to `main` → GitHub Actions builds the image and pushes it to ECR (OIDC) →
argocd-image-updater bumps the tag in `gitops.<product>` → Argo CD syncs on its own. Most repos have
no PR CI (build on push to `main` only), so a local or Docker build and the tests are the validation
before merging. Details: [From push to cluster](../cicd/deploy-flow.md).

## Guardrails

- Never commit directly to `main` without saying so; prefer branch + PR.
- `terraform apply` / `terragrunt apply` and merges into `iac.homelab-live-infra` (which triggers its
  approval-gated apply) only with explicit confirmation.
- No `kubectl apply/patch/delete` to change application state: the change goes to the GitOps repo.
- Renovate major bumps are separate projects: test them, do not batch-merge.
- Coupled dependencies (e.g. the processor's `onnxruntime-gpu` + `nvidia/cuda` tag + `numpy` + `opencv`)
  move together.

## Delegation and token savings

Collecting and reading raw output (logs, `kubectl`, Actions and Argo CD status) and drafting docs and
simple code go to the local model via MCP; only a short structured result comes back into Claude's
context. The local model can be wrong and cannot act: anything decision-critical is verified with a
direct command. Tools, limits and configuration: [Local agents](local-agents.md).
