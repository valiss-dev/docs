# docs

Pure-Markdown content for the valiss project. No Hugo config, no theme: the
`website` repo mounts this as a Hugo Module and renders it under `valiss.dev`
(ADR 0011). Author plain Markdown here; presentation is the website's job.

## Layout

```
guides/   user documentation: quickstarts (Django, ASGI, Go), how-tos
adr/      architecture decision records (PR-governed, ADR 0005)
```

## ADRs

Decision records live in `adr/` and are governed by pull request (ADR 0005):
propose as `status: proposed` in a PR, flip to `accepted` before merge. See
`adr/README.md`.
