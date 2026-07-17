---
title: Emergency revocation
weight: 5
---

When something leaks, the right response depends entirely on what leaked and how
far it reaches. valiss verifies offline against a pinned key, so there is no
issuer to call and no central revoke button; the levers are the allowlist, token
expiry, epoch rotation, and, in the worst case, re-pinning the trust anchor.
This page is a decision runbook: find the scenario by blast radius, take the
steps, and accept the tradeoff each one carries. The underlying semantics are in
the [Security model](/docs/security/); here they are turned into actions.

## The stale-allowlist caveat, up front

Every allowlist-based step below shares one limit, so state it once. A verifier
enforces its own local copy of the allowlist. There is no central push. An id
you remove is still accepted on any server that has not yet reloaded its list.
`LoadAllowlistFile` reads the file once and installs no watcher, so each server
reloads only when you make it (a file watch, SIGHUP, or a poll interval calling
`Set`). Revocation lands only when the
slowest server reloads, so "revoked" means revoked on the servers that have
converged, and the tenant is not actually cut off until the last one has. Treat
every "remove from the allowlist" instruction as "remove and confirm the fleet
reloaded"; the convergence procedure is [Allowlist distribution](../operations/allowlist-distribution.md).

## Leaked bearer token

A [bearer token](/docs/concepts/tokens/) authenticates by possession alone, so a
captured one is directly usable as that user until it expires or its account
leaves the allowlist. There is no per-request signature to fall back on.

Options, and the tradeoff:

- **Wait out the TTL.** If the bearer token was issued with a short validity
  window, the cheapest containment is to let it expire. Damage is bounded by the
  remaining window and no one else is affected.
- **Revoke the account.** Remove the account `jti` from the allowlist
  everywhere. This cuts the leaked token immediately, but it also cuts *every*
  user of that tenant, because the allowlist keys accounts, not users, and there
  is no per-user allowlist entry to remove instead. The tradeoff is stark:
  confine the blast to one tenant, but take the whole tenant down to do it.

**Preventive.** The real mitigation is not reactive. Issue bearer tokens with
short TTLs so a leaked one ages out before an account-wide revocation is ever
worth its collateral. Keep this the standing policy for anything bearer.

## Leaked user seed

A leaked user seed lets the holder sign requests as that user. The containment
lever is the same as for a bearer token, and it has the same limit: there is no
per-user allowlist entry, so the only immediate cut that reaches one user is
revoking its whole account, with the whole-tenant collateral that implies.

- **Immediate:** either let the user token's (short) TTL lapse, or revoke and
  re-issue the account if the exposure cannot wait for the TTL.
- **Clean fix at rotation:** re-issue that user at the next epoch advance. Once
  the [rotation ceremony](../operations/rotation-ceremony.md) retires the old
  epoch, the leaked user token stops verifying with everything else from that
  generation, and the user is re-issued fresh at the new epoch.

The tradeoff is the exposure window: between the leak and either the TTL lapsing
or the next rotation, the only zero-collateral option is to wait, and the only
immediate option takes the tenant down.

## Leaked account seed

A leaked account seed is a full tenant takeover: the holder can sign as the
tenant and issue user tokens beneath it, bounded only by the account's own
scope. This is a tenant-level breach and it must be treated as one.

1. **Revoke.** Remove the account `jti` from the allowlist everywhere. This cuts
   the account and every user beneath it in one edit.
2. **Re-key the tenant.** Generate a new account key pair; the old account key
   is compromised and can never be trusted again.
3. **Re-issue.** Issue a fresh account token for the new key (a new `jti`) and
   re-issue the tenant's users under it. Add the new `jti` to the allowlist and
   distribute the new [credentials files](/docs/concepts/creds/).

The tradeoff is unavoidable downtime for that one tenant: between revocation and
the re-keyed credentials converging across the fleet, the tenant cannot
authenticate. A compromised account key cannot be honored while you re-key, so
the tenant's outage is the cost of containment. It does not reach other tenants:
delegation cannot widen, and an account can never mint another account.

## Leaked or suspected operator seed

This is the worst case, and the levers behave differently, so be precise about
what is contained and what is not.

- **Request-path damage is partly contained by the allowlist.** A forged account
  token minted from the stolen seed has a fresh content-hash `jti` that is on no
  server's allowlist, and an attacker cannot add to your allowlist. So the
  stolen seed does not, by itself, admit a new tenant to an allowlist-enforcing
  server.
- **Message verification is not contained.** `VerifyMessage` roots trust in the
  operator key alone and holds no allowlist, so a stolen operator seed forges
  provenance that offline [message](/docs/concepts/messages/) receivers accept.
- **Epoch rotation does not contain it.** The seed holder re-issues at whatever
  epoch you advance to. Rotation is for the credentials beneath the anchor, not
  for the anchor itself.

The only full remedy is re-pinning a new operator public key everywhere. Sequence
it so the uncontained surface closes first:

1. **Generate a new operator key pair.** This is the new trust anchor.
2. **Re-issue the whole tree beneath it.** New account tokens (new `jti`s) and
   their user tokens, signed by the new operator key, at a chosen epoch.
3. **Re-pin the anchor on message-verification receivers first.** They are the
   uncontained surface, so they are the ones to move to the new operator public
   key before anything else.
4. **Re-pin request verifiers and update their allowlists** to the new `jti`s
   (`NewVerifier` / keyring entries around the new operator key).
5. **Distribute the new credentials** to every tenant.
6. **Retire the old anchor.** Once every verifier pins the new key, the old
   operator key and everything it signed no longer verify anywhere.

The tradeoff is that this is the most expensive operation in the system: every
credential and every verifier config changes. The caveat that governs the whole
sequence: until the last verifier re-pins, the compromised anchor is still
trusted there, which is exactly why message-verification receivers move first.

## What is manual today

Removing an id from the allowlist is a file edit plus a reload; the library
gives you `LoadAllowlistFile` and an in-memory `Set`, but distributing the edit
and confirming convergence is operational work covered by [Allowlist distribution](../operations/allowlist-distribution.md). Re-keying, re-issuing,
and re-pinning are hand-written `IssueAccount` / `IssueUser` / `IssueOperator`
calls and hand-maintained credentials files today; the `minter` example covers
the plain account and user issuance but not operator tokens or epoch stamping.
The planned valiss CLI, holding the operator identity in its per-operator store,
is what will own the re-key and re-pin flows as verbs; until it ships, these
runbooks are manual.
