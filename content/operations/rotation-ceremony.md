---
title: Rotation ceremony
weight: 3
---

[Rotation](/docs/concepts/rotation/) explains why epochs exist: one counter
advance retires an entire generation of credentials beneath the trust anchor.
This page is the ceremony that advances it without refusing a single request in
flight. The safety comes entirely from ordering, so the order is the whole
point.

Two facts about the library set the constraints. First, the request-path epoch
check exists only when a verifier is configured with an operator token
(`WithOperatorToken`) or a keyring (`NewKeyringVerifier`). A plain
`NewVerifier(operatorPub, allowlist)` with no operator policy does not enforce
epochs at all, and advancing the counter changes nothing there. This ceremony
presupposes every verifier runs with an operator token or a keyring; confirm
that before you start. (The [Security model](/docs/security/) states this
independently; it is repeated here because a rotation against verifiers that do
not enforce epochs is a no-op that looks like success.)

Second, the epoch is enforced twice on every request: a keyring verifier selects
its trust entry by the credential's `(operator key, epoch)` pair, so an
unregistered epoch fails at selection before any equality check; and the
verifier then requires `account.epoch == operator.epoch` (and the same for a
user token). Both the incoming and the outgoing generation must therefore be
registered on the verifier at once for both to pass. That is what the keyring
grace overlap is for, and it is why registering the new epoch is the step that
must come first.

## The safe order

Rotating from epoch N to epoch N+1:

1. **Issue the epoch N+1 operator token.** `IssueOperator(operator,
   WithName(domain), WithEpoch(N+1), WithExpiry(...))`. This is the policy
   statement the new generation verifies under, and the verifiers cannot trust
   the new epoch until this token exists to register.
2. **Register both epochs on every verifier, before anything else changes.**
   Build the keyring with the outgoing and incoming operator tokens side by
   side, `NewKeyring(epochNToken, epochN1Token)`, and rebuild each verifier's
   `NewKeyringVerifier(keyring, allowlist)` around it. This step is
   non-destructive: adding epoch N+1 does not stop epoch N from verifying, so no
   request breaks. After it lands fleet-wide, a credential at either epoch
   passes, and the order in which producers switch over no longer matters.
3. **Re-issue account and user credentials at N+1.** Re-issue each account token
   with `IssueAccount(operator, tenantPub, WithEpoch(N+1), ...)` and each user
   token with `IssueUser(account, userPub, WithEpoch(N+1), ...)`. A re-issued
   account token has a new `jti`, so add the new ids to the allowlist (removing
   the old ones is part of retiring epoch N, step 6, not this step).
4. **Distribute the new credentials** to every tenant and every producer, as
   [credentials files](/docs/concepts/creds/). Producers adopt them at their own
   pace; the keyring grace is what makes that pace safe.
5. **Confirm no epoch-N traffic remains.** Watch the epoch a request verified
   under and the `epoch_mismatch` rejection signal (see
   [Monitoring](../operations/monitoring.md)). Rotation is complete only when
   verified epoch-N traffic reaches zero across the whole fleet, not when the
   last credential was handed out.
6. **Retire epoch N.** Let the outgoing epoch-N operator token's `exp` lapse,
   which closes its keyring entry cryptographically, or rebuild the keyring with
   only the N+1 token. Remove the retired account `jti`s from the allowlist.
   From here only epoch N+1 verifies.

The operator token issued in step 1 is programmatic today: it is a library call
(`IssueOperator` with `WithEpoch`), not something the `minter` example produces.
See [What is manual today](#what-is-manual-today).

## The inverted-order failure

> [!WARNING]
> Never flip verifiers to epoch N+1 before the producers have re-issued. The
> moment a verifier trusts only N+1, every still-current epoch-N credential
> fails at once: not one tenant, every tenant on that verifier, a fleet of 401s
> the instant the policy changes. Registering both epochs first (step 2) is what
> removes the hazard.

The failure mode this order exists to prevent is flipping the verifiers to
epoch N+1 alone before the producers have re-issued. The moment a verifier
trusts only N+1, every still-current epoch-N credential fails: on a keyring
verifier the `(operator, N)` lookup finds no entry and the request is rejected
with `no trusted operator <key> at epoch N`; under a single operator token the
equality check fails with `account token epoch N, trust domain epoch N+1`. Both
surface as `epoch_mismatch` (or `unknown_operator`) and both are domain-wide: not
one tenant, every tenant on that verifier, all at once, a fleet of 401s the
instant the policy changes.

The mirror image is just as bad. If producers re-issue at N+1 while the
verifiers still trust only N, the new credentials fail the same way. Neither
side may move alone. Registering both epochs first (step 2) is what removes the
ordering hazard: while both are registered, a producer that has switched and one
that has not both verify, and you are free to migrate producers in any order and
at any pace.

## Aborting mid-ceremony

Epochs are monotonic. The counter only advances; there is no rollback, and
"un-bumping" to N is not an operation the model has. If you must stop partway,
the recovery is not to reverse the ceremony but to hold it open.

- **Keep the keyring.** Do not remove epoch N while any producer might still
  present an epoch-N credential. Removing it is exactly the inverted-order
  failure, self-inflicted.
- **Recovery is completing or extending grace.** Either finish the ceremony
  (re-issue and distribute the stragglers, then retire N at step 6), or extend
  the grace window by re-issuing the outgoing epoch-N operator token with a
  longer `exp` so its keyring entry keeps verifying while you sort out why the
  ceremony stalled.

There is no state to unwind because the keyring grace never removed anything;
the whole design of the overlap is that abandoning it costs nothing but the
extra window.

## How long grace should last

The grace window must stay open until credential redistribution provably
completes: until monitoring shows zero verified epoch-N traffic across the
entire fleet (step 5), not until you believe distribution finished. Size it to
the slowest producer's real update cadence, with margin, and bound it
cryptographically by giving the outgoing epoch-N operator token a short `exp`.
When that `exp` lapses the epoch-N keyring entry closes on its own, so an
over-long grace is self-correcting and an over-short one shows up as a spike of
`epoch_mismatch` on stragglers, which is the signal to re-issue the outgoing
token with more time (see [Aborting mid-ceremony](#aborting-mid-ceremony)).

## What is manual today

The library exposes every primitive the ceremony needs: `IssueOperator`,
`IssueAccount`, `IssueUser` with `WithEpoch`, and `NewKeyring` /
`NewKeyringVerifier`. Driving them in the right order is manual. Concretely:

- **Issuing the operator token is programmatic only.** The `minter` example
  issues plain identity account and user tokens and does not stamp epochs or
  produce operator tokens at all. Step 1, and the epoch stamping in step 3, are
  hand-written Go against the library today, and the manifests and credentials
  files are hand-maintained.
- **Sequencing, distribution, and convergence are manual.** Nothing in the
  library registers a keyring fleet-wide, pushes credentials, or confirms that
  epoch-N traffic has drained. Those are operational steps you run and verify by
  monitoring.

The planned valiss CLI is what will own this. It holds the operator identity in
its per-operator store (see [Custody](/docs/concepts/custody/)) and rolls the
domain forward with a single `operator rotate` verb, issuing the new operator
token and re-issuing beneath it rather than each token by hand. Until
that ships, the runbook above is the ceremony.
