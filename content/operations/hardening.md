---
title: Production hardening checklist
weight: 1
---

Verification is offline, so a misconfigured verifier does not fail loudly at
startup: it runs, accepts traffic, and enforces exactly what you set, no more.
This is the list of things to have in place, and to assert, before a verifier
takes production traffic. Each item names what the library does for you and what
it leaves to you, because the gap between the two is where deployments get hurt.

Work through it in order. The early items are trust configuration you cannot
retrofit safely; the later ones are surfaces that are off until you enable them.

## 1. Pin and verify the trust anchor

`NewVerifier(operatorPub, allowlist)` is the whole trust configuration: one
pinned operator public key, no network call. Whoever substitutes that key owns
the trust domain, because every credential verifies against it and nothing else.
Confirm the pinned value through a second channel before it ships (a published
fingerprint, a signed release, a second operator), not just by copying whatever
a build handed you.

The corresponding operator seed is the ultimate secret in the system, and it
never belongs on a production host. Issuing credentials happens off-box, with
the seed held by the issuer; a verifier holds only the public key. If your
provisioning puts the operator seed anywhere near a serving process, stop and
fix that first.

## 2. Load a real allowlist, and assert it

The allowlist fails closed: an account token is accepted only if its id is in
the set. The development-only `AllowAll` accepts every id and turns revocation
off entirely, and it does so silently. The library emits no warning that you are
running wide open; `AllowAll.Allowed` simply returns true for everything.

So the assertion is yours. At startup, refuse to serve unless the allowlist is a
real `StaticAllowlist` (never `AllowAll`) and is non-empty. An empty allowlist is
not a safe default that denies everything useful by accident: it is a sign the
load failed, and it should abort startup rather than run.

## 3. Wire allowlist reload yourself

There is no file watcher in the library. `LoadAllowlistFile` reads a
newline-delimited file once, and `StaticAllowlist.Set` atomically replaces the
accepted set under a lock. Reload is the two composed by you: retain the
`*StaticAllowlist` you passed to `NewVerifier`, and on your own trigger (a signal,
a timer, an inotify hook) load the new file and call `Set`.

Two rules make that safe. Validate the freshly loaded set before you swap it:
`Set` will install whatever you hand it, including an empty slice, which would
silently revoke every account. Reject an empty or failed load and keep the
previous set. And monitor staleness: a reload that quietly stops (a broken
distribution, a file that never changes) leaves the verifier enforcing an old
list with no error, so track the age of the last successful load as a signal in
its own right. See [Monitoring](../monitoring/).

## 4. Treat the clock as part of the trusted computing base

Freshness has no external authority; it anchors entirely on the verifier's local
clock. The request timestamp is accepted only within a symmetric skew window
around now (`DefaultSkew`, two minutes); token `exp` and `nbf`, the operator
token's own validity window, and the replay cache's retention are all evaluated
against that same clock. A verifier whose clock is wrong will accept stale
requests or reject fresh ones, and neither is visible from inside the library.

Run authenticated NTP on every verifier, and alert on drift. And understand what
`WithSkew` costs: widening it does not only loosen timestamp acceptance. It
widens the window in which a captured request stays replayable, and it lengthens
the grace past a token's stated expiry (expiry is checked with the same skew
slack), so a two-minute widening is a two-minute-longer life for every revoked-
by-expiry credential. Keep it as tight as your real clock spread allows.

## 5. Watch the operator token expiry: it closes the whole domain

When you enforce operator policy (`WithOperatorToken`, or a keyring, where every
entry carries policy), the operator token's own validity window gates every
request. Once it expires, `VerifyRequest` fails every credential with "the trust
domain is closed": not a subset, everything. This is the one setting whose lapse
is a full outage rather than a partial denial.

Monitor its `exp` with a lead time longer than a rotation takes, and re-issue
before it lands. The keyring is what makes the changeover gap-free: it holds
several epochs for one operator key at once, so you register the new-epoch token
alongside the old one and let producers re-issue at their own pace, dropping the
old entry only after the grace overlap. Never let the only trusted operator
token expire with no successor already trusted.

## 6. Decide replay explicitly; the built-in cache has sharp edges

Replay suppression is off by default. Within the skew window the identical
signed request replays verbatim, signature valid and timestamp fresh, unless you
enable it. Turn it on with `WithReplayCache` plus a per-request nonce folded into
the signed context; the verifier then rejects a nonce it has already seen and
requires a nonce on every signed request.

The built-in `MemoryReplayCache` has limits you must design around:

