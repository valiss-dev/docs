---
title: Project status and support
weight: 8
description: "Where valiss stands in its lifecycle: what each implementation ships today, what is planned, and how support works."
---

This page states plainly where valiss is in its lifecycle, what each
implementation can do today, who maintains it, how to report a security issue,
and how it is licensed. It is meant to be read before you depend on valiss in
production, so nothing here is softened.

## Maturity

valiss is pre-1.0. That statement is about the libraries, not the wire format,
and the distinction matters because the two are versioned separately (see
[Versioning and compatibility](/docs/versioning/)).

The wire format is versioned and **spec 1 is stable**: a verifier that
implements spec 1 verifies any spec-1 credential for all time, and the spec
document is frozen once accepted. The **libraries are pre-1.0**, each moving at
its own semantic version (valiss-go on the v0.13.x line, valiss-py on 0.8.x,
valiss-ts on 0.1.x). While libraries are pre-1.0 and the wire is not yet frozen,
nearly every release touches versioned surface, so the release cadence is
frequent and the public API of a given language may still shift between minor
versions. You reason about interoperability through the spec version, never
through a library's semantic version.

## Implementation status

| Component        | Language / repo        | Status                                                                      |
| ---------------- | ---------------------- | --------------------------------------------------------------------------- |
| Reference        | Go (valiss-go)         | Full. Source of truth for the scheme; issue, verify, transports, adapters.  |
| Port             | Python (valiss-py)     | Full port: issue, sign, verify, plus Django and ASGI integrations. On PyPI. |
| Core wire layer  | TypeScript (valiss-ts) | Sign and verify primitives only. Source-only, not published to npm.         |
| Issuer CLI       | valiss-cli             | Designed and in active development. Not released.                           |
| Custodian server | (none yet)             | Planned. Not implemented.                                                   |

The Go implementation is complete and canonical: where any other implementation
disagrees with it, the Go source and the spec are right. The Python port is a
full implementation published to PyPI. The TypeScript package implements the
core sign and verify primitives of the wire layer and is consumed from source;
it is not published to a registry. The CLI is designed and under development but
has not been released. There is no custodian server yet, which is the known gap
for clients that cannot hold or protect any credential at all.

## Maintenance and support

valiss is a single-maintainer open-source project. There is no commercial entity
behind it, no service-level agreement, and no support commitment of any kind.
Community support runs through GitHub issues on the relevant repository, answered
on a best-effort basis. Treat that as the whole of what is on offer, and size
your dependence on valiss accordingly.

## Security disclosure

Report suspected vulnerabilities privately. Do not open a public issue for a
security problem.

Use GitHub's private vulnerability reporting on the affected repository under the
[valiss-dev](https://github.com/valiss-dev) organization: open the repository's
**Security** tab and choose **Report a vulnerability**. That routes the report
privately to the maintainer. Each repository also carries a `SECURITY.md` with
the same instructions. Response is best-effort by a single maintainer with no
SLA, and coordinated disclosure is preferred.

On the state of assurance, honestly: **no third-party security audit has been
performed.** The current assurance story is not an audit badge but the
machinery that makes conformance checkable. It rests on three things: the
published wire specification (SPEC-1), the append-only conformance vectors that
pair each artifact with its expected verify outcome, and the cross-language
interop gate that proves implementations agree on a real exchange. All three are
described in [Versioning and compatibility](/docs/versioning/). They constrain
what an implementation may do and make divergence detectable; they are not a
substitute for an independent review, and none is claimed to be.

## License

valiss is MIT-licensed across its implementation repositories: valiss-go,
valiss-py, valiss-ts, and valiss-cli each ship an MIT `LICENSE`. Dependencies
carry their own licenses. In particular the Go implementation depends on
[nats-io/nkeys](https://github.com/nats-io/nkeys) for nkey encoding, and the
Python implementation depends on the
[cryptography](https://pypi.org/project/cryptography/) package for Ed25519.
Review the dependency licenses that apply to the implementation you ship.
