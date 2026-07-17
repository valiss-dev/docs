---
title: Creds
weight: 7
---

A [token](tokens.md) proves a delegation, but a client needs more than a token
to authenticate: it needs the token *and* the private key that signs its
requests. A **creds file** is those two things packaged together in one
text file, everything a client holds and nothing the server does. Where the
server keeps only public keys and an allowlist, the client keeps its creds.

The `valiss.dev/valiss/creds` package renders and parses this file. `Format`
turns a `Creds` value into the file text, `Parse` reads the text back, and
`Load` reads and parses a path in one call. On the issuer side, the valiss CLI
(early development) is designed to write these files directly with `creds
export`, so an operator distributes credentials without coding against the
package. That command is not yet runnable (a stub today); until it lands, an
operator produces creds files with the `creds` package or `examples/minter`,
which writes a creds file straight from a manifest.

## The marker-delimited layout

A creds file is line-oriented and delimited by begin and end markers, the same
shape NATS uses for its own credentials so the format is familiar and
copy-paste-safe. A version header opens the file, each artifact sits between a
matched pair of markers, and a seed carries a plain-language warning after it:

```
VALISS-CREDS-VERSION: 1

-----BEGIN VALISS ACCOUNT TOKEN-----
<account token>
------END VALISS ACCOUNT TOKEN------

-----BEGIN VALISS SEED-----
<seed>
------END VALISS SEED------

************************* IMPORTANT *************************
Seed lets anyone sign as this identity. Keep it secret.
```

The version header versions the file container only; the tokens inside carry
their own wire version. Parsing is deliberately strict. Each present section
must hold exactly one payload line between its markers, and a truncated,
unclosed, or multi-line section is a parse error rather than a confusing
cryptographic failure downstream. At least one token must be present. An
unrecognized file version is rejected outright instead of being mis-read. The
file fails closed at the door.

## What each kind carries

A `Creds` value has three fields, `AccountToken`, `UserToken`, and `Seed`, and
the combination present decides what kind of creds it is:

| kind              | account token | user token | seed         | signs requests as |
| ----------------- | ------------- | ---------- | ------------ | ----------------- |
| account-level     | yes           | no         | account seed | the account       |
| user-level (lean) | no            | yes        | user seed    | the user          |
| bundle            | yes           | yes        | user seed    | the user          |
| bearer            | one token     | maybe      | none         | nothing           |

**Account-level** creds authenticate a tenant directly: the operator-signed
account token and the account seed that signs its requests. **User-level** creds
authenticate a delegated user: the account-signed user token and the user seed.
A **bundle** is user-level creds that additionally carry the upstream account
token, so the receiver has the whole chain in hand. **Bearer** creds carry
tokens only and no seed; their holder cannot sign, and a server accepts them
only when the effective token is a bearer user token (see
[`WithBearer`](../guides/go.md#issue-options)).

The seed is always the private key of the creds' *subject*: the account seed in
account-level creds, the user seed in user-level ones. It signs as exactly that
identity and never a level above.

## Lean creds versus bundles

A user-level creds file has a choice to make about the account token. The chain
that verifies a user token needs its account token, but that token is the same
for every user under a tenant. Shipping it in every user's file is redundant.

**Lean** user-level creds omit the account token and let the server supply it. A
`Verifier` configured with an account-token resolver looks the account token up
by the account key named in the user token and runs it through the full
verification (operator signature, expiry, allowlist) per request:

```go
resolver, err := valiss.StaticAccountTokens(acctToken1, acctToken2)
verifier := valiss.NewVerifier(operatorPub, allowlist,
    valiss.WithAccountTokenResolver(resolver))
```

`StaticAccountTokens` indexes a fixed set of account tokens by their subject key
from server configuration; `WithAccountTokenResolver` installs the lookup.
Without a resolver, a request carrying only a user token is rejected, because
the server has no way to complete the chain.

A **bundle** makes the opposite choice: it embeds the account token so the
credential is self-sufficient and the server needs no resolver. This is the
shape the message-token transports require, since an emitter mints
[message tokens](messages.md) over its own chain and must therefore hold both
tokens plus the seed. Lean creds keep the tenant's account token in one place on
the server; bundles keep each client self-contained. Pick by where you would
rather hold the account token.

## Custody

The account token and user token are public artifacts: they are signed, they
verify against public keys, and nothing breaks if a receiver sees them. The
**seed is the secret**. It is the private key, and anyone who holds it can sign
as the identity, which is why `Format` appends the plain warning to any file
that contains one. Treat a creds file with a seed as you would any private key:
it is not a token, it is the ability to *be* the subject.

The library keeps custody one-directional. A seed lives only on the signing
side; the server holds an operator public key and an allowlist and never a seed
at all, so a verifier compromise cannot leak signing power. Bearer creds are the
deliberate exception: they carry no seed, so they cannot sign, and in exchange a
bearer token is replayable by whoever holds it until it expires. That trade is
only safe under TLS and a short validity window. The rest of the time, the seed
in the creds file is the thing to guard.
