---
title: Credential renewal
weight: 7
---

The [security model](/docs/security/) recommends short validity windows for user
credentials, and for good reason: the [allowlist](/docs/concepts/allowlist/) keys
accounts, not users, so there is no per-user revocation short of removing a whole
account. A user credential's own `exp` is often the only lever that cuts off that
one user, which argues for a short one. But a short TTL is a standing obligation.
A credential that expires quickly has to be re-issued and redelivered quickly,
over and over, for as long as the user exists. valiss ships no renewal pipeline.
That loop is yours to build, and this page is what it has to do.

## Who re-issues

Renewal is re-issuance, and re-issuance is a signing operation with the seed one
level up. A user token is signed by the account seed (`IssueUser`); an account
token is signed by the operator seed (`IssueAccount`). So the process that renews
user credentials holds the account seed, and the process that renews account
credentials holds the operator seed.

That places renewal firmly on the issuer side, off the serving hosts. The
operator seed in particular never belongs on a production box
([Seed custody](../seed-custody/)), so account-token renewal runs in the issuer's
controlled environment, not on the fleet it feeds. Today the renewing process is
your own code against the issuing API, or `examples/minter` driven from your
automation: it re-issues deterministically from a manifest, and because its
validity boundaries are absolute timestamps, re-issuing against an unchanged
manifest reproduces the same window, so advancing the window is an explicit
manifest edit. The valiss CLI (early development) is the issuer-side tool being
built to own this ceremony; until it lands, library code or `examples/minter` is
the path.

## Renewal cadence

A token is honored until its `exp` plus the verifier's skew slack (`DefaultSkew`,
two minutes): the effective expiry is `exp + skew`, not `exp`. That slack is
clock tolerance, though, not a renewal buffer, and you must not schedule renewal
to land inside it. Treat `exp` as the hard deadline and renew well before it,
leaving a lead time that covers the whole loop: signing the new credential,
writing it to wherever the service reads it, and the service picking it up. Renew
at `exp` minus that lead, and size the lead from your real redistribution latency
plus margin, never from the two-minute skew.

## Redistributing renewed credentials

Re-issuing is half the job. A fresh credential that never reaches the service is
no better than an expired one, so the new
[credentials file](/docs/concepts/creds/) has to arrive before the old one
lapses. That is the same delivery problem as
[allowlist distribution](../allowlist-distribution/), turned around to the client
side: update the secret in your secrets manager and refresh the mount, or push
the new file and hot-reload it, whatever your services read. The deadline is not
"eventually"; it is `exp` minus your lead, per credential.

## Staggering to avoid expiry herds

Credentials issued together with the same TTL expire together. Issue a batch of
users in one run with a fixed lifetime and they all lapse in the same minute,
which produces two coincident problems: a synchronized wave of `expired`
rejections (the transports map these to 401) as every affected client fails at
once, and a renewal stampede as your pipeline tries to re-issue and redeliver the
whole batch simultaneously.

Spread the expiries. Jitter the `exp` across a batch so lapses, and therefore
renewals, distribute over a window instead of piling onto one instant. The same
applies to account tokens under an operator: staggering their windows keeps a
single re-issue run from having to touch every tenant at the same moment.

## Watching the expiry horizon

Renewal is driven from the horizon, not from failures. By the time `expired`
rejections appear, the outage for that holder has already started. Track the
minimum time-to-expiry across the credentials you issue and alert while more than
a lead time is still left, so renewal fires ahead of the lapse. The
expiry-horizon signal is covered in [Monitoring](../monitoring/); renewal is
the ceremony that signal should trigger.

## Bearer tokens: TTL is the only revocation

Bearer user tokens sharpen every point above. A bearer token is accepted with no
per-request signature, so proof of possession cannot help contain it, and the
allowlist keys accounts, so it cannot cut off one bearer user either. That leaves
exactly one lever that retires a leaked bearer token short of removing its whole
account: its own expiry. For a bearer token the TTL is not merely an availability
knob, it is the sole revocation mechanism and the length of your exposure window
if the token is captured. Size it as tight as the client can tolerate
re-issuing, and treat the renewal frequency it forces as a security parameter you
chose on purpose.
