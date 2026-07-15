# 0002. Namespace rooted at valiss.dev, per-language repos named valiss-<lang>

- Status: superseded by [0007](0007-repository-naming-convention.md)
- Date: 2026-07-15
- Deciders: mikluko

## Context

valiss ships one wire-compatible implementation per language (Go and Python
today; TypeScript, Rust, Zig, Java planned). The project needs a consistent
convention for repository names and for how each ecosystem's package is named,
so that the family reads as one project and sorts predictably.

## Decision

- The `valiss.dev` domain is the umbrella namespace for the whole framework.
- Each language implementation lives in its own repo named **`valiss-<lang>`**
  in the `valiss-dev` org: `valiss-go`, `valiss-py`, `valiss-ts`, `valiss-rs`, …
- Package/registry names follow each ecosystem's idiom rather than the repo
  name: Go import `valiss.dev/valiss`, PyPI `valiss`, npm `@valiss/core` (or
  `valiss`), crates `valiss`.
- Infrastructure and cross-cutting concerns live in dedicated repos
  (`valiss-infra`, and a docs/book repo to be named later).

## Consequences

- Repos sort together and the language is obvious from the repo name.
- The import/registry name can stay idiomatic and short without being dictated
  by the repo name (reinforces [0001](0001-go-module-path.md)).
- Adding a language is mechanical: new `valiss-<lang>` repo, same conventions.

## Alternatives considered

- A single monorepo for all languages — rejected: complicates per-ecosystem
  tooling, release cadence, and registry publishing.
- Encoding the language in the package/registry name (e.g. npm `valiss-ts`) —
  rejected: redundant within an ecosystem where the registry already implies
  the language.
