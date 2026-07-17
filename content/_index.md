---
title: valiss
---

# valiss

Decentralized tenant authentication for services, built on a three-level
chain of Ed25519 keys: operator, account, user. Verification is offline.
There is no auth service to run, no token introspection endpoint to call,
and issuing credentials never touches production.

## What valiss is

Most multi-tenant services grow a central auth dependency: an OAuth provider,
a session store, a per-tenant key registry, something every request consults
and every deployment keeps alive. valiss inverts that. Trust is a single
public key baked into the server. Everything else is self-contained signed
credentials that verify against that key with no network call.

An **operator** key signs **account** (tenant) tokens. An account key signs
**user** tokens. Every request is signed by the subject's own key over a
timestamp. The server verifies the chain against the pinned operator public
key, checks the account token against a revocation allowlist, and hands the
tenant and user identity to the handler. Delegation is real: revoking one
account cuts off everything under it, and an account can never grant more
than it holds.

## Who it is for

- **Multi-tenant APIs.** Each customer is an account with its own keypair.
  You sign one account credential per customer; they mint their own user and
  service credentials from it, scoped down as they see fit, without asking you.
- **Machine-to-machine auth.** Services authenticate with keys and per-request
  signatures. No shared secrets, no password rotation, and a stolen token is
  useless without the key that signs requests.
- **Edge and on-prem.** Verifiers need only the operator public key, so
  isolated deployments authenticate the same way as connected ones.
- **Browsers and other weak clients.** Short-lived bearer user tokens cover
  clients that cannot hold a key.

## Documentation

### Start

- [Quickstart](/docs/quickstart/): the shortest working setup in Go. Create an
  operator, account, and user, mint a token, and verify it server-side against
  the allowlist.

### Concepts

The model, one page each.

- [Entities](/docs/concepts/entities/): operator, account, user, and how the
  key roles map to the signing hierarchy.
- [Tokens](/docs/concepts/tokens/): the typed claims valiss signs, validity
  windows, and the wire format.
- [Allowlist](/docs/concepts/allowlist/): server-side revocation by token id.
- [Extensions](/docs/concepts/extensions/): signed, typed authorization claims
  carried on the token.
- [Rotation](/docs/concepts/rotation/): epochs and operator tokens for mass
  revocation of a whole trust domain.
- [Messages](/docs/concepts/messages/): message tokens, offline proof of origin
  for artifacts that travel on their own.
- [Creds](/docs/concepts/creds/): the credentials file, a client's tokens and
  signing seed packaged together.

### Guides

Wiring valiss into real code.

- [Go](/docs/guides/go/): the reference implementation, gRPC and HTTP.
- [Python](/docs/guides/python/): the Python client, mint tokens, sign httpx and
  requests calls, and verify server-side at parity with Go.
- [TypeScript](/docs/guides/typescript/): the TypeScript port, sign and verify
  primitives installed from source, with no transport adapter or integrated
  verifier yet.
- [Verifying credentials offline](/docs/guides/verifying/): the language-neutral
  verification recipe for porting to any runtime.

### Reference

The deeper material: exact rules, guarantees, and compatibility.

- [Reference](/docs/reference/): the map to the Go API docs, wire specification,
  and conformance vectors.
- [Security model](/docs/security/): what the offline model protects, and where
  the protection ends.
- [Versioning and compatibility](/docs/versioning/): the wire-spec and library
  version axes, and how cross-language interop is gated.
- [Troubleshooting](/docs/faq/): every verification rejection mapped from the
  symptom you see back to the cause the verifier decided.

The Go library (`valiss.dev/valiss`) is the canonical implementation and is
where the quickstart and the reference guide live today. Other languages do not
depend on Go: Python is a full client library at parity, and TypeScript
currently ships the verification and signing primitives.
