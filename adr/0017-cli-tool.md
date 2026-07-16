# 0017. The valiss CLI: name, repository, module path

- Status: accepted
- Date: 2026-07-16
- Deciders: mikluko

## Context

Operating a valiss trust domain needs a polished chain-management CLI in the
spirit of NATS' `nsc`: minting operator/account/user tokens, managing keys and
creds files, revocation, with pluggable credential storage. The Go reference
ships only `examples/minter` — an example, not a product. This ADR fixes the
tool's name, repository, and module/install mechanics; the command surface and
storage design are follow-up decisions.

One toolchain fact shapes the module choice: `go install` names the installed
binary after the last element of the main package's import path, and per
[0016](0016-vanity-module-roots.md) a module path is a label that need not
mirror the repository layout.

## Decision

- **Binary: `valiss`.** The flagship CLI takes the product name (the
  `vault`/`terraform` pattern); subcommands namespace the surface
  (`valiss account add`, `valiss user mint`, `valiss creds export`). The
  config surface aligns for free: `~/.valiss/`, `VALISS_*` env vars.
- **Repository: `valiss-dev/valiss-cli`** — a distributable, so prefixed per
  [0007](0007-repository-naming-convention.md). Separate from `valiss-go`: a
  product's release cadence and packaging must not couple to library releases.
- **Module: `valiss.dev/cli/valiss`, `package main` at the repository root.**
  The label is chosen so its base is the binary name:
  `go install valiss.dev/cli/valiss@latest` installs a binary named `valiss`
  with no `cmd/` ceremony and no nested main. The module root gets its vanity
  registry entry per [0016](0016-vanity-module-roots.md), created together
  with the repository.

## Consequences

- One name everywhere: the framework, the domain, the binary, the dot-dir.
- Homebrew/release binaries carry the same `valiss` name; `go install` agrees.
- Follow-up ADRs: the storage-plugin interface, the command framework, and the
  fate of `valiss-go/examples/minter` (absorb or retire).

## Alternatives considered

- **Binary `vsc`/`valissctl`/`minter`** — a 3-letter nsc-alike collides with
  VS Code mindshare; `-ctl` is ceremony without gain; `minter` names one
  function of a chain-management tool.
- **Module `valiss.dev/cli` with root main** — `go install` would name the
  binary `cli`; renaming post-install is not a default anyone keeps.
- **`cmd/valiss` layout** — works, but adds the nested-main ceremony the
  root-main layout avoids; rejected on taste with no functional loss.
- **Shipping the CLI inside `valiss-go`** — couples product releases to
  library releases and bloats the library module with CLI dependencies.
