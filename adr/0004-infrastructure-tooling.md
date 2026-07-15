# 0004. Infrastructure as OpenTofu in a private infra repo

- Status: accepted
- Date: 2026-07-15
- Deciders: mikluko

## Context

The org needs its GitHub repos, `valiss.dev` DNS, and Go vanity hosting managed
as code. The footprint is small (a handful of repos, DNS records, GitHub Pages)
and currently maintained by one person. HashiCorp's BSL relicensing and IBM
ownership are a concern for a public open-source project's tooling.

## Decision

- Infrastructure is defined with **OpenTofu** (not Terraform) to avoid the BSL /
  vendor-lock concern; OpenTofu is drop-in and adds native state encryption.
- Code lives in a **private** repo, `valiss-dev/infra` (bare name per
  [0007](0007-repository-naming-convention.md): it is project-internal, not a
  distributable artifact).
- **Providers**: the GitHub provider manages repos, branch protection, labels,
  and settings, with existing repos brought in via `import` blocks. DNS is
  managed by a **community Porkbun provider** (`marcfrederick/porkbun`
  preferred for its DNSSEC support), version-pinned with a committed lock file.
- **GitHub auth is a GitHub App** (not a PAT): scoped, short-lived installation
  tokens, decoupled from any personal account. Repository permissions
  Administration/Contents/Pages (read-write) plus Metadata; installed org-wide
  with All-repositories access.
- **State**: stored in-repo to start, with **OpenTofu state encryption enabled**
  (passphrase or KMS key provider). Applies are serialized (CI or discipline)
  since a file backend has no locking. `.terraform/` and plan files are
  gitignored.

## Consequences

- No SaaS control plane and no cloud account required to begin.
- State encryption neutralizes the secrets-in-git risk even though no
  secret-bearing resources are planned today.
- No state locking: concurrent applies can corrupt state; acceptable while solo.
- Porkbun has no official provider, so a community one is a dependency risk;
  mitigated by pinning, the lock file, and the option to fork or move DNS to a
  first-class provider (Route53 / Cloud DNS / DNSimple) later.

## Alternatives considered

- **Terraform Cloud / HCP** — resource-based pricing and BSL/IBM trust concerns;
  moot at this scale but points away from HashiCorp regardless.
- **env0 / Scalr** — strong control planes, but their strengths (ephemeral
  environments, cost governance, many workspaces) do not apply here. Scalr's
  free tier remains the fallback if the repo grows secret-bearing resources or a
  second maintainer, in which case migrate state off git.
- **R2 / S3 remote backend** — solves locking and keeps state out of git, but
  requires standing up a cloud account for a footprint this small.
