---
title: Custodianship
weight: 8
---

A credential is only as safe as the party that holds it. In valiss the party
that keeps a subject's seed, and the tokens that go with it, is that subject's
**custodian**. Custodianship is the question of who that party is: the subject
itself, the issuer that minted the credential, or some component standing
between a client and the verifier.

One thing to state plainly and up front: valiss has no custodian server today.
A first-party service that would hold credentials on a client's behalf is
**planned but not implemented**. It does not exist, it has no release, and
nothing in the protocol depends on it. Everything below that describes
custodianship *as it works now* is custody you already arrange yourself out of
the pieces valiss ships. Everything that describes the *custodian server* is a
statement of a problem, not of a built component. The two are kept clearly
apart.

## Custody as it works today

Today custody is self-custody. A service that authenticates with valiss holds
its own [creds file](creds.md), its tokens packaged with the signing seed, and
signs its own requests. Nothing sits between it and the verifier. The verifier
holds only the operator public key and an [allowlist](allowlist.md), never a
seed, so custody is one-directional: signing power lives only on the signing
side, and a verifier compromise cannot leak it. The seed is the secret in the
file, and its holder can *be* the subject; the tokens beside it are public
artifacts that give nothing away.

The issuer is the one custodian valiss treats as special, and the rule there is
firm: the operator seed never touches production. It lives with the issuer in a
secrets manager and comes out only to sign new credentials (see the
[security model](../security.md) on operator-seed custody). The
[`examples/minter`](https://valiss.dev/valiss) tool shows the shape of doing
this well. It is stateless: key pairs are printed once and never stored, and
signing seeds are supplied through the environment rather than held by the
tool, so the seed's custodian is the secrets manager, not the process that
signs. That out-of-process separation, the thing that holds the seed kept
distinct from the thing that uses it, is custodianship done properly with
today's parts.

## The gap custody leaves

Self-custody assumes a client that can hold and protect a whole credential. Not
every client can.

- Constrained environments, where a creds file and a signing key cannot be
  stored safely, or cannot be stored at all.
- Third parties you want to authenticate but cannot hand a full creds file to,
  because handing over the seed is handing over the ability to be the subject.
- Contexts where transmitting a whole token on every call is impractical:
  the token is large, or the channel between the client and the service carries
  only a short opaque string.

For these, self-custody is either impossible or unsafe. Bearer tokens help only
partly: a bearer token still *is* the whole token, replayable by whoever holds
it until it expires. What is missing is a party that will hold the real
credential and let a weak client act through it with something smaller.

## The planned custodian server

That party is the planned **custodian server**, described here at the level of
the problem it solves. Its design is deliberately unsettled, and this page will
not invent one.

What can be said about the intent:

- It is planned as an **optional, first-party service**. An operator chooses
  whether to run one, and a deployment that does not is a complete valiss
  deployment. The protocol never requires it.
- It would **hold credentials and verify tokens on behalf of holders** that
  cannot carry or protect a full credential themselves.
- It would issue **short aliases** that stand in for a stored credential: an
  api-key-style id and secret pair a weak client can carry and present where
  transmitting the whole token is impractical. The alias refers to the
  credential the custodian holds; the client need never see the token or seed
  behind it.
- It is **infrastructure, not the trust root**. Offline verification against
  the pinned operator key stays the core of valiss with or without a custodian.
  Running one adds a convenience for weak clients; it does not move the trust
  anchor onto a server, and everything in the security model still roots in the
  operator public key.

What is deliberately **left open**, and must not be read into the points above:

- whether the custodian stores seeds, or only tokens;
- how an alias is consumed, whether a client exchanges it for a credential or
  the custodian introspects it on the client's behalf;
- the RPC framework, the storage backend, and any timeline.

These are open on purpose. Fixing them here would commit the design before it
is decided. When the custodian server is built, this page becomes its concept
page. Until then it marks the boundary between what valiss does today and one
direction it may grow.

## Related

- [Creds](creds.md): the credential file whose custody this page is about, and
  why the seed inside it, not the tokens, is the secret to guard.
- [Security model](../security.md): the operator-seed custody guidance, and the
  one-directional custody the verifier enforces by holding no seed at all.
- [Allowlist](allowlist.md): a single-method predicate the verifier consults on
  every request. Because it is a pluggable predicate rather than a fixed list, a
  custodian is exactly the kind of service that could back an interface like it.
