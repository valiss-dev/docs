---
title: Security model
weight: 5
---

# Security model

valiss verifies credentials offline against a single pinned public key. That is
its strength and it is also the shape of its threat model: there is no issuer to
consult at verification time, so what a server can and cannot defend against
follows entirely from the keys it holds, the tokens it is shown, and the state
it keeps locally. This page is honest about both. It says what the library
protects, and it says plainly where the protection ends.

Nothing here is a configuration secret, and every claim is a property of the
public source. Where a guarantee depends on an option you must set, that is
called out, because an unset lever protects nothing.

## The trust anchor

A server pins one value: the operator public key. Everything else verifies
against it with no network call. `NewVerifier(operatorPub, allowlist)` is the
whole trust configuration. The verifier walks each credential from the pinned
key down (operator signed the account token, the account signed the user token)
and checks the signature and the key role at every hop.

Because the anchor is a public key, it is not a secret. The corresponding
**operator seed is the ultimate secret in the system.** Whoever holds it can
mint account tokens for the entire trust domain, and through them any user
beneath. The seed never belongs on a server; it lives with the issuer, in a
secrets manager, and touches production only to sign new credentials.

A stolen operator seed is therefore the worst case, and it is worth being
precise about what it buys and what still contains it:

- For request authentication it is partly contained. A forged account token has
  a fresh content-hash id (see [Tokens](/docs/concepts/tokens/)) that is not on
  any server's [allowlist](/docs/concepts/allowlist/), and an allowlist accepts
  only ids you deposited. An attacker cannot add to your allowlist. So a stolen
  operator seed does not, by itself, admit a new tenant to an allowlist-enforcing
  server.
- For offline message verification it is not contained. `VerifyMessage` roots
  trust in the operator key alone and holds no allowlist, so a stolen operator
  seed forges provenance that offline receivers accept.
- Epoch [rotation](/docs/concepts/rotation/) does not contain it either: the
  holder of the seed can re-mint at whatever epoch you advance to.

The only full remedy for operator-seed compromise is pinning a new operator
public key everywhere. Rotation and the allowlist are for the credentials
beneath the anchor, not for the anchor itself.

## What a stolen credential buys

valiss separates two things a naive scheme conflates: the **token**, a public
signed artifact that names public keys, and the **seed**, the private key that
signs. By default a token authorizes nothing on its own. A request is authentic
only if it is signed, per request, by the subject's seed (`SignRequest` on the
client, `VerifySignature` on the server). This is proof of possession: capturing
a token off the wire does not let you act as its subject, because you cannot
produce the signature.

The consequences differ sharply by what an attacker steals and at which level:

| stolen         | what it grants                                                                                       |
| -------------- | ---------------------------------------------------------------------------------------------------- |
| operator token | nothing secret. It is a self-signed public policy statement (epoch, validity window).                |
| operator seed  | full domain compromise. Mint any account, and any user beneath it. See the anchor section above.     |
| account token  | nothing on a signing server. Account-level requests must always sign, so the token alone cannot act. |
| account seed   | full tenant takeover: sign as the tenant and mint user tokens, bounded by the account's own scope.   |
| user token     | nothing, unless it is a bearer token. A signed request still needs the user seed.                    |
| user seed      | act as that user, within its bounds.                                                                 |
| bearer token   | direct use as that user until the token expires or its account leaves the allowlist.                 |

Two properties bound the damage even when a seed is stolen. Delegation cannot
widen: an account seed can mint users, but never more scope than the account
itself holds, and an account can never mint another account. And revoking the
account cuts off every user beneath it in one edit. A stolen account seed is a
tenant-level breach, not a domain-level one.

## Bearer tokens versus signed requests

The default is proof of possession, and it is what makes a captured token inert.
For clients that cannot hold a key at all, typically browsers, a user token may
be marked **bearer** with `WithBearer`. The server then accepts it without a
per-request signature: possession of the token *is* the proof.

That is a real and deliberate weakening, scoped as narrowly as the design
allows:

- Only user tokens may be bearer. Account-level requests always sign.
- A bearer token is a bearer secret. Anyone who captures it can use it until it
  expires or its account is removed from the allowlist. Pair bearer tokens with
  TLS so they are not exposed in transit, and give them a short validity window
  so a captured one ages out quickly.

There is a second reason to prefer signed requests. The possession check runs
*before* any consumer-supplied extension check or validator, so a party that
merely captured a token but cannot sign never triggers that work. A bearer token
waives the signature by design, and with it that ordering guarantee.

## Replay

The request signature covers a timestamp bound to a SHA-256 of the request
context, the transport's canonical description of the operation (method and
path, with a nonce folded in when one is used). Two things follow directly:

- **Binding the context stops substitution.** A signature captured for one
  operation cannot authorize a different one, because the bytes it signed
  include that operation's context.
- **The timestamp bounds the window.** The verifier accepts a timestamp only
  within a symmetric skew window around now (`DefaultSkew`, two minutes,
  overridable with `WithSkew`).

