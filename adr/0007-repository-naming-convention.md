# 0007. Repository naming convention

- Status: accepted
- Date: 2026-07-15
- Deciders: mikluko
- Supersedes: [0002](0002-repo-and-namespace-naming.md)

## Context

The `valiss-dev` org holds two kinds of repositories: distributable
implementations of the framework, and repositories that support the project but
are never depended on. [0002](0002-repo-and-namespace-naming.md) established the
`valiss-<lang>` pattern for implementations but did not give a rule for the
supporting repositories, nor a rule for languages that share a runtime (JVM) or
compile target (TypeScript to JavaScript).

## Decision

### The dividing test

A repository's name is decided by one question:

> **Does a consumer pull this into their own project as a dependency/artifact?**

- **Yes → `valiss-<tag>`** (prefixed). It is published to a package registry and
  imported by user code.
- **No → `<name>`** (bare). It is project-internal or meta: it supports
  developing, documenting, or operating valiss, but nothing depends on it.

Examples: `valiss-go`, `valiss-py`, `valiss-ts` are prefixed; `infra`, `docs`,
`examples`, `.github` are bare.

### The unit of a distributable repo is the distribution ecosystem

One `valiss-<tag>` repository per **distribution ecosystem** (runtime + package
registry), not per language. Languages that share an ecosystem are served by one
core artifact through interop, plus optional idiomatic adapter artifacts if
demand appears (adapters are themselves published, so also `valiss-<lang>`).

| Ecosystem      | Repo         | Serves                                     |
| -------------- | ------------ | ------------------------------------------ |
| Go             | `valiss-go`  | Go                                         |
| Python         | `valiss-py`  | Python                                     |
| Node / browser | `valiss-ts`  | TypeScript + JavaScript                    |
| JVM            | `valiss-jvm` | Java, Kotlin, Scala, Clojure (via interop) |
| Rust           | `valiss-rs`  | Rust                                       |
| Haskell        | `valiss-hs`  | Haskell                                    |
| Zig            | `valiss-zig` | Zig                                        |

- `valiss-jvm` is the Java-authored core on Maven Central; Kotlin/Scala consume
  it directly. Idiomatic layers (`valiss-kotlin`, `valiss-scala`) are optional
  and deferred.
- `valiss-ts` serves JavaScript too; there is no separate `valiss-js`.

### Registry/import names stay idiomatic

The published name follows each ecosystem's idiom, not the repo name: Go import
`valiss.dev/valiss`, PyPI `valiss`, npm `@valiss/core`, crates `valiss`, Maven
coordinates under a `dev.valiss` group. This is unchanged from
[0001](0001-go-module-path.md).

### Consequences

- Every repo name is decidable from the dividing test; no case-by-case debate.
- Bare supporting repos sort ahead of `valiss-` implementations in an
  alphabetical listing, so meta/support floats to the top and implementations
  cluster together.
- Adding a language is mechanical, and the JVM family does not fragment into
  three repos.
- The repo created earlier as `valiss-infra` is renamed to **`infra`**; ADR 0004
  and cross-references are updated accordingly.
- 0002 is superseded; this ADR is the single authoritative naming record.

## Alternatives considered

- Naming distributable repos per language rather than per ecosystem — produces
  redundant JVM repos (`valiss-java`, `valiss-kotlin`, `valiss-scala`) for one
  interop-shared artifact.
- Prefixing every repo (`valiss-infra`, `valiss-docs`) — loses the
  dependency/meta distinction and the useful sort ordering.
- Leaving supporting repos unspecified (0002 as-is) — the gap this ADR closes.
