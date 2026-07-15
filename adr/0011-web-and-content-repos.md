# 0011. Web presentation and content repositories

- Status: accepted
- Date: 2026-07-15
- Deciders: mikluko

## Context

The project publishes a public site at `valiss.dev`: a landing page, user
documentation, the rendered specification, and more over time. That site also
already serves the Go vanity import metadata (ADR 0003). The question is how to
split presentation from content across repositories so that content authors
write plain Markdown without touching site machinery, and so the frozen
normative content is not entangled with freely edited prose.

## Decision

Separate presentation from content:

- **`website`** — the Hugo site: theme, styling, configuration, landing page,
  the go-import vanity, and site metadata. It is the single GitHub Pages origin
  for `valiss.dev`. It mounts the content repositories as **Hugo Modules**
  (which are Go modules) and renders them: `docs` under `/docs/`, `spec` under
  `/spec/`, and further sections as they appear.
- **`docs`** — pure Markdown user documentation (guides, quickstarts). No Hugo
  config, no theme; content only. Carries a `go.mod` so `website` can mount it.
- **`spec`** — pure Markdown `SPEC-N.md` plus the conformance `vectors/` and a
  `go.mod` (ADR 0010). Frozen per spec version. Mounted by `website` for
  `/spec/`, and pinned by implementations for the vectors.

Consequences for related decisions:

- The Pages build moves from static-branch to a **GitHub Actions Hugo build**
  (`github_repository_pages.build_type` `legacy` → `workflow`), updating the
  hosting mechanics of ADR 0003. The vanity meta itself is unchanged and is
  preserved as a static file in `website` so `go get valiss.dev/valiss` keeps
  resolving.
- The ADRs' home (ADR 0005 named "the docs/book repo") is the **`docs`** repo:
  they are Markdown, PR-governed there.

All repositories are bare-named per ADR 0007 (none is a distributable).

## Consequences

- Content authors edit plain Markdown in `docs`/`spec` with no Hugo knowledge;
  all styling and site logic is isolated in `website`.
- Content is version-pinned into the site via Hugo Modules, so a site rebuild
  picks a deliberate content revision rather than whatever is on a branch.
- One Pages origin, one build; the earlier "render docs elsewhere and publish
  into website" mechanism is unnecessary — a single Hugo build in `website`
  produces `/docs/` and `/spec/` together.
- `spec` serves two consumers cleanly: Hugo Module for rendering, and a pinned
  vector source for implementations.

## Alternatives considered

- **One `website` repo holding presentation and all content** — mixes freely
  edited prose, frozen normative content, and site machinery in one place;
  raises the authoring barrier and blurs freeze semantics.
- **A separate docs *site* repo that renders and publishes into `website`**
  (an earlier plan) — adds a cross-repo publish step and a second build for no
  benefit once `website` mounts content via Hugo Modules.
- **Folding `spec` into `docs` or `website`** — rejected: the frozen contract
  and its machine-consumed vectors do not belong with mutable guides or the
  presentation layer (ADR 0010).
