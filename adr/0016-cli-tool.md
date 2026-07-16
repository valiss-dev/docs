# 0016. The valiss CLI: naming, module path, and vanity mechanics

- Status: accepted
- Date: 2026-07-16
- Deciders: mikluko

## Context

Operating a valiss trust domain needs a polished chain-management CLI in the
spirit of NATS' `nsc`: minting operator/account/user tokens, managing keys and
creds files, revocation, with pluggable credential storage. The Go reference
ships only `examples/minter` — an example, not a product. The first decisions
are the tool's name, repository, and Go module/install mechanics; its command
surface and storage design are follow-up decisions.

Two Go-toolchain facts shape the mechanics:

- `go install` names the installed binary after the last element of the main
  package's import path.
- A module path is a label: its segments need not correspond to repository
  directories (`gopkg.in/yaml.v3` precedent). What binds a path to a repo is
  the vanity `go-import` meta whose declared root **equals** the module path;
  the toolchain's 404 walk-up only reuses a module root's page for that
  module's subpackages — it cannot define a new module root.

## Decision

- **Binary: `valiss`.** The flagship CLI takes the product name (the
  `vault`/`terraform` pattern); subcommands namespace the surface
  (`valiss account add`, `valiss user mint`, `valiss creds export`). Config
  surface aligns for free: `~/.valiss/`, `VALISS_*`.
- **Repository: `valiss-dev/valiss-cli`** — a distributable, so prefixed per
  [0007](0007-repository-naming-convention.md). Separate from `valiss-go`: a
  product's release cadence and packaging must not couple to library releases.
- **Module: `valiss.dev/cli/valiss`, with `package main` at the repository
  root.** The path is a pure label chosen so its base is the binary name:
  `go install valiss.dev/cli/valiss@latest` installs a binary named `valiss`
  with no `cmd/` ceremony and no nested main.
- **Vanity: one explicit page per Go module root**, minimal content — the
  `go-import`/`go-source` metas plus a human redirect to pkg.go.dev. The CLI
  adds `/cli/valiss/` beside the library's `/valiss/`. When the website moves
  to Hugo ([0011](0011-web-and-content-repos.md)), the pages are generated
  from a single registry data file (`vanity.yaml`: module → repo), so adding a
  module root is one line and pages cannot drift.
- The vanity page and the repository are created **together**; publishing the
  mapping first would only move the failure from meta resolution to clone.

## Consequences

- One name everywhere: the framework, the domain, the binary, the dot-dir.
- `go install valiss.dev/cli/valiss@latest` works with root-main layout;
  Homebrew/release binaries carry the same `valiss` name.
- Each future module root (another tool, a spec module) costs exactly one
  registry line / one static page.
- Follow-up ADRs: storage-plugin interface, command framework, and the fate of
  `valiss-go/examples/minter` (absorb or retire).

## Alternatives considered

- **Binary `vsc`/`valissctl`/`minter`** — a 3-letter nsc-alike collides with
  VS Code mindshare; `-ctl` is ceremony without gain; `minter` names one
  function of a chain-management tool.
- **Module `valiss.dev/cli` with root main** — `go install` would name the
  binary `cli`; renaming post-install is not a default anyone keeps.
- **`cmd/valiss` layout** — works (`gopls` pattern inverted) but adds the
  nested-main ceremony the root-main layout avoids; rejected on taste with no
  functional loss.
- **Serving the CLI meta from a parent path** (`/cli`, or the landing page) —
  a meta root of `valiss.dev/cli` would force the module into a `valiss/`
  subdirectory, and one root page cannot map two repositories; both defeat the
  layout.
