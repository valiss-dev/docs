---
title: Troubleshooting
weight: 7
description: "Every verification rejection mapped from the symptom you see back to the cause the verifier decided, and what to check."
---

valiss verification fails closed, and every rejection has one specific cause.
This page maps the symptom you see (an HTTP status, a gRPC code, an error
string) back to what the verifier decided and what to check. The strings quoted
here come from the Go reference; the Python and TypeScript ports raise
`ValissError` whose `reason` is the same spec section 7 reason code, so the
causes apply to all three. Switch on `reason` rather than matching message text:
the code is stable, the message is illustrative.

The transports translate a verifier failure to a status once. `httpauth` answers
401 for an authentication failure and 403 for an extension denial; `grpcauth`
maps the same split to `Unauthenticated` and `PermissionDenied`. The response
body or status message carries the underlying cause.

## The client is rejected 401 but does send a token

**Missing credentials.** Symptom: 401 with `missing credentials` (gRPC
`Unauthenticated: missing credentials`). It means neither the account nor the
user token header reached the server, so the middleware rejects before
verifying anything. Check that the client uses a signing transport
(`httpauth.NewTransport`, `grpcauth.NewCredentials`, or the Python and
TypeScript auth hooks), and that no proxy strips the `valiss-account-token` and
`valiss-user-token` headers.

**Account token not recognized.** Symptom: 401, `account token not recognized`
(reason `not_allowlisted`). The account token verified against the pinned
operator key, but its `jti` is not in the server's allowlist, which fails
closed. Check that the `jti` the minter printed is the one loaded, that the
allowlist file was reloaded after any re-issue (issuing with different claims
produces a different `jti`), and that you did not mean to run `AllowAll`.

**Wrong operator pinned.** Symptom: 401, `account token not signed by the
expected issuer`. The account token was signed by a different operator than the
public key the server pinned. Check that the operator public key passed to
`NewVerifier` is the one that issued these account tokens, and in a
multi-operator deployment that the credential's trust domain is in the keyring.

## A token is expired or out of its window

Symptom: 401 with `account token expired`, `user token expired`, `account token
not yet valid`, or `operator token expired: the trust domain is closed`. A
token's `exp` or `nbf` fell outside the skew window (2 minutes by default); an
expired operator token closes the whole trust domain until a fresh one is
published. Check the clocks on the issuing and verifying sides, the TTL you
issued with, and whether an operator-token rotation ceremony was allowed to
lapse. `WithSkew` widens the tolerance when drift is unavoidable.

## Epoch mismatch after a rotation

Symptom: 401 with `account token epoch N, trust domain epoch M`, the same for a
user token, or `no trusted operator <key> at epoch N` (reason `epoch_mismatch`
or `unknown_operator`). The verifier enforces an operator token or keyring at
one epoch and the presented token carries another. This is the intended outcome
of an epoch bump: every earlier-epoch token is rejected cryptographically. To
resolve it, re-issue the account and user tokens at the current epoch. For a
grace period, register both the outgoing and incoming epochs in a `Keyring` so
the old epoch keeps verifying until its operator token's `exp` lapses. Unstamped
tokens are epoch 0.

## The request signature does not check out

**Signature required, not a bearer token.** Symptom: 401, `request signature
required: not a bearer token` (reason `not_bearer`). The request carried no
per-request signature and the effective token is not a bearer user token, so
possession is unproven. Check that the client creds carry a seed and sign
through the transport, or, for a client that cannot hold a key, issue the user
token with `WithBearer()` and pair it with TLS.

**Signature verification failed.** Symptom: 401, `request signature verification
failed` (reason `bad_request_signature`). The Ed25519 check over the
reconstructed request context failed, almost always because the server rebuilt
different context bytes than the client signed. The signature binds the method,
host, and path exactly. Check that no proxy rewrote the `Host` header or the
path, and that the host the client signed is the one the wire carries (an
explicit `Host` header wins, otherwise the URL host keeping any non-default
port). A captured signature cannot be reused across a different method or path.

**Timestamp outside the skew window.** Symptom: 401, `request timestamp outside
the 2m0s skew window` (reason `skew`). The client's request timestamp is too far
from the server's clock, or is unparsable. Check client clock sync, or widen the
verifier with `WithSkew`.

**Nonce required, or replay.** Symptom: 401, `request nonce required` or
`request nonce already seen (replay)`. The verifier runs a replay cache. The
first says the signed request carried no nonce; the second says the nonce was
already spent. Enable the nonce on the client (`httpauth.WithNonce()`, or
`nonce=True` in Python), and make each retry use a fresh nonce rather than
resend the identical request.

