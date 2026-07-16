# 0015. Credential and proof-of-origin adapters are separate contrib packages

- Status: accepted
- Date: 2026-07-16
- Deciders: mikluko

## Context

`valiss-go` ships two adapter families under `contrib/`: credential
authentication (request auth: `httpauth`, `grpcauth`, `ginauth`, `echoauth`)
and per-message proofs of origin (message tokens: `httpsig`, `grpcsig`,
`ginsig`, `echosig`). The scheme itself treats these as distinct concerns —
a credential grants an identity; a message token proves the origin of exact
bytes and grants nothing (valiss-go#9, design point 4).

The separation is established practice with a thin paper trail. Message-token
wiring originally lived inside the auth packages (`message.go`,
`Message*`-prefixed names) and was split out in valiss-go commit `97809d1`
with a one-line rationale: "separating the two concerns the way the scheme
itself does (credentials vs proofs) and shortening the names." No decision
record existed. With framework adapters multiplying the package count
([0014](0014-go-framework-integration-targets.md) targets nine frameworks,
two packages each), the layout is now load-bearing and needs ratifying or
revisiting.

## Decision

Ratify the established practice. In `valiss-go`, every transport or
framework integration separates its two concerns into two `contrib/`
packages:

- **`<platform>auth`** — credential authentication: server middleware or
  interceptors verifying the per-request credential and enforcing transport
  extensions, plus the client side where the platform has one. Grants an
  identity (`valiss.IdentityFromContext`).
- **`<platform>sig`** — message tokens: receiving middleware or interceptors
  verifying per-message proofs of origin, including chain negotiation.
  Grants no identity (`valiss.MessageFromContext`).

Future contribs must comply. Framework adapters in either family wrap the
transport package's exported verification core (`httpauth.Authenticate`,
`httpsig.Receiver`); they never reimplement verification.

## Consequences

- **Security legibility.** The package boundary structurally enforces the
  doctrine that possession of a message token grants nothing: the two verify
  paths, context accessors, and option sets never share a namespace, so an
  integrator cannot mistake `MessageFrom` for authentication. Protection on a
  route is visible in its imports; combining both is explicit stacking.
- **No false sharing.** The families share no configuration surface
  (verifier/allowlist/extensions/replay cache vs operator key/chain
  negotiation/chain cache/body binding); a merged package would still contain
  two of everything.
- **Package count.** Two small packages per platform instead of one larger
  one; at the [0014](0014-go-framework-integration-targets.md) horizon,
  eighteen instead of nine. Accepted: each is thin sugar over a shared core,
  and the `<platform><concern>` names keep a sorted or grepped listing
  self-organizing.
- **Cross-language divergence is deliberate.** Python merges per framework
  (`valiss.django` carries everything Django;
  [0008](0008-python-package-structure.md)). Consistency across languages is
  held at the distribution level (one distributable per ecosystem, ADR 0007),
  not at internal module layout, where each ecosystem follows its own norms.

## Alternatives considered

- **Merged per transport** (message tokens inside `httpauth`/`grpcauth`) —
  the original state, abandoned by `97809d1`: `Message*` name prefixes
  reintroduced the separation awkwardly inside one namespace, and the
  credential/proof distinction blurred exactly where it matters most.
- **Merged per framework** (`contrib/gin` carrying both families,
  Python-style) — halves the package count but breaks the
  `<platform><concern>` naming algebra, splits convention between transport
  and framework adapters, and puts `IdentityFrom` and `MessageFrom` side by
  side in one API surface, inviting the conflation the doctrine forbids.
