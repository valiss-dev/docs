---
title: Reference
weight: 10
---

The [Concepts](/docs/concepts/) and [Guides](/docs/guides/) explain the model
and how to wire it in. This page is the map to the authoritative material
underneath them: the Go API documentation, the wire specification, the
conformance vectors, and the source repositories. Reach for it when you want
an exact signature, a byte-level rule, or the code itself.

## Go API

The reference implementation is the module `valiss.dev/valiss`. Its generated
documentation is the exact API surface: every type, function, and option.

- [valiss.dev/valiss](https://pkg.go.dev/valiss.dev/valiss) - the core: token
  issue and verify, the `Verifier` and `Identity`, the allowlist, keyring,
  replay cache, and message tokens.
- [valiss.dev/valiss/creds](https://pkg.go.dev/valiss.dev/valiss/creds) - the
  credentials-file format: `Format`, `Parse`, `Load`, and the `Creds` struct a
  client authenticates with.

### Credential transports and framework adapters

The `<framework>auth` packages carry credential authentication and grant a
verified identity:

- [contrib/httpauth](https://pkg.go.dev/valiss.dev/valiss/contrib/httpauth) -
  net/http middleware and a signing `RoundTripper`; defines the HTTP extension
  `Ext{Hosts, Methods, Paths}`.
- [contrib/grpcauth](https://pkg.go.dev/valiss.dev/valiss/contrib/grpcauth) -
  gRPC server interceptors and per-RPC credentials; defines the gRPC extension
  `Ext{Methods}`.
- [contrib/ginauth](https://pkg.go.dev/valiss.dev/valiss/contrib/ginauth) - Gin
  middleware over the httpauth verification core.
- [contrib/echoauth](https://pkg.go.dev/valiss.dev/valiss/contrib/echoauth) -
  Echo middleware over the httpauth verification core.

### Message-token transports

The `<framework>sig` packages carry message-token verification and grant no
identity (they expose `MessageFrom`, not an authenticated identity):

- [contrib/httpsig](https://pkg.go.dev/valiss.dev/valiss/contrib/httpsig) -
  net/http message-token middleware and client transport.
- [contrib/grpcsig](https://pkg.go.dev/valiss.dev/valiss/contrib/grpcsig) -
  gRPC message-token interceptors; the checksum binds the deterministic
  protobuf encoding of the message.
- [contrib/ginsig](https://pkg.go.dev/valiss.dev/valiss/contrib/ginsig) - Gin
  message-token middleware.
- [contrib/echosig](https://pkg.go.dev/valiss.dev/valiss/contrib/echosig) -
  Echo message-token middleware.

## Wire specification

- [SPEC-1](/spec/) - the scheme at the byte and algorithm level: the JWS
  envelope, nkey encoding, `jti` derivation, the signing inputs, the
  verification order, and the section 7 reason-code taxonomy. Normative and
  language-independent; the Go source is canonical where the two disagree.

## Conformance vectors

- [Conformance vectors](https://github.com/valiss-dev/spec/tree/main/vectors) -
  language-neutral, verify-side test vectors frozen with spec 1; each JSON file
  in the linked corpus pairs its artifacts with their expected verification
  outcome. An implementation conforms if it runs every vector and gets the
  expected outcome, so a port checks itself against these rather than against the
  Go library directly.

## Repositories

All under the [valiss-dev](https://github.com/valiss-dev) GitHub organization:

- [valiss-go](https://github.com/valiss-dev/valiss-go) - the Go reference
  implementation and the source of truth for the scheme.
- [valiss-py](https://github.com/valiss-dev/valiss-py) - the Python port: creds,
  mint, sign, verify, plus the Django and ASGI integrations.
- [valiss-ts](https://github.com/valiss-dev/valiss-ts) - the
  TypeScript/JavaScript port of the core wire layer.
- [valiss-cli](https://github.com/valiss-dev/valiss-cli) - the planned
  issuer-side CLI (`valiss`) for keys, tokens, creds, and allowlist management;
  early development, its command tree is designed but every command is currently
  a stub, not yet released.
- [spec](https://github.com/valiss-dev/spec) - SPEC-1 and the conformance
  vectors.
- [docs](https://github.com/valiss-dev/docs) - the source of this documentation
  site.
