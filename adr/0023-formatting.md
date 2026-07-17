# 0023. Formatting is enforced: canonical formatters for code, dprint for Markdown

- Status: accepted
- Date: 2026-07-17
- Deciders: mikluko

## Context

Source and prose across the valiss repos drift in layout: table columns fall out
of alignment, emphasis markers mix `_` and `*`, trailing whitespace and
inconsistent list markers accumulate. Every such difference is a line a reviewer
has to read past, and worse, a line a contributor can flip back and forth so that
diffs carry churn no one decided on. Code already answers this: Go is run through
`gofmt`, which is canonical, non-negotiable, and unconfigurable by design.
Markdown, the documentation and the ADRs in this repo, has had no equivalent, so
the discipline that holds for `.go` files stops at the `.md` boundary.

The question is whether formatting should be advisory (a convention people mostly
follow) or enforced (a gate that fails a build), and if enforced, which tool
formats Markdown without a heavy toolchain and without mangling the YAML
front matter that Hugo depends on.

## Decision

**Formatting is enforced, not advisory.** A file that is not in its formatter's
canonical form fails CI. Layout stops being a matter of reviewer attention or
contributor habit; the tool decides and the gate holds the line.

**Code is formatted by each language's canonical formatter.** For Go that is
`gofmt`, and this ADR decides nothing new there: it states the standing baseline
so the enforcement principle is on record for every language. As other languages
join, each adopts its own canonical formatter under the same rule.

**Markdown is formatted by dprint with the markdown plugin.** Documentation and
ADRs are formatted by [dprint](https://dprint.dev) running its markdown plugin.
The configuration lives in a `dprint.json` at each adopting repo's root, and the
plugin version is pinned by URL to a specific WASM artifact, so a formatter
upgrade is an explicit, reviewable change rather than something that drifts under
the repo. CI enforces the format with `dprint check`, which reports any file that
is not already formatted and exits non-zero.

**Adoption is per-repo.** Each repo opts in by committing its own `dprint.json`
and a CI job; there is no org-wide flag day. This repo (docs) adopts now.

**The spec repo is explicitly excluded for now.** SPEC files are freeze-governed
under [0006](0006-spec-versioning.md): the normative wire specification is frozen
at a version, and reformatting normative content, even whitespace, is a change to
a frozen artifact. Whether and how to format the spec is its own decision, to be
taken separately; until then dprint does not touch that repo.

## Consequences

- Layout differences leave code review. A malformed table or a stray emphasis
  style is caught by the gate, not by a human, and cannot be reintroduced without
  failing CI.
- Markdown diffs shrink to real content changes, since everyone commits from the
  same canonical form. Table alignment in particular stops generating noise.
- Adopting a repo costs one `dprint.json` and one CI job. The formatter is a
  single downloaded binary plus a pinned WASM plugin, with no language runtime to
  install.
- A formatter upgrade is a deliberate act: bump the plugin URL, re-run, review
  the reformatting as its own commit. The pin means CI and local runs always
  agree on the exact plugin build.
- The front matter Hugo reads is preserved: the markdown plugin formats the body
  and leaves the YAML block intact.
- The spec repo stays untouched, so this decision creates no pressure on the
  freeze. The cost is that formatting coverage is not yet universal; closing that
  gap is deferred to a spec-specific decision.
- dprint's configuration surface extends to JSON, TOML, and YAML through
  additional pinned plugins, so a repo that later wants those formatted adds a
  plugin rather than a second tool.

## Alternatives considered

- **Prettier.** The de facto standard for Markdown and web formats, with the
  largest ecosystem and the most predictable output. Rejected as the default
  because it pulls in a Node.js runtime and an `npm` dependency tree, a
  disproportionate toolchain for repos that are otherwise Go and prose, and a
  runtime none of these repos would otherwise carry.
- **mdformat.** A Python Markdown formatter, CommonMark-strict and pluggable.
  Rejected on the same shape of cost as Prettier (a language runtime plus a
  plugin set to assemble for tables, front matter, and the like) without a
  decisive advantage over dprint's single binary.
- **mdox.** Go-native, which fits the Go-centric toolchain and formats plus
  validates links. Rejected because its formatting is narrower and its ecosystem
  and momentum are markedly smaller than dprint's, and it does not generalize to
  the other file types dprint can grow into.
- **markdownlint.** Widely used and valuable, but it is a linter, not a
  formatter: it flags violations and does not rewrite tables into aligned form.
  It answers a different question (which rules are broken) than the one here
  (make every file canonical), and could complement a formatter later rather than
  replace one.

dprint is chosen for the combination the alternatives each miss: a single static
binary with no language runtime, fast enough to run on every commit, front
matter safe so Hugo keeps working, plugins pinned by URL to an exact WASM build
for reproducibility, and one tool that extends to JSON, TOML, and YAML if those
are ever wanted.
