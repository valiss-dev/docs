# 0014. Trust-tree management ships as a first-class CLI, grown from the minter

- Status: proposed
- Date: 2026-07-16
- Deciders: mikluko

> Concerns tooling only. The CLI is a workflow layer over the reference
> library's issue/verify API; it defines no wire behavior and adds no normative
> surface. The wire contract stays owned by the spec and the conformance vectors
> ([0006](0006-spec-versioning.md), [0009](0009-wire-format-versioning.md),
> [0010](0010-conformance-model.md), [0012](0012-vector-immutability.md)).

## Context

valiss is a three-level chain of Ed25519 keys — operator signs account
(tenant), account signs user, plus an optional per-message level. Running that
trust tree is a lifecycle: generate a key at the right level, issue the token
that delegates to it, assemble a creds file (lean, bundle, or bearer), tell the
server which account `jti` to allowlist, bump the operator epoch and re-mint to
mass-revoke, and inspect or verify an artifact offline. Today only the first two
steps have a tool: `valiss-go/examples/minter`, and it is deliberately *an
example, not a product* — the Go library is library-first with no product
binary. Everything else is raw calls against `Issue*`/`Verify*`/`creds`, which
is expert-only and invites exactly the mistakes the scheme spends cryptography
to prevent: signing across key levels, minting against a stale epoch, leaking a
seed, issuing a plain token the fail-closed transports will reject.

The maintainer named two starting points, and they pull in opposite directions
on the one axis that matters most — **does the tool hold keys and own state?**

- **`nsc`** (NATS) manages the same operator/account/user shape valiss borrowed
  its delegation model and creds format from. But `nsc` is **stateful with key
  custody**: a keystore, a config store, and an account server to push to. Its
  strength is that it owns the whole tree on disk.
- **The `minter`** is the deliberate opposite: **stateless, no custody**. The
  manifest holds public data only (safe to commit); signing seeds arrive from
  `VALISS_SEED_<PUBKEY>` env vars; nothing is written anywhere. It fits the
  scheme's premise — offline verification, no infrastructure to issue — but it
  only mints.

