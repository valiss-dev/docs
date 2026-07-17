---
title: Documentation
description: "Documentation for valiss, offline tenant authentication for services: quickstart, concepts, language guides, reference, and the wire specification."
---

valiss is offline tenant authentication for services, built on a
three-level chain of Ed25519 keys:
[operator, account, user](/docs/concepts/entities/). Verification is
offline against a single pinned public key, so there is no auth service to run,
no token introspection endpoint to call, and issuing credentials never touches
production.

Most multi-tenant services grow a central auth dependency: an OAuth provider, a
session store, a per-tenant key registry, something every request consults and
every deployment keeps alive. valiss inverts that. Trust is one operator public
key baked into the server, and everything a request carries is a self-contained
signed credential that verifies against that key with no network call.

The model is small. An operator key signs account (tenant)
[tokens](/docs/concepts/tokens/), an account
key signs user tokens, so every credential is signed by the key one level up.
Each request is signed by the subject's own key; the server walks the chain
back to the pinned operator key, checks the account token against a revocation
[allowlist](/docs/concepts/allowlist/) that fails closed, and hands the tenant
and user identity to the handler.
[Proof of possession](/docs/security/) is the default, so a captured token is
inert
without the seed that signs requests. Bearer user tokens are the deliberate
exception, for clients that can hold a short-lived token but not a key.

valiss fits multi-tenant APIs, where each customer is an account that mints its
own scoped-down user and service credentials without asking you; machine-to-machine
auth, where services carry keys and per-request signatures rather than
shared secrets; and edge or on-prem deployments, where a verifier needs only
the operator public key, so isolated installs authenticate exactly like
connected ones. It is a poorer fit elsewhere. If you want one central switch to
invalidate every credential at once, that is
[epoch rotation](/docs/concepts/rotation/), not the per-request path. If a
client cannot hold or protect any credential at all,
[custody](/docs/concepts/custody/) has a real gap today and there is no
custodian server yet. And if a
conventional session or OAuth stack already serves a single-tenant app, valiss
buys you little.

The sections below map the rest: the [concepts](/docs/concepts/) one page at a
time, the language [guides](/docs/guides/) for wiring valiss into real code,
and the [reference](/docs/reference/) material for the exact rules and
guarantees.