## The client is denied 403 (PermissionDenied)

**Request outside the extension.** Symptom: 403, `token does not permit GET
/admin`, or the gRPC form `token does not permit /pkg.Svc/Method`.
Authentication succeeded, but the request falls outside the token's HTTP or gRPC
extension bounds. Check the `Methods`, `Paths`, and (HTTP) `Hosts` on the token.
Each dimension is an AND-filter that constrains only when populated, and paths
take a trailing `*` prefix wildcard. Extensions on both the account and the user
token are enforced together, so an account-level cap bounds every user beneath
it.

**Token carries no extension.** Symptom: 403, `token carries no http extension`
or `token carries no grpc extension`. Transport enforcement fails closed, and
a token somewhere in the chain does not carry the transport extension at all.
Issue every token in the chain with the extension (the zero-value `Ext{}` grants
nothing; allow-all is the explicit wildcard `[]string{"*"}`). Only when
authorization lives entirely outside the transport, build the middleware with
`AllowMissingExtension()`.

## User-only credentials are rejected

Symptom: 401, `request carries no account token and the server has no account
token resolver` (reason `no_resolver`). The client sent only a user token and
the server cannot supply the matching account token. Configure
`WithAccountTokenResolver` (for example `StaticAccountTokens(...)`), or have the
client send a bundle that embeds the account token.

## A credentials file will not load

Symptom: `creds: no token markers found`, `creds seed: ...`, `creds token:
...`, or `creds: unsupported version N`. The file is not a valiss credentials file, a
token or seed block is corrupt, or it declares a format version this build does
not implement. Check that the file is the one the minter wrote, whole and
untruncated, that the seed and token blocks are intact base32, and that the
library is not older than the format that produced the file.

## Issuing a token errors

Issuance validates the key roles up front, so these fire at `Issue*`, not at
verify time:

- `account tokens must be signed by an operator-type nkey (expected an SO...
  seed)`, or `user tokens must be signed by an account-type nkey (expected an
  SA... seed)`: the signing key is the wrong level. The signer must sit exactly
  one level above the subject.
- `invalid tenant public key (expected an A... nkey)`, or `invalid user public
  key (expected a U... nkey)`: the subject public key is the wrong role.
- `audience, checksum, and chain apply only to message tokens`: `WithAudience`,
  `WithChecksum`, or `WithChain` was passed to an identity issuer.
- `message tokens must carry an expiry (WithTTL or WithExpiry)`: message tokens
  are short-lived by rule and cannot be issued without one.
- `duplicate extension "name"`: two extensions with the same `ExtensionName` on
  a single token.

## A message token will not verify

A message token is a proof of origin, verified offline; a `Verifier` never
accepts one as a credential. Its own failures:

- `missing message token` (httpsig, 401), or `message token chain required`
  (401): the receiver got no message token, or not the provenance chain it
  needs.
- `payload checksum mismatch` (reason `checksum_mismatch`): the body the
  receiver hashed differs from the one the emitter signed. Anything that
  re-encodes the payload in transit breaks it. For gRPC the checksum is over the
  deterministic protobuf encoding, so both ends must share the same message
  definition; a schema drift or a mismatched `google.golang.org/protobuf`
  produces different bytes and fails here.
- `message token audience "...", expected "..."` (reason `wrong_audience`): the
  token was issued for a different destination, or carries none while the
  receiver expects one. Set `ExpectAudience` to the destination you verify for.
- `message token expired`: a stored message was verified at `now` instead of its
  receipt instant. Pass `valiss.At(receivedAt)`.
- `message token carries no chain and none was supplied (WithChainTokens)`
  (reason `no_chain`): issue with `WithChain` to embed the provenance, or supply
  it out of band on the verify side with `WithChainTokens`.
- `message checksum requires a proto.Message, got T` (grpcsig): the value handed
  to the gRPC message signer is not a protobuf message.

## A token decodes but is rejected outright

- `unsupported wire version N`, or `unsupported token type`: the token comes
  from a wire version or envelope this build does not implement. `ver` is read
  from the header; the current version is 1.
- `token signature verification failed`, or `malformed token`: the token bytes
  are corrupt or were re-serialized. The signing input is the header and payload
  exactly as they appear on the wire, so a reparsed-and-re-encoded token no
  longer verifies.