Be clear about the gap this leaves. Within that window, the identical signed
request can be replayed verbatim: the signature is still valid and the timestamp
is still fresh. **The library does not deduplicate requests by default.**

Suppressing replay is opt-in. `WithReplayCache`, together with a per-request
nonce (128 random bits) folded into the signed context, makes the verifier
reject a nonce it has already seen, for a retention of twice the skew window (the
longest a replay could still land inside a valid timestamp). The limit to be
honest about: the built-in `MemoryReplayCache` is process-local. Across several
server instances a replay could land on an instance that has not seen the nonce.
Exactly-once across instances requires backing the cache with shared storage.

Message tokens have their own replay surface and their own levers, both
receiver-side and both off unless set. Cross-destination replay is closed by
binding a token to its destination (`WithAudience`, checked with
`ExpectAudience`); payload tampering is closed by binding it to the payload bytes
(`WithChecksum`, checked with `WithPayload`). A receiver that knows its own
identity should set the audience check; a receiver that requires payload binding
should set the checksum check.

## Fail-closed enforcement

Two enforcement points default to denial rather than permission.

**The allowlist.** An account token is accepted only if its id is present in the
server's allowlist. An unlisted token is rejected even with a flawless signature
and a far-future expiry. The list is an explicit set of ids you deposited, not a
denylist of revoked ones, so nothing is trusted by default. The development-only
`AllowAll` accepts every id and turns revocation off entirely; it is not for
production. See [The allowlist](/docs/concepts/allowlist/).

**Transport extensions.** The HTTP and gRPC integrations authorize through signed
[extensions](/docs/concepts/extensions/), and they fail closed. Every token in
the chain must carry the transport extension, the zero-value extension grants
nothing, and allow-all is the *explicit* wildcard, never an unset field. One
subtlety is worth internalizing: an empty dimension imposes no restriction, so
`Ext{Paths: ["/admin/*"]}` permits those paths under *any* method. To bound a
surface you must name every dimension you mean to bound. A deployment that
authorizes entirely outside the transport can opt out with
`AllowMissingExtension`, but that is a deliberate choice, not an accident.
Enforcement also compounds down the chain, so an account-level extension caps
every user the account mints.

The verifier applies these in a fixed order and reports a precise reason:
chain signatures, then epoch and validity windows, then the allowlist, then
proof of possession, then extension checks, then custom validators.

## Revocation reach, and its limits

Revocation without an issuer to call is bounded, and the boundaries matter.

- **Selective revocation is an allowlist edit.** Remove an account's id and the
  next request carrying that account token fails, along with every user beneath
  it, because a user token is only valid under a listed account. No token needs
  to have expired.
- **Reach is local to each verifier's copy of the list.** There is no central
  push. A revocation takes effect on a given server only once that server's
  allowlist is updated and reloaded (`LoadAllowlistFile` and reload on change).
  Revocation is as prompt as your distribution of the list, no more.
- **The allowlist keys accounts, not users.** There is no per-user allowlist
  entry, so you cannot revoke a single user through the allowlist without
  revoking its account. To cut off one user, let its (short) TTL lapse, or
  revoke the whole account and re-mint the rest.
- **Domain-wide revocation is epochs.** Bump the operator epoch and re-mint;
  every token from an earlier epoch is rejected on next use, and the operator
  token's own expiry bounds the whole domain. See [Rotation](/docs/concepts/rotation/).
- **None of it revokes the operator seed.** As above, a leaked operator seed is
  outside the reach of both levers; only re-pinning the anchor cures it.

Because expiry is checked with the same skew slack as the request timestamp, a
token is honored up to the skew past its stated expiry. Size validity windows
with that two-minute grace in mind.

## Scope and pitfalls

- **valiss authenticates; it does not encrypt.** It verifies identity and
  carries signed authorization claims. It provides no transport confidentiality.
  Use TLS for that, and to protect bearer tokens in transit.
- **Key custody is yours.** The model's security rests on seeds staying secret.
  Storing and distributing them is the issuer's responsibility and outside the
  library.
- **Never authorize on names.** A token's `name` is an issuer-asserted label,
  not checked for uniqueness and free to collide across operators. Key
  authorization decisions on public keys and extensions, not on names. See
  [Entities](/docs/concepts/entities/).
- **The nkey role prevents cross-level confusion.** Because the key role travels
  in the key material, a verifier checks at every hop that an operator key signed
  the account token and an account key signed the user token. A user key
  masquerading as an account cannot even be expressed.

## Where to go next

- [The allowlist](/docs/concepts/allowlist/): revocation semantics in depth.
- [Rotation](/docs/concepts/rotation/): epochs and the keyring for domain-wide
  rotation and grace periods.
- [Extensions](/docs/concepts/extensions/): the authorization mechanism the
  transports enforce.
- [Verifying credentials offline](/docs/guides/verifying/): the verification
  algorithm every implementation follows.
