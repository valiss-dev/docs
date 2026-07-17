---
title: Allowlist distribution
weight: 6
---

The [allowlist](/docs/concepts/allowlist/) revokes a tenant by removing its
account id from the accepted set, and it does so offline: no issuer is called, so
a revocation takes effect on a given verifier only once that verifier's own copy
of the list is updated. Reach is local to each copy
([Security](/docs/security/)). A fleet of verifiers therefore holds as many
copies as it has instances, and the revocation you issued is not real until the
last of them has it.

**A tenant is revoked only when the slowest verifier in the fleet has applied the
change.** Everything on this page follows from that one sentence. Getting a list
to a fleet is not a fire-and-forget push; it is a convergence you have to drive
and confirm, because until the fleet converges the revoked tenant is still being
served somewhere.

## The source of truth

Today the accepted set is a file you maintain: the newline-delimited list of
account ids that `LoadAllowlistFile` reads, one id per line, with blank lines and
`#` comments ignored. Where that file comes from is your issuing process. If you
issue with `examples/minter`, each `creds` invocation prints the account `jti`
your servers must accept to its metadata output, and collecting those ids into
the file is your step. The valiss CLI (early development) will keep the allowlist
as a first-class object beside the tokens it issues and export the exact file a
server loads; until it lands, the file is yours to assemble and version.

Whatever assembles it, treat that artifact as the authority. A verifier's
in-memory set is a cache of it, and a revocation is an edit to the artifact
followed by distribution, never an edit made on one box.

## Pushing the list to the fleet

The library has no distribution mechanism, and there is no first-party agent that
ships the file to your servers. That plumbing is yours to build, and any of the
ordinary shapes works:

- **Configuration management.** Render the artifact as a managed file and let the
  same config system that ships your other configuration converge it onto every
  host, then signal the process to reload.
- **Object store and poll.** Publish the artifact to a bucket or key-value store
  and have each instance poll for a new version on an interval, downloading and
  applying it when the version changes.
- **Sidecar.** Run a companion process that owns fetching and writing the file,
  and have the serving process watch that file for changes.

The mechanism does not matter to the library. What matters is that it is
observable (below) and that each instance applies the list safely (next).

## Applying a new list on each instance

`LoadAllowlistFile` is a one-shot: it opens a path, reads it once, and returns a
fresh `*StaticAllowlist`. It is the right call at startup and the wrong one for
reload, because the verifier holds the instance you passed to `NewVerifier` and
will not notice a new one. Live update goes the other way: retain that
`*StaticAllowlist` and drive it with `Set`, which atomically replaces the
accepted set under a lock, so every request before the swap sees the old set and
every request after sees the new one, with no half-applied state in between.

Between reading the file and calling `Set` is the one place you must not be
naive. Neither `LoadAllowlistFile` nor `Set` validates anything: an empty file
parses to an empty set, and because the allowlist fails closed, an empty set
rejects every tenant on that instance. A truncated download, a botched render, or
a cleared file is therefore not a no-op; applied verbatim it is a total denial.
So validate before you swap:

- **Refuse an empty set.** If the freshly read list is empty, do not call `Set`.
  Keep the last-known-good set and raise an alarm.
- **Refuse a cliff drop.** A list that shrank drastically from the last good one
  is far more likely a distribution fault than a mass revocation you meant.
  Compare against the previous count and refuse a drop past a threshold you pick,
  rather than swapping and cutting off most of your tenants at once.
- **Keep last-known-good.** On any failed or rejected load, hold the set you were
  already serving. A stale but valid list denies no one wrongly; an empty one
  denies everyone.

The file format is trivial to read for that check (ids one per line, `#` and
blank lines skipped), so the validation gate is a few lines of your own around
the `Set` call. Asserting a real, non-empty `StaticAllowlist` at startup belongs
on the [hardening checklist](../hardening/); this is the same gate applied to
every reload, not just the first load.

## Confirming the fleet converged

An applied list is silent: a verifier enforcing a stale set returns identities
and logs nothing, so the only way to know the fleet converged is to measure it.
Version-stamp the artifact and expose, per instance, the version currently
loaded. A version rides in the file for free as a comment, since
`LoadAllowlistFile` ignores `#` lines, so a `# version: 42` header travels with
the ids without affecting parsing.

With each instance reporting its loaded version, convergence lag becomes visible:
an instance still on an old version after you pushed a new one is a revocation
that has not taken effect there yet, and it is exactly the slowest verifier the
invariant is about. That signal, and the age of each instance's last successful
reload, are treated as monitoring signals in [Monitoring](../monitoring/); the
distribution side's job is to emit the version so those signals have something to
read.

## Rollback

Because the artifact is the authority, rollback is re-publishing the
last-known-good version through the same push path, not editing instances by
hand. The per-instance validation gate keeps a bad artifact from reaching the
accepted set (it refuses the swap and keeps serving), but the gate only holds the
line locally. The durable fix is to make the distributed artifact good again, so
a fresh instance, which has no last-known-good to fall back to, also loads a
valid list.

## The blast radius

> [!CAUTION]
> An empty or truncated allowlist applied fleet-wide fails closed on every
> request, every tenant, everywhere the bad list landed, with no error the
> library raises to explain it. Validate before every swap: refuse an empty set,
> refuse a cliff drop, and keep the last-known-good list.

A bad allowlist is not one failure mode among many; it is the outage. A revoked
tenant served a little too long is a bounded miss, but an empty or truncated list
applied fleet-wide fails every request on every instance that took it, with no
error the library raises to explain why. There is no partial version of this
failure: failing closed means all tenants, at once, everywhere the bad list landed.

That is why the validation gate is not optional polish. It is the safety that
turns the worst plausible distribution fault, a file that arrived empty or
gutted, from a domain-wide denial into a refused swap and a page. Distribution
speed matters, but a fast pipeline that ships a bad list quickly is worse than a
slow one that ships a good list. Validate first, then converge.
