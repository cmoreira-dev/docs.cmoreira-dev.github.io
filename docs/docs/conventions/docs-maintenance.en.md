# Documentation upkeep

How to keep this site current as part of the work, not after it.

!!! note "Source of truth"
    Short rule in the root `CLAUDE.md` (session protocol). The full procedure is here.

## When to update

After any **functional** change (endpoint, contract, environment variable, route, infrastructure,
deploy process, architecture decision) and when closing or discovering a product open item. The update
goes in the same piece of work as the change, ideally in the same window of PRs.

## Which page

| Pillar | Pages under `docs/docs/` |
|---|---|
| Cluster, networking, secrets | `architecture/*` |
| IaC | `iac/*` |
| GitOps, addons, Burrito, Renovate, generic chart | `gitops/*`, `kubernetes/generic-app-chart` |
| CI/CD, registry, Argo CD | `cicd/*`, `argocd` |
| Repo conventions and workflow | `repos`, `conventions/*` |
| teupadel.com | `apps/teupadel`, `apps/teupadel-{api,ui,processor}`, `products/teupadel-{backlog,brand,email-ses}` |
| Sara | `apps/sara`, `apps/sara-{api,ui,backlog}` |

## How

1. Edit the Portuguese page (`x.md`) and its English twin (`x.en.md`) with the same structure.
2. New page: add it to `nav:` in `docs/mkdocs.yml` and its title translation under
   `plugins.i18n.languages[en].nav_translations`.
3. Every page starts with a "Source of truth" note saying what feeds it.
4. Validate: `cd docs && ../venv/bin/mkdocs build --strict` (fails on broken links or missing nav entries).
5. Branch + PR in `docs.cmoreira-dev.github.io`; the `deploy-docs.yml` workflow publishes to `docs.cmoreira.dev`.

## Drafting with the local agent

`draft_docs_update` reads a repo diff and the current page and returns the sections that need to change
as proposed markdown. It writes nothing: review and apply with Edit. Details in [Local agents](local-agents.md).

## Backlog pages

`products/teupadel-backlog` and `apps/sara-backlog` are the single source of open items and status.
Update the item in the same piece of work that resolves or creates it, with the date. Do not create
loose status files in product folders (they are not repositories and are not versioned).

## What does not go into `CLAUDE.md` files

Env var tables, JSON contracts, directory trees, deploy details and incident history live here. A repo's
`CLAUDE.md` keeps only commands, gotchas with the reason, and a pointer to the page.
