---
title: Monitoring
weight: 2
---

The library emits no metrics and writes no logs. A verifier is a function that
returns an identity or an error, and everything observable about it is what you
record around that call. This page is the list of signals worth recording and,
for each, the benign explanation and the attack that wears the same shape.
Because verification is offline, these signals are often the only place an
attack becomes visible at all: there is no issuer logging failed lookups on your
behalf.

## Instrumentation is yours; the reason codes are the spec

`VerifyRequest` returns a Go error, and the transport integrations map it to a
status (the HTTP middleware to 401 for authentication failures and 403 for
extension denials). Neither step counts anything. Your instrumentation goes in
that error-mapping layer: as you translate a rejection into a response, record
what it was.

Record it by reason, not just as a count. The wire specification defines a
stable, language-neutral taxonomy of failure codes ([SPEC-1 section 7](https://valiss.dev/valiss)): `not_allowlisted`, `epoch_mismatch`, `skew`,
`bad_request_signature`, and the rest. The short codes are fixed across
implementations even though the human messages are illustrative, so they are the
right label to alert on. The Go
library returns error values rather than the codes directly, so mapping cause to
code is part of the work you do in the error-mapping layer; do it once, there,
and every signal below becomes a query over reason codes.

A rejection rate is meaningless without a baseline. Establish the normal per-
reason rate for your traffic first; the guidance below is about deviation from
that baseline, not absolute numbers.

## Authentication failures, by reason code

Watch the failure rate broken out per reason. A flat total can hide the signal
that matters: which reason moved. Each of these has a routine cause and an
adversarial one, and the reason code is what tells them apart.

- **`not_allowlisted` spikes.** Benign: a stale allowlist on one or more
  instances, an account legitimately removed while a client keeps trying, or a
  distribution that has not converged (see fleet convergence below). Adversarial:
  someone probing with account ids that were never issued, or replaying a token
  whose account you have already revoked. Correlate with the allowlist age per
  instance: if the list is current everywhere and `not_allowlisted` is climbing,
  it is probing, not staleness.
- **`epoch_mismatch` with no rotation in progress.** During a rotation ceremony
  a burst is expected as producers catch up. Outside one it is a real signal:
  credentials minted at an epoch your domain has moved past, which means either
  replayed old credentials or a ceremony that half-completed and left some
  issuer behind. Either way the fix is operational, and the alert should fire
  precisely because no rotation was supposed to be happening.
- **`skew` spikes.** Benign: clock drift on a verifier or a population of
  clients, the ordinary failure mode when NTP lapses. Adversarial: timestamp
  manipulation, an attacker probing the edges of the acceptance window. Because
  the clock is in the trusted computing base ([Hardening](../hardening/)),
  separate the two by checking the verifier's own clock health first; if the
  verifier clock is good, the drift or manipulation is client-side.
- **`bad_request_signature` spikes.** Benign: a broken or mismatched client,
  typically one whose canonical request context has drifted from what the server
  reconstructs, so its signatures no longer verify. Adversarial: tampering or a
  man-in-the-middle altering signed requests in flight. A single deploying client
  regressing looks different from a spread across many clients; the latter is the
  one to treat as an attack until proven otherwise.

## The operator-token expiry horizon

This is the one timer whose lapse is a total outage rather than a partial
denial. When operator policy is enforced (`WithOperatorToken`, or any keyring
entry), an expired operator token fails every request with "the trust domain is
closed." Track the operator token's `exp` as a countdown and alert on the
horizon, not on the failure: by the time requests are failing, the outage has
started.

Set the alert lead time to at least the time a rotation ceremony takes plus your
team's response time, so the page arrives with enough runway to re-issue and
distribute a successor before the current token lands. This is the single most
important expiry to watch, because it is the only one that fails closed for the
entire domain at once.

## Allowlist age and fleet convergence

A verifier enforcing a stale allowlist looks perfectly healthy: it returns
identities, logs nothing, and simply honors accounts you meant to revoke. The
only way to see staleness is to measure it, and the library does not measure it
for you.

Stamp a version or generation number into whatever you distribute (a comment
line in the allowlist file is enough, since `LoadAllowlistFile` ignores `#`
lines) and expose, per instance, the generation currently loaded and the age of
the last successful reload. Two signals fall out of that:

- **Absolute age.** An instance whose last successful reload is older than your
  distribution interval has a broken reload path, whatever the reason. Alert on
  the age exceeding a threshold you choose from that interval.
- **Fleet convergence.** After you push a new generation, the fleet should
  converge on it within a bounded window. An instance stuck on an old generation
  is a revocation that has not taken effect there yet. A revocation is only as
  prompt as your slowest-converging instance, so convergence lag is the true
  latency of your kill switch, and it is worth a dashboard.

## Replay cache size and reject rate

If you enabled `WithReplayCache`, two of its properties are worth a metric. Its
size is bounded by retention rather than by a cap, growing with request rate
times twice the skew window, so on a busy instance the live set is a real memory
consumer; graph it. And the replay reject rate (`replay` reason) is itself a
signal: a nonzero rate under normal operation means either a client retrying too
aggressively or an actual replay attempt, and it should be near zero when
neither is happening. If you back the cache with shared storage, watch that
backend's own health too, since the `ReplayCache` interface cannot report its
failures through the return value.

## Credential expiry horizons

Account and user credentials carry their own `exp`, and when one lapses the
holder's requests fail (`expired`) until it is renewed. Unlike the operator
token this is a scoped outage, one tenant or one user rather than the domain, but
it is still an outage for whoever is affected and still invisible until requests
start failing. Track the expiry horizons of the credentials you issue and drive
renewal from the horizon, ahead of the lapse, rather than reacting to `expired`
rejections after the fact. Credential renewal as an operator ceremony is covered
on its own page.

## The framing to keep

Every signal here has a benign reading and an adversarial one, and they share a
shape: a stale list and a prober both raise `not_allowlisted`; a drifting clock
and a manipulated timestamp both raise `skew`. Monitoring valiss is not about
catching a single malicious event. It is about knowing your baseline well enough
that the deviation is legible, and instrumenting the reason codes finely enough
that you can tell which of the two explanations you are looking at.
