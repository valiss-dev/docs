---
title: valiss
---

valiss is decentralized tenant authentication for services, built on a
three-level chain of Ed25519 keys: operator, account, user. Verification is
offline against a single pinned public key, so there is no auth service to run,
no token introspection endpoint to call, and issuing credentials never touches
production.
