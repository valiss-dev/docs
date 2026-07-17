---
title: Concepts
weight: 2
description: "The valiss model, one concept per page: the operator/account/user key hierarchy, tokens, the fail-closed allowlist, rotation, extensions, messages, creds, custody, and its NATS lineage."
---

valiss is offline tenant authentication: a server can authenticate a
request and attribute it to a tenant and an end user knowing only one pinned
public key, with no network round-trip to an issuer. Everything a request
carries is a self-contained signed credential that verifies offline. There is
no auth service to run and no introspection endpoint to call.

This section explains the ideas you need before wiring valiss into a service,
in the order they build on each other:

- **Entities** are the Ed25519 keypairs that make up the trust hierarchy:
  operator, account (tenant), user, and the optional per-message level.
- **Tokens** are the signed artifacts that bind those keys into a delegation
  chain. Each token is signed by the key one level up, and its identifier is a
  hash of its own contents.
- **The allowlist** is how issued account tokens are revoked before they
  expire. It fails closed: a token absent from the list is rejected.
- **Rotation** is how an operator re-keys or mass-revokes an entire trust
  domain at once, using an epoch counter carried in a signed operator token.
- **Extensions** are the typed, signed claims that carry authorization. The
  HTTP and gRPC transports are themselves built on this mechanism.
- **Messages** are the optional per-message proofs of origin a user key mints
  for the artifacts it emits, verifiable offline long after the request that
  produced them.
- **Creds** are the token-and-seed files a client holds to authenticate:
  everything the signing side keeps, and nothing the server does.
- **Custody** is the question of who holds a subject's seed and tokens, and the
  gap valiss has there today.
- **Lineage** traces what valiss inherits from the NATS nkey and creds formats,
  and where it deliberately diverges.

Read them in order the first time. The reference implementation is
[valiss-go](https://valiss.dev/valiss); the byte-level rules referenced here
are fixed by the wire specification.
