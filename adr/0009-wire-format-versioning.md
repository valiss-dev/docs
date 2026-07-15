# 0009. Wire-format version discriminator

- Status: accepted
- Date: 2026-07-15
- Deciders: mikluko

## Context

[0006](0006-spec-versioning.md) requires a wire-format version that lets a
future spec version coexist with the current one: a verifier must decide, from
the bytes alone, which version an artifact is, so it can dispatch to the right
rules or reject cleanly rather than mis-parse. An audit of the Go reference
found no such discriminator on any of the three serialized artifacts (tokens,
creds file, request signature); the only on-wire tag was the JWT header's fixed
`alg`, which identifies the algorithm, not the format version.

## Decision

Add an explicit version discriminator to each artifact, dispatched **before**
the versioned content is parsed. This is the single pre-freeze breaking change
for spec 1; the current wire version is **1**.

- **Tokens.** The JWT header carries `"ver":1`. A version-agnostic reader
  (`peekVersion`) reads the envelope and `ver` without touching the payload;
  `decodeToken` dispatches on it to a per-version decoder (`decodeV1`, future
  `decodeVN`) that normalizes into a version-neutral internal struct. All spec
  versions share the JWS-compact envelope (`base64url(header).payload.sig` with
  a JSON header); `ver` versions the contents within it.
- **Creds file.** A `VALISS-CREDS-VERSION: 1` header line, checked before the
  payload. It versions the file *container* only; the tokens inside carry their
  own token version. An absent header reads as the current version.
- **Request signature.** A version tag is bound into the signed bytes
  (`valiss-req-v1\n…`). Because it is signed, a verifier that reconstructs v1
  bytes fails **closed** on any other version rather than mis-verifying.
  Explicit dispatch via a transport `valiss-version` header is the additive
  step introduced when a second version exists; v1 needs no such signal.

Two normative rules hold for every version:

1. **Version selects the parser; the signature is always verified.** Dispatch on
   an unauthenticated header must never skip verification, so a forged version
   either routes to a real parser that rejects the bad signature or is an
   unknown version that is rejected outright. Downgrade is blocked because the
   version is part of the signed material.
2. **The JWS-compact envelope is fixed across versions.** `ver` lives in the
   header, so it is only readable while the envelope shape holds. Committing to
   that shape is deliberate; a hypothetical future that abandons JWS entirely
   would need an outer discriminator, which is explicitly out of scope.

Versioning appears in the public API only as an integer value, never as a named
identifier: the wire layer is suffixed internally (`wireV1`, `decodeV1`,
`tokenHeaderV1`, …), while the public functions and the `Claims` result types
stay version-free. Tokens are always minted at the current version; any
supported version can be read.

## Consequences

- Spec 1 can be frozen with a real coexistence guarantee (satisfies
  [0006](0006-spec-versioning.md), which moves to accepted).
- Adding a version is additive: new `wireVN`/`decodeVN` types and one `case` in
  the dispatcher, with nothing outside that set changed. This is verified by a
  test that mints a v1 token and rejects both a synthetic future v2 and a
  synthetic version-0 (ver-absent) token before payload parsing.
- Already-issued pre-freeze tokens (which lack `ver`) no longer verify and must
  be reissued. Acceptable: the module does not yet resolve under its new path,
  so there are no external holders (see the v0.12.0 rollout).
- The request-signature change is breaking, but requests are ephemeral, so there
  is nothing to migrate.

## Alternatives considered

- **Overload `alg`** (e.g. `ed25519-nkey-2` for v2) — reuses an existing checked
  field, but conflates algorithm with format version: a version that keeps
  Ed25519 but changes the payload layout could not be expressed.
- **Version in the payload** — rejected: the payload cannot be safely parsed
  until the version is known, so the discriminator must precede it (header or
  envelope).
- **Outer byte prefix before the JWS string** — most flexible (survives even an
  envelope change), but breaks JWS-compact tooling compatibility for a benefit
  that only a non-JWS future version would need.
