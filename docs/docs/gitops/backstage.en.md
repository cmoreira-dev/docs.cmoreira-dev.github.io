# Backstage (`backstage.homelab`)

Internal developer portal, giving unified visibility over the org's
components and services (software catalog), built on
[Backstage](https://backstage.io/).

## Catalog registration

Several repositories already carry a `catalog-info.yaml`, registering
themselves in Backstage's catalog:

- Addons: `gitops.cnpg`, `gitops.echoserver`,
  `gitops.headlamp`, `gitops.template`
- Infra: `homelab-bootsrap-k3s`
- Apps: `gitops.local-sara`, `gitops.teupadel.com`

The annotation pattern (`type: website`, `lifecycle: lab`, tags
`kubernetes` / `homelab` / `infrastructure`, link to the corresponding
Application via the `argocd/app-name` annotation) is the same across every
registered repo — see [Repository Patterns](../repos.md) for the full
structure of a `gitops.<app>`.

## Persistence

The Backstage instance uses Postgres via the **CloudNativePG** operator
(`gitops.cnpg`), which provisions a dedicated database
(`kustomize/backstage/`) with a `PodMonitor` for observability.

## Deployment

Like any other of the cluster's own apps, Backstage is delivered via GitOps
— see [GitOps pattern](pattern.md).

## Visual theme (Apple restyle)

The Backstage frontend (`backstage.homelab`) uses a theme inspired by Apple's design system, taken from
the `DESIGN-apple.md` document (repo root). The document describes marketing pages; only the visual
language (colors, typography, radii, elevation, buttons) is applied to real Backstage patterns, not the
marketing layout.

- **Tokens** live in a single file, `packages/app/src/theme/appleTheme.ts` (no hex outside it),
  registered through the new frontend system's `ThemeBlueprint` (`createFrontendModule`) as `light` under
  `pluginId: 'app'` (overrides the default theme).
- **Self-hosted Inter** via `@fontsource/inter`, not Google Fonts: the app ships a hardened CSP
  (`app-config.yaml`) and an external `<link>` would require opening `style-src`/`font-src` and leak
  viewer IPs to Google. The stack leads with `system-ui, -apple-system`, so Apple devices use SF Pro.
- **No decorative gradients, no shadows**: depth through surface changes; the sidebar and the pressed
  state (`scale(0.95)`) are done with theme overrides, without editing components.
- **Verification so far is build-level** (`tsc --noEmit`, `yarn workspace app lint/build`); not yet
  confirmed visually in a browser.
