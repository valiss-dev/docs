---
title: Operations
weight: 4
description: "Running valiss in production: the pre-production hardening checklist, what to monitor and why, and the operator runbooks for rotation, the operator token lifecycle, emergency revocation, allowlist distribution, credential renewal, and seed custody."
---

The [Concepts](/docs/concepts/) explain the model and [Security](/docs/security/)
is honest about its limits. This section is about running it. valiss verifies
offline against one pinned key, so the library keeps almost no state and calls
no one at verification time. That is the point, and it is also what moves work
onto you: the allowlist you reload, the clock you trust, the epochs you advance,
and the seeds you keep are all outside the library, and getting them wrong fails
in ways a running verifier will not announce.

These pages say what that work is and how to do it safely:

- **[Hardening](hardening/)** is the pre-production checklist: the settings and
  assertions to have in place before a verifier takes real traffic, each one
  grounded in what the library does and does not do for you.
- **[Monitoring](monitoring/)** is what to watch and why. The library emits no
  metrics or logs, so the signals are yours to build. Each one is framed as a
  benign cause and the attack it could also indicate.
- **[Rotation ceremony](rotation-ceremony/)** is the step-by-step for advancing
  an epoch: re-issuing the operator token, re-minting beneath it, and using the
  keyring grace overlap so no request is refused mid-ceremony.
- **[Operator token lifecycle](operator-token/)** is the runbook for the token's
  fields, above all its validity window: what the operator token asserts, and how
  to keep its expiry from taking the whole domain down at once.
- **[Emergency revocation](emergency-revocation/)** is the decision runbook for a
  leak: find the scenario by blast radius and reach for the right lever, from the
  allowlist through epoch rotation to re-pinning the trust anchor.
- **[Allowlist distribution](allowlist-distribution/)** is getting the
  accepted-id set to every verifier: stamping a version, pushing it out, and
  confirming the fleet converged.
- **[Credential renewal](credential-renewal/)** is keeping account and user
  credentials fresh before their expiry closes them, without a re-issue stampede.
- **[Seed custody and recovery](seed-custody/)** is where the private keys live,
  how they are backed up, and what re-pinning the trust anchor costs when the
  operator seed is lost or exposed.