- **It is process-local.** Each instance sees only its own nonces, so a replay
  landing on a different instance is not caught. Exactly-once across a fleet
  requires backing `WithReplayCache` with shared storage.
- **Its footprint is bounded by retention, not by a cap.** Each nonce is held
  for twice the skew window, and there is no size limit; memory grows with
  request rate times twice the skew. Every insert also scans the whole map to
  prune expired entries, under a single mutex, so cost per request is O(n) in
  the live set. On a busy instance this is a real hot spot; size skew and rate
  accordingly, or use a backend built for it.
- **The interface has no error channel.** `ReplayCache.SeenBefore` returns a
  bare `bool`. A shared backend that times out or errors cannot report that
  through the interface, so your implementation must decide, explicitly, whether
  a backend failure fails open (serve, risking a missed replay) or fails closed
  (reject, risking an outage). Make that choice explicitly; there is no
  default to inherit.
- **It never covers bearer tokens.** The replay check lives on the signed path.
  A bearer request carries no signature and no nonce, so the cache does nothing
  for it. Bearer replay is bounded only by the token's TTL and the allowlist.

## 7. Constrain bearer tokens

A bearer user token is accepted with no per-request signature: possession is the
proof. That is a deliberate weakening for clients that cannot hold a key, and it
comes with obligations.

- **TLS is mandatory, not advisory.** A bearer token is a bearer secret; anyone
  who captures it in transit can use it. valiss does not encrypt.
- **Use the shortest workable TTL.** A captured bearer token is usable until it
  expires or its account leaves the allowlist, so its lifetime is your exposure
  window.
- **Revocation is account-level only.** The allowlist keys accounts, not users;
  there is no per-user allowlist entry. You cannot revoke one bearer user without
  removing its whole account. Plan for that: keep the TTL short enough that
  lapse, not revocation, is the normal cutoff.
- **Client-side exfiltration is outside the model.** Bearer tokens are for
  browsers and similar clients; theft from the client (XSS, a compromised page)
  is not something the verifier can defend against. Scope and expire them on that
  assumption.

## 8. Keep seeds off the serving path and out of everything that captures text

The security of the whole model rests on seeds staying secret, and seed custody
is outside the library. A credentials file carries the subject's token together
with the seed that signs its requests, in one file.

- Store credentials files with owner-only permissions (`0600`) and never commit
  one to version control.
- Prefer a secrets manager to environment variables. Env vars leak into process
  listings, crash dumps, and child processes more readily than a file read on
  demand.
- Treat anything that captures text as a seed sink. A log line, a stack dump, or
  an error report that echoes a credentials file or an environment block is seed
  exposure, not a cosmetic bug.

Issuing and holding seeds is the issuer side, not the verifier. The valiss CLI
(early development) is the tool being built to own that custody in an encrypted
per-operator store; until it lands, the working path is the library's issuing
functions run in your own controlled tooling, off the serving host, with the
seed material handled as above. See [Custody](/docs/concepts/custody/).

## 9. If you set `AllowMissingExtension`, prove your external gate runs

The HTTP and gRPC integrations authorize through signed extensions and fail
closed: without `AllowMissingExtension`, every token in the chain must carry the
transport extension or the request is denied. Passing `AllowMissingExtension`
accepts tokens that carry no extension and imposes no constraint on them, which
means it moves the entire authorization decision to whatever gate you run
outside the transport.

That is legitimate, but only if the gate actually executes on every path. Test
it: send a request that the extension would have denied and confirm your external
check denies it. A deployment that sets `AllowMissingExtension` and then forgets
to wire (or accidentally bypasses) the external gate is authenticating requests
and authorizing nothing.

## 10. Handle message-token replay and binding on the receiver

Message tokens (`IssueMessage` / `VerifyMessage`) are proofs of origin, and their
verification carries no replay cache at all. There is no same-destination replay
suppression within a token's TTL: the same valid message token can be presented
repeatedly until it expires.

- **Deduplicate non-idempotent messages yourself.** A verified `MessageClaims`
  carries the token's `jti` (its content-hash id). A receiver acting on
  non-idempotent messages must record and reject ids it has already processed;
  the library will not.
- **Audience and checksum are opt-in receiver checks.** A token is bound to a
  destination only if the receiver calls `ExpectAudience`, and to its payload
  bytes only if the receiver calls `WithPayload` (or requires one with
  `RequireChecksum`). A receiver that knows its own identity should set the
  audience check; one that requires payload binding should set the checksum
  check. Neither is enforced unless you ask for it.
