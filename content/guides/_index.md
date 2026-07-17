---
title: Guides
weight: 3
description: "Integration guides for valiss: issuing credentials and verifying offline in Go, Python, and TypeScript, plus the language-neutral recipe."
---

Integration guides for building on valiss: issuing credentials, wiring
server-side verification into your transport or framework, and verifying
tokens offline.

valiss is offline tenant authentication built on a three-level chain
of Ed25519 keys (operator, account, user), with an optional fourth level of
per-message proof-of-origin tokens. Trust is a single public key pinned in
the server; everything else is self-contained signed credentials that verify
offline, with no auth service to run.

Start with the guide for your stack:

- [Go](go.md) - install the module, issue tokens, verify requests
  server-side, and wire the HTTP, gRPC, Gin, and Echo integrations.
- [Python](python.md) - the Python client: load credentials, sign httpx and
  requests calls, and verify tokens and requests server-side.
- [TypeScript](typescript.md) - the TypeScript port: sign and verify
  primitives, installed from source, with transport adapters still to come.
- [Verifying tokens](verifying.md) - the offline decode-and-verify recipe,
  and the porting reference for implementing verification in another
  language.
