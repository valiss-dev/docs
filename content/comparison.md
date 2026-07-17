---
title: Why valiss, and why not
weight: 9
description: "When to choose valiss and when not to: an honest comparison against OAuth2/OIDC, mTLS, SPIFFE/SPIRE, SSH certificates, and Biscuit/macaroons, plus how to adopt it alongside an existing identity provider and how to leave."
---

valiss is one point in a large and well-populated design space, and it is not
the right point for every problem. This page is about where it fits and where a
different tool fits better. It compares valiss against five established
approaches, one section each, and it tries to be honest in both directions: each
section says plainly where the incumbent wins before it says where valiss does.
If you are weighing valiss against something below and the honest answer is that
the incumbent already serves you, that is the answer.

The short version of the argument for valiss is the one the
[documentation hub](/docs/) opens with: most multi-tenant services grow a central
auth dependency, something every request consults and every deployment keeps
alive, and valiss inverts that into a single pinned operator public key and
self-contained signed credentials that verify with no network call. The rest of
this page is about when that inversion is worth its costs.

## OAuth2 and OIDC

OAuth2 is the delegated-authorization framework the web runs on, and OpenID
Connect is the identity layer built on top of it. Between them they own the
problem of logging a human in.

**Where they win, and it is a lot.** The ecosystem is enormous and the tooling
is mature: hosted identity providers (Auth0, Okta, Entra ID, Google, Keycloak)
take the operational burden off you entirely, the browser redirect flows
(authorization code with PKCE) are exactly what OIDC was designed for and valiss
has no equivalent, the libraries are widely audited, and nearly every engineer
you will hire already knows the model. If you are authenticating people through
a browser, or you want single sign-on and social login, this is the answer and
valiss is not a substitute.

**One point of honesty about the request path.** It is often said that OAuth
"puts an auth server in every request," and that is not quite fair. An OIDC
access token that is a JWT can be verified offline against the provider's public
keys, fetched once from a JWKS endpoint and cached. The tether that remains is
real but narrower: your verifiers still depend on the provider for key rotation
and its availability, opaque (non-JWT) tokens require an online introspection
call, and revoking a token before it expires generally means introspection too.

