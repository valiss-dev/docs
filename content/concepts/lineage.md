---
title: valiss and NATS
weight: 9
description: "What valiss inherits from NATS and its nsc tooling (nkey encoding, the operator/account/user chain, the credentials-file shape, the ed25519-nkey JWT algorithm, content-hash token ids) and what it deliberately changes, with a note on what an nsc user will find familiar and different."
---

valiss did not invent its key encoding, its delegation chain, or its
credentials-file layout. All three come, closely and deliberately, from
[NATS](https://nats.io) and its `nsc` tooling. The reference implementation
imports the same `github.com/nats-io/nkeys` library that NATS itself uses, and if
you have run `nsc` you will recognize a great deal of valiss on sight. This page
credits that heritage plainly, says exactly what is inherited and what is
changed, and explains why each change was made. Being coy about the lineage would
only make it look hidden; it is not.

## What valiss inherits from NATS

**The nkey encoding, with role prefixes.** Identities are Ed25519 keypairs
rendered in the NATS nkey text format: base32 of a one-byte role prefix, the
32-byte public key, and a CRC-16 checksum. That format, and the role prefixes
that make an operator key read as `O…`, an account as `A…`, and a user as `U…`
(with seeds `SO…`, `SA…`, `SU…`), are the nkeys library's, not valiss's. valiss
depends on that library directly for creating, validating, and decoding keys. See
[Entities](entities.md) for how the encoding is used.

**The operator, account, user trust chain.** The three-level hierarchy in which
an operator signs account credentials and an account signs user credentials is
the shape of NATS 2.0 decentralized authentication, the model `nsc` exists to
manage. valiss keeps that chain, including the property that the operator public
key is the single trust anchor a verifier pins.

**The credentials-file shape.** A valiss [credentials file](creds.md) is
line-oriented, delimited by `-----BEGIN … -----` and `------END … ------`
markers, and carries a plain-language warning after any seed. That is the same
shape NATS uses for its `.creds` files, down to the marker style and the seed
warning, which is why the format feels familiar and stays copy-paste-safe.

**The `ed25519-nkey` JWT algorithm.** valiss [tokens](tokens.md) are JWTs whose
header declares the algorithm `ed25519-nkey` rather than the registered `EdDSA`.
This identifier is inherited from NATS: the NATS JWT library uses exactly
`ed25519-nkey`. The consequence, shared by both, is that a standards-compliant
JWT library rejects the header outright instead of half-verifying a token it does
not fully understand. That reject-rather-than-half-verify behavior is the
standing reason to keep a distinct algorithm identifier, and valiss gets it by
staying compatible with NATS's choice rather than by inventing one.

**Content-hash token ids.** A valiss `jti` is not random: it is derived by
blanking the id, serializing, hashing, and base32-encoding the digest, so
identical claims always produce the same id. This principle, too, comes from
NATS, whose JWT library computes its id the same way, blanking the id field
first to make the hash repeatable. valiss inherits the idea that a token's id is
a reproducible fingerprint of its own contents.

## What valiss deliberately changes, and why

**There is no NATS server anywhere.** This is the largest divergence, and it
reframes everything else. NATS JWTs exist to authenticate clients to a NATS
messaging server, and the operator/account/user model ultimately authorizes
publish and subscribe on subjects. valiss reuses the *identity* model and throws
away the messaging context entirely: it authenticates requests to your own
services, over HTTP or gRPC or anything else, and there is no `nats-server`, no
subjects, and no connection to NATS in the picture. valiss is a library you link,
not a broker you run.

**A fail-closed allowlist instead of the account resolver.** A NATS server
resolves account JWTs through a resolver (a preloaded in-memory map, a URL
service, or the built-in NATS-based resolver that stores account JWTs in a
directory), and it handles early revocation through revocation lists embedded in
account JWTs. valiss has no server to resolve against, so it replaces that whole
mechanism with a single [allowlist](allowlist.md) predicate the verifier consults
per request, keyed by the content-hash id. The difference is not only mechanical:
the allowlist is an explicit allow-set that fails closed, so a credential absent
from it is rejected, where a resolver is a positive-distribution mechanism paired
with separate revocation state.

**Epochs for domain-wide rotation.** valiss adds an operator
[epoch](rotation.md) counter, carried in a signed operator token, so that
advancing one integer and re-issuing invalidates an entire generation of
credentials at once. NATS expresses revocation differently, per-subject through
account revocation lists, and has no single-counter mass-rotation lever of this
shape. Epochs are valiss's, added because domain-wide re-keying is a first-class
operation for the deployments valiss targets.

**Typed extension claims instead of messaging permissions.** A NATS account or
user JWT carries a fixed vocabulary of messaging permissions and limits, publish
and subscribe rules over subjects. valiss carries generic, self-naming typed
[extensions](extensions.md) that the scheme assigns no meaning to; the HTTP and
gRPC transports are themselves built as extensions. The change follows directly
from having no messaging subjects to permission: authorization in valiss is
whatever typed grants your application defines.

**Two smaller, deliberate divergences.** valiss accepts *only* `ed25519-nkey`,
while NATS also accepts a legacy `ed25519` identifier; the stricter check is
intentional. And although both derive the id by blank-then-hash, the digests
differ: valiss hashes its token payload with SHA-256, while NATS hashes the whole
claims with SHA-512/256. The ids are therefore not interchangeable, and
reproducing a valiss id in another language means reproducing valiss's exact
serialization, not NATS's (see [Tokens](tokens.md)).

## What an nsc user should expect

If you come from `nsc`, a lot will feel familiar. Keys carry the same role
prefixes and seeds, the operator/account/user hierarchy is the one you know, you
issue an account and then issue users beneath it, credentials files are files you
can read and recognize, verification is offline against an operator public key,
and token ids are reproducible content hashes.

What will feel different is everything downstream of "there is no NATS server."
You do not point a `nats-server` at an account resolver; you pin an operator
public key in your own service and load an allowlist file. Authorization is your
own typed extensions rather than subject permissions. Mass rotation is an epoch
counter. And issuance is not `nsc`: the issuer-side tool for holding operator
keys and exporting credentials is the valiss CLI (early development, see
[Custody](custody.md)), which plays the role `nsc` plays for NATS while producing
valiss credentials for valiss verifiers.

## Related

- [Entities](entities.md): the nkey encoding and the role prefixes, in use.
- [Tokens](tokens.md): the JWT shape, the `ed25519-nkey` algorithm, and the
  content-hash `jti`.
- [Creds](creds.md): the credentials-file layout that mirrors NATS `.creds`.
- [Allowlist](allowlist.md) and [Rotation](rotation.md): the revocation and
  rotation levers that replace the NATS resolver and revocation lists.
