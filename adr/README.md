# Architecture Decision Records

Org-level decisions for `valiss-dev`, in [MADR](https://adr.github.io/madr/)
format. Governed by pull request (see [0005](0005-adr-process.md)).

> Temporary home. These will move to the docs/book repo once it is named and
> created; [0005](0005-adr-process.md) will be updated at that point.

| # | Title | Status |
|---|-------|--------|
| [0001](0001-go-module-path.md) | Go module path is a vanity import under valiss.dev | accepted |
| [0002](0002-repo-and-namespace-naming.md) | Namespace rooted at valiss.dev, per-language repos `valiss-<lang>` | superseded by [0007](0007-repository-naming-convention.md) |
| [0003](0003-go-vanity-import-hosting.md) | Serve the Go vanity import from static GitHub Pages with Porkbun DNS | accepted |
| [0004](0004-infrastructure-tooling.md) | Infrastructure as OpenTofu in a private infra repo | accepted |
| [0005](0005-adr-process.md) | ADRs are MADR-format and governed by pull request | accepted |
| [0006](0006-spec-versioning.md) | Wire spec versioned independently of library semver; current spec is 1 | accepted |
| [0007](0007-repository-naming-convention.md) | Repository naming convention | accepted |
| [0008](0008-python-package-structure.md) | Python ships as one valiss package with framework extras | accepted |
| [0009](0009-wire-format-versioning.md) | Wire-format version discriminator | accepted |
| [0010](0010-conformance-model.md) | Two-layer conformance: static vectors and a live interop matrix | accepted |
| [0011](0011-web-and-content-repos.md) | Web presentation and content repositories | accepted |
| [0012](0012-vector-immutability.md) | Conformance vectors are immutable and append-only | proposed |
| [0013](0013-library-lifecycle-interop-gate.md) | Non-patch library releases are gated on interop against the stable frontier | proposed |
| [0014](0014-go-framework-integration-targets.md) | Go framework integration targets selected by stars and effort | accepted |
| [0015](0015-contrib-auth-sig-separation.md) | Credential and proof-of-origin adapters are separate contrib packages | accepted |
| [0016](0016-vanity-module-roots.md) | One explicit vanity page per Go module root | accepted |
| [0017](0017-cli-tool.md) | The valiss CLI: name, repository, module path | accepted |
| [0018](0018-website-theme.md) | The website uses the Hextra theme, whole-site | accepted |
| [0019](0019-command-framework.md) | Command framework: the spf13 suite for CLI tools, stdlib flags for servers | accepted |
| [0020](0020-credential-storage.md) | Credential storage: pluggable backends, one encrypted DuckDB file per operator | proposed |

New ADR: copy [0000-template.md](0000-template.md) to the next number, open a PR
with `status: proposed`, flip to `accepted` before merge.
