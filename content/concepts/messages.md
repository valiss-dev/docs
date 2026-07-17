---
title: Messages
weight: 6
---

[Entities](entities.md) names a fourth, optional level below the user: a proof
a user key mints for a single artifact it emits. [Tokens](tokens.md) shows how
that message token sits at the bottom of the signing hierarchy, self-signed
(`iss == sub`) by the user key. This page is about what that level is *for*:
attaching a verifiable origin to a message that travels on its own, long after
the request that produced it has ended.

A per-request credential answers "who is calling this server right now?" It
lives for the length of one connection and proves possession of a key against a
live verifier. A webhook, a queue message, or an exported document has no such
moment. It is produced now, delivered later, and read by a party that never
spoke to the sender. The **message token** is the proof that rides along with
it: a short-lived, self-signed claim that binds the payload and its destination
to the user that emitted them, verifiable offline by anyone holding the operator
public key.

## A proof, not a credential

This is the distinction that separates the message level from the three above
it. A user token is a *credential*: it delegates authority, and its holder
signs live requests with the matching seed (or, for a bearer token, presents it
directly). A message token is a *proof of origin*: it makes a statement about one
specific payload sent to one specific destination, and possessing it grants
nothing at all.

The library enforces the asymmetry. `Verifier.VerifyRequest`, the entry point
for per-request credentials, never accepts a message token; a receiver that
verifies a message token must not treat it as a bearer credential. Capturing a
message token in flight lets an attacker prove that a message *was* emitted, no
more. It does not let them act as the user, reach the server, or forge a
different payload.

To keep that exposure small, a message token must carry an expiry: `IssueMessage`
refuses to mint one without a validity window, and the contrib transports default
to a 30-second window (`valiss.DefaultMessageTTL`). An eternal proof of origin
only widens the window in which a captured token still verifies.

## What a message token binds

A message token is minted from the emitter's user key and pins the message down
along several axes at once:

- **The payload**, through a SHA-256 checksum (`valiss.Checksum`). A receiver
  that re-hashes the bytes it actually received and compares closes payload
  tampering.
- **The destination**, through an audience (`aud`). A token minted for one
  receiver does not verify at another, which closes cross-destination replay.
- **The epoch**, so a message is bound to the same trust-domain epoch as the
  credentials that authorized its emitter (see [Rotation](rotation.md)).
- **The provenance chain**: the account and user tokens that delegate down to
  the signing key, so a receiver can walk operator to account to user to message
  from the operator key alone.

The emitter chooses those bindings at mint time; the receiver decides which of
them to insist on at verification time. The [Go guide](../guides/go.md#message-tokens)
covers the `IssueMessage` and `VerifyMessage` calls; [Verifying tokens](../guides/verifying.md#verifying-a-message-token-full-chain)
gives the byte-level algorithm for porting to another language.

## How keyrings verify message-level signatures

Verifying a message token is a whole-chain check, exactly as it is for a
per-request credential: the receiver walks operator to account to user to
message, checks each signature and role, requires every level to agree on the
epoch, and checks every validity window at the verification instant. A stored
message verifies as of its receipt instant, not now, so a token that has since
expired still checks out against the moment it arrived.

A receiver that trusts a single trust domain roots the walk in one pinned
operator public key. A receiver that trusts several roots it in a
[keyring](rotation.md#grace-periods-with-a-keyring) instead. The chain names its
own domain: the account token's issuer and epoch select exactly one keyring
entry, so the receiver never trials keys. A message whose chain names an unknown
operator, or a known operator at an unregistered epoch, fails immediately. The
matched entry also carries policy that is always enforced, its validity window
and its exact epoch, and the same entry surfaces on the verified claims so a
handler that serves several domains can tell them apart.

Offline receivers hold no allowlist, so a verified message token is not subject
to per-tenant revocation on its own. A receiver that wants revocation checks the
chain account token's id against its own allowlist (see [Allowlist](allowlist.md)),
just as a request verifier does.

## Message-token transports

Two contrib packages carry message tokens over the wire, mirroring the
credential transports but proving origin rather than authenticating a caller.
Per the naming split, the `sig` suffix marks message-token verification that
grants no identity, against the `auth` suffix that grants one:

- **`contrib/httpsig`** attaches a freshly minted token to each outgoing HTTP
  request and verifies it on the receiving side, binding the audience to the
  request's host and path and the checksum to the body.
- **`contrib/grpcsig`** does the same per gRPC call, binding the checksum to the
  request message's deterministic protobuf encoding.

A handler reads the verified message claims from its context with
`valiss.MessageFromContext`, which returns claims that prove origin and
explicitly are not an identity. Because origin and caller-authentication are
separate questions, a route that needs both pairs a `sig` transport with the
matching `auth` one.

## Chain negotiation

A self-contained message token embeds its whole provenance chain, so a receiver
verifies it with nothing but the operator key. That self-containment costs
bytes: the account and user tokens ride along with every single message.

Chain negotiation trades those repeated bytes for a one-time handshake. The
emitter sends a chainless token; a receiver that cannot resolve the chain
answers with a "chain required" signal; the emitter retransmits once with the
chain attached in detached headers. A receiver that keeps a chain cache
remembers the emitter's chain after the first exchange, so the steady state is a
bare token per message and the retransmit happens once per emitter rather than
once per message. `valiss.ErrNoChain` is the one verification failure a
retransmit can cure, which is what lets the transports drive this automatically.
The [Go guide](../guides/go.md#message-tokens) shows how to turn negotiation on.

## Related

- [Creds](creds.md): the token-and-seed file a client holds to sign the requests
  and messages it emits.
