# 0001. Go module path is a vanity import under valiss.dev

- Status: accepted
- Date: 2026-07-15
- Deciders: mikluko

## Context

The Go reference implementation was transferred into the `valiss-dev` GitHub
org and renamed to `valiss-go`. Its original module path
`github.com/mikluko/valiss` no longer resolves. A new module path must be
chosen before the library is consumable again.

valiss is a multi-language framework. The project owns the `valiss.dev` domain,
which is the natural namespace root for every language and for documentation.

Go convention is that the last path element of the import path equals the
package name (`package valiss`), so linters flag a mismatch.

## Decision

The Go module path is **`valiss.dev/valiss`**, a vanity import path served from
the `valiss.dev` domain and mapped to `github.com/valiss-dev/valiss-go`.

## Consequences

- Import path stays `valiss.dev/valiss` regardless of future GitHub org or repo
  renames; only the `go-import` meta mapping changes. The org rename this
  session already demonstrated why this matters.
- Last path element (`valiss`) matches the package name: no linter noise, users
  write `valiss.Foo()`.
- Requires hosting a `go-import` meta tag at `valiss.dev` (see
  [0003](0003-go-vanity-import-hosting.md)); until that exists, `go get` fails.
- The repo name `valiss-go` differs from the import path. This is expected and
  is exactly what vanity paths decouple.

## Alternatives considered

- `github.com/valiss-dev/valiss-go` — zero infra, but the `-go` suffix mismatches
  the package name (linter warnings) and re-couples the import to the repo name,
  so any future rename breaks consumers again.
- `valiss.dev/go` — short, but last element `go` mismatches `package valiss`.
- `valiss.dev/valiss-go` — double stutter plus the same package-name mismatch.
