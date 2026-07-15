# 0008. Python ships as one valiss package with framework extras

- Status: accepted
- Date: 2026-07-15
- Deciders: mikluko

## Context

The Python implementation (`valiss-py`) provides the core scheme plus
integrations for several web frameworks: a generic ASGI middleware
(FastAPI, Starlette, Litestar, Quart), a Django integration (middleware and
DRF auth), and eventually WSGI and Flask. The question is whether each
integration is its own PyPI distribution (`valiss`, `valiss-asgi`,
`django-valiss`, …) or whether one distribution carries them all.

The Go reference already answers the equivalent question: its adapters ship as
subpackages inside the single module (`contrib/httpauth`, `contrib/grpcauth`,
`contrib/httpsig`, `contrib/grpcsig`), not as separate modules. Cross-language
structural consistency is a goal of the framework.

## Decision

Ship **one PyPI distribution, `valiss`**, from the `valiss-py` repository.
Framework integrations are **submodules** of that package, each gated by an
**optional-dependency extra** so the core install pulls no web dependencies:

```
valiss/
  __init__.py     core: mint/verify, creds, request signing
  asgi.py         valiss.asgi     pip install valiss[asgi]
  wsgi.py         valiss.wsgi
  django/         valiss.django   pip install valiss[django]
  flask.py        valiss.flask
```

```toml
[project.optional-dependencies]
asgi   = ["asgiref>=3.7"]
django = ["django>=4.2"]
flask  = ["flask>=3.0"]
```

- Adapter submodules **lazy/guard-import** their framework, so `import
  valiss.django` without Django installed fails with a clear message rather
  than an obscure ImportError, and the core stays importable without any web
  dependency.
- One repository (`valiss-py`), one version, one release pipeline, one PyPI
  trusted publisher. This mirrors the Go `contrib/*` layout.

## Consequences

- **Cross-language parity**: the same "one module, framework subpackages" shape
  as Go, reinforcing [0007](0007-repository-naming-convention.md) (one
  distributable per ecosystem).
- **Lean, low-surface core**: extras are opt-in, so a core install carries no
  framework code paths' dependencies; and there is a single publisher,
  environment, and supply-chain surface to secure rather than one per adapter,
  which matters for a security library.
- **Coupled versioning**: core and adapters release together. Pre-1.0, while
  they co-evolve, this is a simplification, not a limitation.
- **Django discoverability cost**: there is no `django-valiss` on PyPI for
  Django users who search `django-*`. Mitigation, if Django adoption warrants
  it: publish a thin `django-valiss` shim whose only content is a dependency on
  `valiss[django]` and a re-export. Deferred until needed.
- **Split-on-demand escape hatch**: if a single adapter later needs a very
  different release cadence or a heavy or conflicting dependency, that one
  adapter can be spun out into its own distribution without disturbing the
  rest.

## Alternatives considered

- **One PyPI distribution per adapter** (`valiss-asgi`, `django-valiss`, …) —
  gives independent versioning and ecosystem-idiomatic names, but multiplies
  repos, pipelines, and publishers (more supply-chain surface), diverges from
  the Go layout, and front-loads a cross-package compatibility matrix while the
  core API is still unstable. Revisitable per adapter via the split-on-demand
  hatch above.
- **Bundling framework dependencies unconditionally** (no extras) — rejected: it
  would force every consumer of the core to pull Django, Starlette, Flask, etc.
