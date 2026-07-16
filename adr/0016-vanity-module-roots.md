# 0016. One explicit vanity page per Go module root

- Status: accepted
- Date: 2026-07-16
- Deciders: mikluko

## Context

[0003](0003-go-vanity-import-hosting.md) established the vanity hosting for the
single library module (`valiss.dev/valiss`), relying on the toolchain's 404
walk-up so one page covers every subpackage. The project is about to publish
more Go module roots under `valiss.dev` (a CLI, potentially other tools), which
the walk-up does **not** cover:

- The walk-up only reuses an existing module root's page for that module's
  subpackages; it cannot define a new module root. A request for an undeclared
  root walks to pages that either do not exist or declare a different root, and
  resolution fails.
- Serving a new module from a parent-path meta does not work either: a meta
  root that is a proper prefix of the module path makes the toolchain expect
  the module in a corresponding repository subdirectory, and a single page
  cannot map two repositories.
- A module path is otherwise a pure label — its segments need not correspond
  to repository directories (`gopkg.in/yaml.v3` precedent) — so a repo may
  freely claim a path like `valiss.dev/cli/valiss` with `package main` at its
  root, provided the meta root equals the module path exactly.

## Decision

- **Every Go module root gets exactly one explicit vanity page** whose
  `go-import` meta root equals the module path, with minimal uniform content:
  the `go-import` and `go-source` metas plus a human redirect to pkg.go.dev.
  Subpackages keep riding the walk-up; new module roots never do.
- A module root's vanity page and its repository are created **together**;
  publishing the mapping first only moves the failure from meta resolution to
  clone.
- When the website moves to Hugo ([0011](0011-web-and-content-repos.md)), the
  pages are **generated from a single registry data file** (`vanity.yaml`:
  module → repo). Adding a module root becomes one registry line; hand-authored
  pages cannot drift apart. Until then, new pages are static twins of
  `/valiss/`.

## Consequences

- Publishing a new Go tool or module under `valiss.dev` costs one registry
  line (one static page today), regardless of the repository's internal
  layout.
- Module paths can be chosen for install ergonomics (the `go install` binary
  name is the path's last element) independently of repository structure.
- The registry becomes the authoritative map of the `valiss.dev` Go namespace.

## Alternatives considered

- **Relying on walk-up alone** — covers only subpackages of already-declared
  roots; new roots fail to resolve.
- **A parent-path or landing-page meta** — forces subdirectory layouts and
  cannot map multiple repositories.
- **A dynamic vanity responder** — rejected already in
  [0003](0003-go-vanity-import-hosting.md); nothing here needs runtime logic.