So "be like `nsc`" and "grow the minter" are in tension. Any decision also has
to respect prior ADRs: a tool you install and run is not a dependency you import
([0007](0007-repository-naming-convention.md)'s dividing test); the project
consolidates rather than sprawls repos and pipelines
([0008](0008-python-package-structure.md)); and the library and wire are pre-1.0
and unfrozen ([0006](0006-spec-versioning.md)), so new repos and release
surfaces are costly to stand up now.

## Decision

**Promote trust-tree management to a first-class CLI**, grown from
`examples/minter` and backed by the reference library. It is a maintained
product, not an example; the Go module's "no product binary" posture is
reversed for this one binary and `valiss-go/AGENTS.md` is updated to say so.

- **Home: `valiss-go/cmd/valiss`, installed with `go install
  valiss.dev/valiss/cmd/valiss@latest`.** It lives inside the reference module
  so it reuses `Issue*`/`Verify*`/`creds`/`Keyring`/`Allowlist` with no wrapper
  and versions with the library — the in-module consolidation of
  [0008](0008-python-package-structure.md) and the `contrib/*` layout, applied
  to a command. Per [0007](0007-repository-naming-convention.md) it is a tool,
  not an importable distributable, so it earns no `valiss-<tag>` repo of its
  own. Prebuilt, signed cross-platform binaries (a release-time artifact) are
  how non-Go users get it; being authored in Go no more makes it Go-only than
  `nsc` being Go makes it serve only Go clients.

- **Name: `valiss`**, subcommand-structured (`keygen`, `issue`, `creds`,
  `verify`, `describe`, `revoke`/`rotate`, `tree`), superseding `minter` — which
  named only issuance and undersells a lifecycle tool.

- **Opinionated: the tool encodes the scheme's invariants so the easy path is
  the correct one.**
  - *Key-level strictness.* `keygen operator|account|user` emits only the right
    nkey type; the tool signs strictly down the chain (operator→account→user)
    and refuses cross-level operations, matching the library's hierarchy and the
    role-in-key-material property.
  - *Rotation is a verb.* `rotate` re-mints the operator token at the next epoch
    and re-issues account/user tokens stamped with it — the cryptographic
    mass-revocation workflow — instead of leaving operators to wire `WithEpoch`
    and `WithOperatorToken` by hand.
  - *Allowlist surfaced.* Minting an account emits the account `jti` a server
    must allowlist as explicit, machine-readable output (the minter already
    writes it to stderr), and the tool can maintain an allowlist file.
  - *creds / bundle / bearer* are explicit modes with the library's exact
    semantics (lean user creds, `-bundle` embedding the account token, bearer
    token-only with no seed).
  - *Fail-closed extensions.* Because the contrib transports reject a token that
    lacks their extension unless `AllowMissingExtension()` is set, the tool
    attaches typed extensions (`httpauth.Ext`, `grpcauth.Ext`, domain
    extensions) and warns when it mints a plain identity token those transports
    will refuse — no silently-useless creds.
  - *Seed hygiene.* Seeds print exactly once at `keygen` (to stdout, guidance to
    stderr — the minter's separation preserved), are never logged, and are
    written to disk only where the model requires (creds) or the user opts into
    a store.
  - *Offline by default.* Verify and inspect need no network and there is no
    mandatory resolver or account server — a deliberate divergence from `nsc`,
    aligned with offline verification.

- **Generic: custody-agnostic, stateless by default, scriptable.** This is how
  the `nsc`/minter tension resolves — **custody is available but never
  mandatory.**
  - *Stateless core (the minter's stance).* By default the tool holds no keys
    and owns no state: the public tree is a commit-safe manifest; signing seeds
    come from a **pluggable key source** — env (`VALISS_SEED_<PUB>` as today),
    files, or an OS keychain — behind one interface.
  - *Optional local store.* An opt-in `init`/`--store` mode provides an
    `nsc`-style layout — a keystore for seeds kept separate from a store for
    public tokens and the tree, XDG paths, `0600` — for those who want the tool
    to manage custody. It ships after the stateless path, not before it.
  - *Machine-friendly.* Every command offers `--output json|yaml|text`, reads
    and writes stdin/stdout, returns meaningful exit codes, and is
    non-interactive first, so it drops into CI, GitOps, and secrets managers.
  - *Deterministic and policy-free.* Absolute RFC3339 validity means re-minting
    a manifest yields the same window (as the minter guarantees); no
    deployment's authorization is baked in — all authz rides pluggable typed
    extensions, not scope strings.

- **Non-normative and library-backed.** The CLI ships no crypto or wire logic of
  its own; it orchestrates the library, so every artifact it emits is already
  governed by the spec and the conformance vectors
  ([0006](0006-spec-versioning.md), [0009](0009-wire-format-versioning.md),
  [0010](0010-conformance-model.md), [0012](0012-vector-immutability.md)). If
  the CLI and the library ever disagree, the library is canonical. The tool is
  therefore **not** part of the interop matrix
  ([0010](0010-conformance-model.md),
  [0013](0013-library-lifecycle-interop-gate.md)): that gates implementations
  of the wire, and the CLI implements none.

## Consequences

- Operators get a safe, repeatable lifecycle tool; the error classes the scheme
  prevents cryptographically are now also prevented ergonomically. The `minter`
  graduates from example to product, and a getting-started guide for it belongs
  in this repo (the "Getting started (Go)" and "Verifying credentials offline"
  entries in `guides/`).
- `valiss-go/AGENTS.md`'s "no product binary, only `examples/`" statement
  becomes false and must be updated; `examples/minter` becomes `cmd/valiss`,
  while the remaining `examples/*` stay end-to-end demos.
- One more thing releases from `valiss-go`. `go install` works immediately;
  prebuilt binaries add a release step and a supply-chain surface (checksums,
  signing) to secure — the same single-publisher concern
  [0008](0008-python-package-structure.md) raises for a security library, now
  for a binary.
- Custody-optional keeps the default safe and stateless (CI/GitOps) while still
  serving custody users, but the store's on-disk format is a new thing to design
  and secure (file layout, permissions, at-rest encryption). It is deferrable
  behind the flag; the stateless path ships first.
- The normative surface does not grow: no new vectors, no matrix cells, nothing
  added to freeze. Keeping the tool outside conformance is deliberate.
- Pre-1.0, the CLI's own flags and UX may churn with the library; that is
  expected and cheap while both are unstable. A second-language tool (e.g. a
  `valiss-py` console script) is possible but **deferred** — the Go binary
  serves all users meanwhile, so tooling does not fragment per language now
  (echoing [0007](0007-repository-naming-convention.md)'s one-artifact-per-
  ecosystem, adapters-on-demand instinct).

## Alternatives considered

- **Keep `minter` as the only tool (library-only, status quo).** Rejected: an
  example carries no product guarantees, covers only issuance, and pushes
  verify/rotate/revoke/inspect onto raw library calls — expert-only, and exposed
  to exactly the mistakes the model exists to prevent.
- **Port or fork `nsc`'s stateful-custody model as the default.** Rejected:
  `nsc` mandates a keystore, a config store, and an account server to
  distribute; valiss verifies offline and deliberately shed custody in the
  minter. Making state and custody mandatory fights the scheme's
  no-infrastructure premise. We take `nsc`'s tool shape, lifecycle verbs, and
  its store as an *option*, not its mandatory statefulness.
- **A dedicated bare tool repo now**, with its own pipeline. Rejected pre-1.0:
  it would depend on `valiss.dev/valiss` anyway, duplicate release machinery,
  and split library and tool cadence before either is stable. `cmd/` in-module
  is the low-friction start; a split into its own **bare** repo
  ([0007](0007-repository-naming-convention.md)) is the documented escape hatch
  ([0008](0008-python-package-structure.md)-style split-on-demand) if cadence or
  a second language's tool ever demands it.
- **Many small binaries** (`valiss-keygen`, `valiss-creds`, …) instead of one.
  Rejected: subcommands under one binary match the mental model and a single
  install; separate binaries multiply distribution for no gain.
- **Names `vsc` (mirror `nsc`) or keeping `minter`.** `vsc` is opaque and
  collides with editor muscle memory; `minter` names only issuance. `valiss` is
  the project's own namespace and reads as *the* tool. This is the sub-point
  most open to revision in review.
- **A GUI, web console, or server-side issuance API.** Out of scope: issuing
  credentials must never require production infrastructure, so a local, offline
  CLI is the aligned surface. A service could later be layered on the same
  library, but that is not this decision.
