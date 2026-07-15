# 0010. Two-layer conformance: static vectors and a live interop matrix

- Status: accepted
- Date: 2026-07-15
- Deciders: mikluko

## Context

valiss is a multi-language framework whose entire value is that a credential
minted by one implementation verifies in every other. That interoperability has
to be *proven*, and two different classes of bug threaten it:

- **Wire-level divergence** — implementations disagree on bytes or on outcomes:
  JSON escaping that changes the `jti` hash, signature encoding, header
  dispatch, or which reason a bad artifact is rejected for.
- **Transport-level divergence** — the wire agrees, but the real HTTP/gRPC
  integration does not: header names, content negotiation, middleware wiring,
  version dispatch on the request path.

A single mechanism catches only one class. Static offline tests are precise and
cheap but never exercise a real server/client exchange; live end-to-end tests
exercise the whole stack but are noisy and, on failure, do not tell you *which
byte* differs.

## Decision

Adopt a **two-layer conformance model**, wire-first.

**Layer 1 — static conformance vectors (the wire contract).** Frozen JSON
vectors: a corpus of artifacts (tokens, signatures, creds files) each paired
with the expected outcome (success + verified claims, or failure + a spec §7
reason code). They are verify-side and deterministic; every implementation
ships an offline runner that must pass all of them. Vectors are **normative**
and live with the specification in the `spec` repo (`spec/vectors/`), frozen per
spec version (a change that flips an outcome is a new spec version, not an
edit). They are generated from the reference implementation and reviewed.

**Layer 2 — live interop matrix (the transport proof).** A dedicated **`interop`**
repository (bare-named per [0007](0007-repository-naming-convention.md); it is
project-internal test infrastructure, not a distributable) holding:

- a shared, language-neutral **scenario suite** and key fixture (valid creds
  accepted, revoked/expired/wrong-key rejected, bearer flow, message-token
  flow);
- a per-implementation **harness contract**: each impl provides a server and a
  client binary speaking a fixed protocol;
- an **orchestrator** that runs the grid `server_lang × client_lang ×
  transport` (HTTP and gRPC), starting each server and running each client
  suite against it, cross-language included.

It generalizes the existing `valiss-py/tests/interop` Go↔Py seed.

**Ordering is wire-first.** The vector layer gates the matrix: if the wire
disagrees, live tests fail confusingly, so vectors must be green before the
matrix runs. The matrix then tests transport integration on top of a known-good
wire.

## Consequences

- Both layers run in CI: vectors as a fast per-implementation gate, the matrix
  as a cross-implementation integration gate.
- The matrix grows as `O(languages² × transports)`; Go and Python over HTTP and
  gRPC is a 2×2×2 = 8-cell start.
- A new language implementation is "done" only when it passes the vectors and
  joins the matrix.
- Vectors staying in `spec` keeps the normative contract (prose plus executable
  examples) in one place; `interop` stays purely test harness. See
  [0006](0006-spec-versioning.md) and [0009](0009-wire-format-versioning.md).

## Alternatives considered

- **Vectors only** — misses every transport-level bug; the wire can be correct
  while the HTTP/gRPC integration is not.
- **Live matrix only** — exercises everything but is noisy, cannot pinpoint a
  byte-level divergence, and pays the `O(n²)` cost without a cheap wire gate to
  fail fast.
- **One `conformance` repo holding vectors and the matrix** (spec = pure prose)
  — rejected: the vectors are normative and define behavior as much as the
  prose does, so they belong with the spec; separating them weakens the
  contract and the repo name (`interop`) would be less accurate than one
  spanning both.
