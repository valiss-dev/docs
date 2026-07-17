---
title: Operator token lifecycle
weight: 4
---

The operator token is the trust domain's self-signed policy statement: its
epoch, its validity window, its extensions, signed by the pinned operator key
over its own public key. A verifier configured with it (`WithOperatorToken`, or
a keyring entry) enforces that policy on every request. One field on it deserves
its own runbook, because getting it wrong takes the whole domain down at once:
its `exp`.

## The exp bounds the whole domain

When the operator token expires, nothing beneath it verifies. The verifier
checks the operator token's window before it checks anything else, and a lapsed
one is rejected with `operator token expired: the trust domain is closed`. That
rejection is not per tenant or per server. It is simultaneous and total: every
account, every user, on every verifier that pins this domain, the moment the
token ages past its `exp`. The only slack is the skew window (`DefaultSkew`, two
minutes), the same grace applied to every expiry check. Two minutes after `exp`,
the domain is dark.

This is deliberate. Bounding the domain on the operator token's `exp` forces a
periodic renewal rather than letting a domain drift indefinitely on one key
state. The cost of that property is that the `exp` is a domain-wide deadline you
must never miss.

## Sizing the TTL against on-call reality

The `exp` is a deadline, so size it against how fast you can actually meet a
deadline, not against a calendar. Before `exp` you must issue a fresh operator
token, distribute it to every verifier, and confirm adoption. The TTL has to be
comfortably longer than that whole cycle, including the human parts: paging the
on-call, the on-call responding, and retrieving the operator seed if it is not
already to hand (see [The seed dependency](#the-seed-dependency)).

- **Too short** and you are running renewal ceremonies under time pressure, and
  a single missed page closes the domain.
- **Too long** and you weaken the periodic-renewal property that is the reason
  the deadline exists.

Pick a TTL where renewal duration plus worst-case on-call response plus seed
retrieval fits inside it several times over. That headroom is what the alert
(below) spends.

## Renewing before expiry

Renewal is issuing a fresh operator token at the *same* epoch with a new `exp`.
Same epoch is what makes it a renewal and not a rotation: it does not touch
account or user credentials, only the domain's clock. (Advancing the epoch is
the [rotation ceremony](../operations/rotation-ceremony.md); do not conflate
them.)

1. **Re-issue the operator token.** `IssueOperator(operator, WithName(domain),
   WithEpoch(current), WithExpiry(next))` with the same epoch and a fresh `exp`.
2. **Distribute it to the verifiers.** On a single-anchor verifier, rebuild it
   with `WithOperatorToken(newToken)`. On a keyring verifier, rebuild the
   keyring with the renewed token in place of the old one: the keyring keys
   entries by `(operator key, epoch)`, so the renewed token occupies the same
   slot as the one it replaces, and `NewKeyring` rejects registering a different
   token at an already-occupied `(key, epoch)` pair. Renewal is a rebuild and
   swap of the keyring, not an overlay.
3. **Verify adoption.** Confirm the `operator token expired` signal stays at
   zero and the expiry horizon (below) resets to the new `exp` across the fleet.
   Renewal is complete when every verifier reports the fresh window, not when
   the token was handed out.

## The alert

Alert on the operator token's expiry horizon: how much time remains before
`exp`. The lead time must be at least the renewal duration plus your response
time, so that when it fires you still have room to issue, distribute, and
confirm before the domain closes. Because renewal needs the operator seed, fold
seed retrieval into that lead time when the seed is not immediately available.
Wire this to the domain's `exp` and to the `operator token expired` rejection
count; see [Monitoring](../operations/monitoring.md) for both signals.

## When it has already lapsed

If the alert was missed and the token expired, the domain is already down: every
request is failing `operator token expired: the trust domain is closed`, and it
has been since two minutes past `exp`. Recovery is renewal under fire.

1. **Re-issue from the operator seed.** `IssueOperator` at the same epoch with a
   fresh `exp`. This requires the operator seed; if it is in cold storage,
   retrieving it is the long pole and the domain stays down until it is in hand.
2. **Push to every verifier**, single-anchor or keyring, as in the renewal
   procedure above.
3. **Watch the recovery signature.** As verifiers adopt the fresh token, the
   authentication error rate collapses back to baseline: the `operator token
   expired` rejections stop and normal traffic resumes. A fleet that recovers
   unevenly tells you which verifiers have not yet taken the new token.

Because the only grace is the two-minute skew, there is no gentle warning at
runtime: the alert firing ahead of `exp` is the warning, and it is the one you
must not miss.

## The seed dependency

Every path above (renewal and emergency recovery alike) requires the operator
seed, because `IssueOperator` signs with the operator key pair. The seed is the
ultimate secret in the system and belongs in custody, not on a server (see the
[Security model](/docs/security/) and [Custody](/docs/concepts/custody/)). The
operational consequence is direct: if the seed lives in cold storage, the alert
lead time must include the time to retrieve it. An alert horizon that assumes
the seed is at your fingertips is too short by exactly the retrieval time, and
that gap is discovered at the worst possible moment.

## What is manual today

Issuing an operator token is programmatic only. The `minter` example mints
account and user credentials; it does not produce operator tokens. Renewal and
emergency recovery are hand-written `IssueOperator` calls against the library,
with hand-maintained keyring rebuilds and distribution. The planned valiss CLI,
holding the operator identity in its per-operator store, is what will own
operator-token issuance and renewal as first-class verbs; until it ships, the
procedures here are manual.
