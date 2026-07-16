# 0018. The website uses the Hextra theme, whole-site

- Status: accepted
- Date: 2026-07-16
- Deciders: mikluko

## Context

The `website` repo ([0011](0011-web-and-content-repos.md)) renders the landing
page, `/docs/` (mounted from `docs`), `/spec/` (mounted from `spec`), and the
vanity module-root pages ([0016](0016-vanity-module-roots.md)). Picking its
presentation layer is bounded by standing constraints:

- **No Node toolchain** — the site builds with the Hugo binary alone.
- Content arrives as **Hugo Modules**, so the theme must be pinnable and
  composable the same way.
- The audience is developers: dark mode, first-class code blocks, offline
  search, and **multi-language code tabs** (Go/Python side by side) matter.
- Docs are versioned by path per spec revision, which needs a version-switcher
  slot no theme ships natively.
- valiss has no visual identity yet; the theme's default look must be credible
  without design investment.

## Decision

- Use **Hextra** for the entire site — landing, docs, spec, and any future
  sections — pinned as a Hugo Module in the website's `go.mod`.
- **Customization discipline**, in escalating order: `hugo.toml` config first;
  then CSS (Hextra's hue variables and its `custom.css` hook — its Tailwind is
  precompiled, so styling stays plain CSS and no Node enters the build); then
  **shadowed templates** (same-path overrides in `layouts/`) only when config
  and CSS cannot express the need. Every shadowed template is listed in the
  website README — the inventory is the upgrade-friction budget.
- The spec-revision **version switcher** is our own partial injected through
  the theme's partial hooks, not a theme fork.
- The theme is never forked; upgrades are a module version bump reviewed
  against the override inventory.

## Consequences

- One coherent theme across landing and docs; the landing page does not look
  like a manual, and the docs get search, tabs, and dark mode for free.
- The no-Node build constraint survives; CI builds with the plain Hugo binary.
- Accepting Hextra's opinionated look until a visual identity exists; identity
  lands later as CSS variables and a logo, not a theme change.
- Divergence risk is bounded and visible: the shadowed-template inventory is
  the whole cost of upgrades.

## Alternatives considered

- **relearn (docs) + custom landing layout** — deep manual-style navigation,
  but the landing needs custom work anyway and the site ends up with two
  visual systems to keep coherent.
- **Geekdoc** — pleasantly minimal, but weaker search and no tabs; batteries
  we would end up building.
- **Docsy** — rejected earlier already: drags in a Node/PostCSS pipeline.
- **Fully custom theme** — maximal control, unjustified before a visual
  identity exists; revisit only if Hextra's structure ever fights the content.
