# 0003. Serve the Go vanity import from static GitHub Pages with Porkbun DNS

- Status: accepted
- Date: 2026-07-15
- Deciders: mikluko

## Context

[0001](0001-go-module-path.md) requires `valiss.dev/valiss?go-get=1` to return a
`go-import` meta tag, over HTTPS, or `go get` fails. The meta must also cover
every subpackage (`valiss.dev/valiss/creds`, `valiss.dev/valiss/contrib/...`).

`valiss.dev` is registered at Porkbun, whose DNS is Cloudflare-backed and
supports A, AAAA, CNAME, ALIAS, TXT, NS, SRV, TLSA, CAA, HTTPS, SVCB records,
including ALIAS at the apex. Porkbun cannot host pages or edge functions.

`go get` walks *up* path segments on a 404: a request for `.../valiss/creds`
that 404s is retried at `.../valiss`, and the meta there (root
`valiss.dev/valiss`) is reused for all subpaths.

## Decision

- Serve the vanity page as **static GitHub Pages**. A single
  `/valiss/index.html` carrying
  `<meta name="go-import" content="valiss.dev/valiss git https://github.com/valiss-dev/valiss-go">`
  covers the module and, via go's walk-up behaviour, every subpackage. No
  per-package generation and no runtime.
- **DNS via Porkbun**: `ALIAS @ -> valiss-dev.github.io` (apex, survives GitHub
  IP changes), `CNAME www -> valiss-dev.github.io`. The Pages repo carries a
  `CNAME` file `valiss.dev`; GitHub auto-provisions the TLS cert and HTTPS is
  enforced.

## Consequences

- Zero runtime infrastructure and no Cloudflare dependency; the whole vanity
  surface is one HTML file plus DNS records.
- New subpackages need no action — walk-up reuses the root meta.
- A new disjoint import prefix (a second module under `valiss.dev`) would need
  its own meta page.
- DNS is managed by a community Porkbun provider (see
  [0004](0004-infrastructure-tooling.md)).

## Alternatives considered

- Cloudflare Worker/Pages — clean and dynamic, but requires moving the zone to
  Cloudflare and adds a vendor for what one static file solves.
- Dynamic responder on Vercel / Fly / Cloud Run / AWS — unnecessary given a
  finite, prefix-covered package set.
- Per-package generated pages — not needed once go's walk-up behaviour is
  confirmed.
- Hardcoded apex A records (`185.199.108-111.153`) — brittle vs a single ALIAS.
