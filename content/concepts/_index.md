---
title: Concepts
weight: 2
description: "The valiss model, one concept per page: the operator/account/user key hierarchy, tokens, the fail-closed allowlist, extensions, rotation, and custody."
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
- **Extensions** are the typed, signed claims that carry authorization. The
  HTTP and gRPC transports are themselves built on this mechanism.
- **Rotation** is how an operator re-keys or mass-revokes an entire trust
  domain at once, using an epoch counter carried in a signed operator token.

Read them in order the first time. The reference implementation is
[valiss-go](https://valiss.dev/valiss); the byte-level rules referenced here
are fixed by the wire specification.