**Where valiss wins.** valiss pins one operator public key and needs nothing
else, so verification is fully offline with no JWKS refresh and no provider in
the loop at all. Issuing credentials never touches production: the operator seed
lives with the issuer and comes out only to sign. And delegation is per tenant.
Each account issues its own scoped-down users and services without calling you,
where OIDC models multi-tenancy centrally in the provider. The fit is
service-to-service and multi-tenant machine auth. A common and healthy end state
is both together, covered under [coexistence](#coexistence-and-exit) below: OIDC
keeps the humans, valiss carries the services.

## mTLS

Mutual TLS authenticates both ends of a connection with X.509 certificates at
the transport layer, and it is everywhere.

**Where it wins.** Identity is established at the connection itself, before a
byte of application data flows, and it comes with transport encryption in the
same handshake. The tooling is deeply mature, from OpenSSL through cert-manager
to service meshes that automate issuance and rotation, and it slots into
existing PKI you may already run.

**Where it costs.** The certificate lifecycle is the perennial pain: issuance,
distribution, rotation, and above all revocation through CRLs or OCSP, which is
notoriously awkward to operate well. And a certificate authenticates a subject
but carries no application-level claims. If you want to say "this caller may read
tenant 42's orders but not write them," you are bolting structured data onto
certificate fields or resolving it out of band.

**Where valiss differs.** valiss carries application-level identity together with
typed [grants](/docs/concepts/extensions/), and there is no certificate authority
in the hot path to keep reachable. The two are not strictly rivals: valiss
authenticates but does not encrypt (see the [security model](/docs/security/)),
so a common arrangement is valiss over TLS. Choose mTLS when you want mutual
transport authentication and encryption and you already operate PKI or a mesh.
Choose valiss when you want per-request application identity and typed
authorization that travels with the request through intermediate layers, without
running a CA or living with CRL and OCSP.

## SPIFFE and SPIRE

Of everything here, SPIFFE and SPIRE are the closest to valiss in spirit. SPIFFE
is a standard for workload identity, and SPIRE is its reference implementation.
Both give workloads short-lived, automatically rotated, offline-verifiable
identities instead of long-lived shared secrets, which is exactly valiss's
instinct.

**The defining difference is what you have to run.** SPIRE is infrastructure: a
central SPIRE Server that anchors the trust domain and issues identity documents
(SVIDs), plus a SPIRE Agent on every node that attests workloads locally and
serves them the Workload API. valiss is a library with no runtime components at
all, no server and no agent.

That difference cuts both ways, and SPIRE wins real things for it. Its agents
perform **attestation**: they prove *what* a workload is, from node and process
properties, before any identity is issued. valiss does not attest; it verifies
credentials that some issuer already decided to grant. SPIRE also delivers
automatic rotation to workloads and supports federation across trust domains,
and it is a mature CNCF project that integrates cleanly with mTLS and service
meshes.

Where valiss diverges is delegation. A SPIFFE identity is a path in a flat
namespace, and there is no tenant-delegation chain in which a tenant issues its
own sub-identities offline. valiss's operator, account, and user chain is exactly
that. Choose SPIRE when you want attested workload identity with automatic
rotation and you can run the server and agents. Choose valiss when you want
delegated tenant issuance and typed grants as a library with no infrastructure to
stand up.

## SSH certificates

SSH certificates are the most familiar instance of the idea at valiss's core: a
certificate authority signs a short-lived certificate for a principal, with a
validity window and a set of options, and verifiers trust anything the CA signed.
Anyone who has run an SSH CA already has valiss's mental model of a parent
signing a scoped, expiring credential for a child.

They are also simple and battle-tested, and for their job (authenticating SSH
access to hosts) they are the right tool and valiss is not a competitor. SSH even
has revocation, through key revocation lists, so that is not a valiss-only
capability.

valiss generalizes the same model in three ways. It is a **multi-level chain**
rather than a single CA-to-certificate hop: SSH certificates are signed directly
by a CA and the protocol has no notion of intermediate CAs or certificate chains,
whereas valiss walks operator to account to user. It carries **typed extension
claims** of your own definition, where an SSH certificate's extensions and
critical options are a constrained, largely fixed set. And its revocation is a
fail-closed [allowlist](/docs/concepts/allowlist/) keyed by content hash. Reach
for SSH certificates for SSH. Reach for valiss when you want that same
parent-signs-child shape extended to a tenant-delegation hierarchy with typed
authorization for application requests.

## Biscuit and macaroons

Biscuit and macaroons are authorization tokens built around *attenuation*: a
holder can take a token and, offline and without contacting the issuer, produce a
strictly more restricted version of it. Macaroons do this with chained
HMAC caveats; Biscuit does it with chained public-key-signed blocks, adds a
Datalog policy language, and supports third-party blocks that distribute
verification across parties. Both rest on genuine formal semantics, and this is
worth being generous about: their attenuation is more expressive than anything
valiss offers.

valiss's closest analogue is the way an [extension](/docs/concepts/extensions/)
present on both an account and a user token is enforced at both levels, so an
account-scoped grant caps every user the account issues. That is a real
capping-down-the-chain property, but it is a simpler, fixed-depth relative of
Biscuit's general attenuation: valiss's hierarchy is a fixed operator, account,
user chain, not an arbitrarily attenuable capability, and its grants are typed
claims rather than a policy language.

The two also frame the problem differently. valiss binds *identity*, who is
calling, to a chain rooted in a pinned key, with per-request proof of possession
as the default so a captured token is inert. Biscuit and macaroons are
capability-first, describing *what* the bearer may do, and are typically used as
bearer tokens. Choose Biscuit when you want rich, delegatable, offline-attenuable
capabilities with a policy language. Choose valiss when you want a fixed
tenant-identity hierarchy with typed grants and proof of possession, matched to
multi-tenant service authentication.

## When valiss fits, and when it does not

The [documentation hub](/docs/) states the fit in full and the
[security model](/docs/security/) states the boundaries; this section only points
at them rather than restating them. valiss fits multi-tenant APIs where each
customer is an account that issues its own credentials, machine-to-machine auth
where services carry keys and per-request signatures, and edge or on-prem
deployments where a verifier needs only the operator public key.

It is a poorer fit in three cases the hub names. If you want one central switch
to invalidate every credential at once, that is
[epoch rotation](/docs/concepts/rotation/), a domain-wide re-issue rather than
the per-request path. If a client cannot hold or protect any credential at all,
[custody](/docs/concepts/custody/) has a real gap today, because there is no
custodian server yet. And if a conventional session or OAuth stack already serves
a single-tenant application well, valiss buys you little.

## Coexistence and exit

**Adopting valiss alongside an existing identity provider.** valiss does not
have to replace anything. The cleanest adoption splits by audience: humans keep
authenticating through your OIDC provider and browser flows, while
service-to-service calls move to valiss. The two coexist without friction because
valiss adds no server and pins a single key. A gateway or service can accept an
OIDC session for a browser caller and a valiss credential for a machine caller on
the same route, each verified by its own middleware. Nothing about running valiss
constrains the identity provider, and nothing about the provider constrains
valiss.

**The exit story.** valiss is deliberately cheap to remove. Its tokens are
self-contained, so there is no session store to drain, no per-tenant key registry
to decommission, and no data migration: a verifying server held only a pinned
operator public key and an allowlist file, and nothing in your data model is
coupled to valiss. Removing it means removing the verification middleware from
your services and retiring the issuance pipeline that signed credentials. The
absence of stored state on the verifying side is the same property that makes
verification offline, seen from the other end.
