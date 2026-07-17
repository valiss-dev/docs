---
title: TypeScript
weight: 3
---

# TypeScript integration

`valiss-ts` is the TypeScript/JavaScript implementation of the valiss scheme:
Ed25519 nkeys, token mint and verify, credentials files, request signatures, and
offline message-token verification. It implements wire spec 1 and is
byte-for-byte wire-compatible with the Go reference and the Python port.

This is the core wire layer. It is enough to load credentials, sign requests,
and verify tokens and signatures. It does not yet ship the transport or
framework adapters or the integrated request verifier; see
[Current limitations](#current-limitations) before you build on it.

## Requirements

Node.js 20 or newer, or any runtime with WebCrypto Ed25519
(`globalThis.crypto.subtle`): modern browsers, Deno, and Bun. There are no
runtime dependencies.

## Install

The package is not published to npm yet. Install it from source, for example as
a git dependency:

```json
{
  "dependencies": {
    "valiss": "github:valiss-dev/valiss-ts"
  }
}
```

The published entry point emits ESM plus type declarations. In a checkout,
`npm install && npm run build` produces `dist/`; `npm test` runs the unit suite
and the spec-1 conformance vectors.

The library is ESM-only. Import from the package root:

```ts
import { nkeys, parseCreds, signRequest, verifyUser } from "valiss";
```

## Load credentials

A credentials file holds a subject's tokens plus the seed that signs its
requests. The library parses file *contents*; read the file with your runtime's
own API and hand the string to `parseCreds`:

```ts
import { readFileSync } from "node:fs";
import { parseCreds, signerOf } from "valiss";

const creds = parseCreds(readFileSync("alice.creds", "utf8"));
const signer = await signerOf(creds); // KeyPair, or undefined for bearer creds
```

A parsed `Creds` has three fields: `accountToken` (operator-signed),
`userToken` (account-signed), and `seed`. `signerOf` returns the signing key
pair, or `undefined` when the credentials carry no seed (bearer credentials,
which cannot sign requests). `formatCreds` renders a `Creds` back to file text.

## Authenticate client requests

The library signs requests; it does not yet attach the signature to an HTTP
client for you. Build the signature with `signRequest` and set the headers
yourself.

`signRequest` signs a timestamp bound to a canonical *request context* string.
For HTTP the context is `http`, the method, the host, and the path, each
followed by a newline, with a final field for the replay nonce (empty when
replay suppression is off):

```ts
import { readFileSync } from "node:fs";
import { parseCreds, signRequest, signerOf } from "valiss";

const creds = parseCreds(readFileSync("alice.creds", "utf8"));
const signer = await signerOf(creds);
if (signer === undefined) throw new Error("bearer creds cannot sign");

const method = "GET";
const host = "api.example.com";
const path = "/v1/whoami";
const context = `http\n${method}\n${host}\n${path}\n`;

const { timestamp, signature } = await signRequest(signer, context);

await fetch(`https://${host}${path}`, {
  method,
  headers: {
    "valiss-account-token": creds.accountToken,
    "valiss-user-token": creds.userToken,
    "valiss-timestamp": timestamp,
    "valiss-signature": signature,
  },
});
```

The server must reconstruct identical context bytes, so the host must be the one
the wire carries and the method and path must match exactly. The signature binds
the context: a captured signature cannot authorize a different method or path.
Build a fresh signature per request.

For a server with a replay cache, mint a nonce, append it as the last context
field, and send it in the `valiss-nonce` header:

```ts
import { newNonce } from "valiss";

const nonce = newNonce();
const context = `http\n${method}\n${host}\n${path}\n${nonce}`;
const { timestamp, signature } = await signRequest(signer, context);
// ...also send { "valiss-nonce": nonce } in the request headers.
```

These are the same primitives a future `valiss/fetch` adapter will wrap, so a
transport hook you build on `signRequest` today keeps working when the adapter
ships.

## Verify tokens and requests

The library verifies each token in the chain and the request signature. There is
no integrated request verifier yet (see below), so a server composes the checks
itself, walking the chain from the pinned operator key downward.

Each `verify*` function checks a token's signature, type, and issuer and returns
its claims. It does not check expiry, activation, or the allowlist; do those
against the returned claims:

```ts
import { verifyAccount, verifyUser, verifySignature, expired } from "valiss";

// Pin the operator public key as the trust anchor.
const account = await verifyAccount(accountToken, operatorPubKey);
const user = await verifyUser(userToken, account.subject);

if (expired(account) || expired(user)) throw new Error("token expired");

// Rebuild the same context the client signed, then check the signature.
const context = `http\n${method}\n${host}\n${path}\n`;
await verifySignature(user.subject, timestamp, signature, context);
```

`verifySignature` binds the timestamp to a symmetric skew window
(`DEFAULT_SKEW_MS`, two minutes) and confirms the signature was made over the
context. `verifyOperator` verifies the self-signed operator token when you want
to enforce the trust domain's epoch and validity window. `expired` and
`notYetValid` apply the window checks to any claims.

### Verifying a message token

A message token is a self-signed proof of origin (a webhook body, for example),
verified offline against the operator key. It is a proof, never a credential:

```ts
import { verifyMessage } from "valiss";

const claims = await verifyMessage(proof, operatorPubKey, {
  audience: "https://api.example.com/ingest",
  payload: body,
});
```

`verifyMessage` walks the operator, account, user, and message chain, requires
every level to agree on the epoch, checks each validity window, and enforces the
audience and checksum bindings you request. `checksum(payload)` computes the
lowercase-hex SHA-256 a message token embeds.

## Errors

Every failure throws `ValissError`. Its `reason` property is the stable spec
section 7 reason code (a `Reason` enum member), so a server can map a failure to
a status code without matching on the message text.

## Current limitations

The TypeScript port is the core wire layer. Compared with the Python client,
these are not present yet (they are planned as additive subpath exports such as
`valiss/fetch` and `valiss/express`):

- No client transport adapter. There is no `fetch`, axios, or other HTTP-client
  hook; attach the signed headers yourself, as shown above.
- No integrated request verifier. There is no allowlist, keyring, resolver, or
  replay-cache verifier that turns request headers into a verified identity in
  one call. Compose `verifyAccount`, `verifyUser`, and `verifySignature`
  yourself.
- No credentials file loader. `parseCreds` takes file contents; read the file
  with your runtime's own API.
- No exported header-name or request-context constants. The header names
  (`valiss-account-token`, `valiss-user-token`, `valiss-timestamp`,
  `valiss-signature`, `valiss-nonce`) and the context format are wire-spec
  constants you write out by hand.

Two documented wire edge cases fail closed rather than round-trip: an `epoch`
above `Number.MAX_SAFE_INTEGER`, and a token `name` containing a `\b` or `\f`
control character (which would derive a different content id than Go). Neither
arises from anything the spec's claims produce today.
