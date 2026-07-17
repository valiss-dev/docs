---
title: Tokens
weight: 2
---

A token is the signed artifact that binds one key to another. Structurally it
is a JWT: a JWS Compact Serialization string of three base64url parts,
`header.payload.signature`, where the payload is JSON using the familiar
registered claim names (`iss`, `sub`, `exp`, `nbf`, `iat`, `jti`) plus a nested
`valiss` object that carries the level-specific body.

What makes valiss a chain rather than a pile of independent tokens is who signs
what.

## Parent-signed chains

Each token is signed by the key one level up, and the signing rules are strict:

| token    | signed by       | subject (`sub`) is      |
| -------- | --------------- | ----------------------- |
| operator | itself          | the operator key        |
| account  | an operator key | the tenant's key        |
| user     | an account key  | the user's key          |
| message  | itself          | the emitting user's key |

An account token is only valid if its issuer is the operator you pinned. A user
token is only valid if its issuer is the `sub` of a valid account token. So a
verifier does not trust a user token on its own: it walks the chain from the
pinned operator key down to the presented credential, checking the signature and
the role of each key at every hop. Trust is a property of the whole chain, not
of any single token.

This is also why delegation cannot widen. An account key can sign user tokens,
but it can never sign another account token, because the verifier requires an
account token's issuer to be an operator key. The role prefix baked into each
nkey (see [Entities](entities.md)) is what makes that check cheap and total.

Operator and message tokens are self-signed (`iss == sub`): the operator token
is a policy statement the operator makes about its own domain, and a message
token is a proof the user makes about a payload it is emitting.

## The token id is a content hash

The `jti` claim is not random. It is derived from the token's own contents:
serialize the payload with `jti` blanked, take the SHA-256 of those bytes, and
encode the digest as base32. The result is a stable 52-character identifier.

Two consequences matter in practice:

- **The id is reproducible.** Anyone who can see a token can recompute its
  `jti` and confirm the id was not tampered with. Nothing needs to look it up.
- **Identical claims yield an identical id.** Re-issuing a token with the same
  contents produces the same `jti`, which is what lets keyrings and allowlists
  deduplicate by id, and what lets account token ids serve as globally unique
  allowlist keys that cannot collide between operators (see
  [Allowlist](allowlist.md)).

The valiss CLI (early development) exposes this reproducibility as `inspect`: an
offline decode of any token that prints its claims and derived id without
evaluating trust, the quick way to see what a token carries and confirm its
`jti`.

Reproducing a `jti` byte-for-byte across languages requires reproducing the
exact JSON serialization the reference implementation uses (field order, no
insignificant whitespace, and Go's HTML-style escaping of `<`, `>`, and `&`). A
verifier that only checks signatures may treat `jti` as opaque; one that
re-derives it must match the serialization exactly.

## Validity is optional and absolute

`exp` and `nbf` are absolute Unix-second timestamps and both are optional. An
absent `exp` means the token never expires; an absent `nbf` means it is valid
immediately. Window checks apply a small clock-skew tolerance (2 minutes by
default) so that a slightly fast or slow clock does not spuriously reject a
fresh token.

## Not a stock JWT, on purpose

valiss tokens are shaped like JWTs but are deliberately **not** verifiable with
off-the-shelf JWT libraries:

- the algorithm identifier is `ed25519-nkey`, not the registered `EdDSA`, so a
  standards-compliant library rejects the header outright; and
- `iss` and `sub` carry nkey-encoded public keys, not JWKs.

This is a considered trade. Putting the key's role into the key material is what
makes the strict signing hierarchy checkable at every hop, and no stock JWT
middleware can walk a multi-level chain, enforce epochs, or check payload
checksums anyway. The signature check is the only layer such a library could
have reused. The library is the canonical verifier; other languages port the
verification algorithm rather than reuse a JWT stack.
