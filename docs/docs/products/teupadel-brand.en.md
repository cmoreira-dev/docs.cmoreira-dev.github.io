# teupadel: brand and visual identity

!!! info "Source of truth / update here"
    This page merges and replaces `contexto-identidade-visual.md` and the `teupadel-brand-context` skill,
    corrected with the real state of the UI (`ui.ia.teupadel.com`: `src/index.css`, `public/brand/`,
    `src/fonts/`). When in doubt, the UI code wins. Update here (PT + EN) when the brand changes.
    Implementation: [teupadel UI](../apps/teupadel-ui.en.md).

## What it is

teupadel uses AI to analyse a padel player's movement from a video. The user films a play with a phone,
uploads it and within seconds gets a report with strengths, points to improve and practical suggestions per
stroke: the feedback that usually only a coach gave.

Anchor line (hero): "A tua análise de padel, feita por IA." (PT-PT; the site is translated per locale).
Eyebrow: "1st analysis free". The product **does not position itself as a coach replacement**, but as an
accessible starting point between sessions: a tone of confidence and technical competence, neither clinical
nor pretentious.

**Current commercial model:** login is required to analyse; each analysis costs 1 bola; a new account gets 3
welcome bolas and referrals give +1 to each side; `/pricing` shows bola plans per region (BR and EU), still
without checkout. The brand must not promise "free, unlimited".

## Audience and markets

Amateur padel players without regular access to a coach. Three markets via URL prefix: `/pt-pt`, `/pt-br`
and `/en`. The identity must work in all three (avoid PT-only or BR-only cultural references).

## How it works (for visual storytelling)

1. Video upload (phone, no equipment).
2. Computer vision extracts pose frame by frame (17 points: shoulders, elbows, wrists, hips, knees, ankles,
   etc.).
3. An LLM turns the data into a natural-language report.
4. The user sees **their own video with the detected skeleton drawn live on top** (canvas overlay,
   `PoseCanvasOverlay`) and stroke-moment thumbnails, with "Watch in video" links. There is no GIF.

The skeleton (lines and nodes joining joints) is the product's most distinctive element; the identity
converses with it (lines, nodes, trajectories) instead of relying only on ball and racket. Strokes: serve,
forehand, backhand, volley, smash and ready position.

## Name and tone

- The name is always spelled **teupadel**, all lowercase, even at the start of a sentence or in titles.
- Tone: approachable, encouraging, technically credible, not intimidating. Court language: reports contain
  no degrees, angles or centimetres.
- **Privacy as a value:** the video is never stored (only the report JSON), there is a cookie banner and a
  privacy page. Convey "trustworthy" and "transparent", not "surveillance".
- **Always-visible disclaimer** on reports: it is AI, may be wrong, does not replace a coach. Humble tone.
- Marketing site and tool in the same app, on the `teupadel.com` domain.

## Current visual identity

**Single light theme**; there is no dark theme or toggle. Tokens in `src/index.css` (Portuguese names).

| Token | Value | Use |
|---|---|---|
| `--quadra` | `#2B5BFF` | Primary blue: links, secondary buttons, logo, focus |
| `--bola` | `#C8FF3D` | Lime green: primary CTA (`tp-btn-principal`), skeleton (`--esqueleto`) |
| `--noite` / `--on-noite` | `#0E1220` / `#FFFFFF` | Fixed dark bands (hero, footer, banners) |
| `--tinta` / `--tinta-suave` | `#0E1220` / `#525B70` | Text |
| `--superficie` / `--superficie-elevada` | `#F6F7FA` / `#FFFFFF` | Backgrounds |
| `--vidro` / `--linha` / `--borda` | `#EDF0FF` / `#DCE1EC` / `#7D869A` | Highlights, dividers, borders |
| `--positivo` / `--melhorar` / `--erro` | `#0A7266` / `#A84B06` / `#BE2F28` | Strengths, to improve, errors (each with a `-suave` variant) |
| `--selo` / `--vidro-escura` | `#3ED0C9` / `#1A2338` | Accents inside dark cards |

Spacing `--space-1..20` (4 to 80 px), radii `--radius-s/m/l` (6/12/24 px) and `--radius-bola` (circle),
shadow `--sombra-cartao`. Shared classes: `tp-btn` (`principal`, `secundario`, `quadra`, `noite`), `tp-tag`,
`tp-card`, `tp-rotulo`.

**Typography** (self-hosted, `next/font/local`, OFL licence): Unbounded 500-800 (display, logo, titles),
Figtree 400/500/700 (body), JetBrains Mono 500/600 (data and timestamps).

**Assets** in `public/brand/` (from the brand design system): `logo-horizontal.svg` and
`logo-horizontal-negativo.svg`, `simbolo.svg` (vertical bar plus dots on an ascending diagonal, the
trajectory), `selo-quadra.svg` (blue square with court lines and a lime ball, also the `icon.svg`),
`raquete.svg` / `raquete-branca.svg` / `raquete-limao.svg` and `favicon-180.png`. Do not reintroduce the
text "TeuPadel" or the old icon in place of the real logo.

## Still open

- Illustrations and dedicated iconography for each stroke.
- Written visual-tone guidelines (currently implicit in the copy and tokens).
- Dark version of the design system: for when the site gets a theme selector.

!!! warning "History / to confirm"
    These claims came from the old documents and are **not verifiable** in the current code; do not use
    them without confirming with the product owner:

    - That telemetry is "opt-in and anonymised" as described. There is a consent banner (`CookieConsent`)
      and `/cookies` lists `tp_consent`/`tp_vid`/`tp_sid`, but the exact behaviour before consent was not
      re-verified for this page.
    - That the product is "free, no paywall" (true before bolas; today only the 1st analysis is free).

    The dark palette `#0a1018` with orange `#ffa53e`, the Oswald + Inter typography, the orange icon with
    four dots, the absence of a logotype and the "GIF of the video with skeleton" were from the initial
    prototype and have been replaced; they are no longer a reference.
