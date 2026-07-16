# 0014. Go framework integration targets selected by stars and effort

- Status: accepted
- Date: 2026-07-16
- Deciders: mikluko

## Context

The Go implementation (`valiss-go`) ships adapters as subpackages under
`contrib/` (ADR [0008](0008-python-package-structure.md) mirrors this layout
for Python). The existing `contrib/httpauth` middleware has the standard
signature `func(http.Handler) http.Handler`, so anything that routes plain
`net/http` handlers is covered without further work. Frameworks with their own
handler and context types (Gin, Fiber, Echo, …) need dedicated adapters that
expose the verified `valiss.Identity` through the framework's native context
and use its native abort semantics.

The question is which frameworks get dedicated adapters, in what order, and
where the line is drawn. We want an objective, cheap-to-re-evaluate criterion
rather than case-by-case debate.

## Decision

Select integration targets by **GitHub star count** and order the work by
**effort class first, stars within the class**, subject to two qualifying
conditions:

1. the project is **actively maintained** (commits and releases within the
   last year, not in announced maintenance mode), and
2. it actually **needs an adapter** (its middleware chain cannot consume
   `func(http.Handler) http.Handler` directly).

Effort classes follow from how the verified identity travels: it rides the
request `context.Context`, so any framework that wraps `*http.Request` gets a
thin adapter reusing `httpauth`'s verification core (**S**); a framework with
its own HTTP engine and no `net/http` types needs its own
request-canonicalization path against the wire spec (**L**). All S adapters
ship before any L adapter: the whole S class costs roughly one L adapter.

**Cutoff: 7,000 stars.** At the snapshot below this sits in a natural gap
(Hertz at 7.3k, next qualifying candidate Goravel at 4.8k). Re-evaluate when a
watchlist project crosses the cutoff or a target stops qualifying.

Adapters are `contrib/` subpackages named `<framework>auth` (credential
middleware, mirroring `httpauth`/`grpcauth`); message-token counterparts, when
demanded, take `<framework>sig`. **First wave, implemented alongside this
ADR: `contrib/ginauth` and `contrib/echoauth`.**

Star counts snapshot 2026-07-16.

### Tier 0 — already covered by `contrib/httpauth`, document only

| Framework   | Repo                       |  Stars |
|-------------|----------------------------|-------:|
| net/http    | (stdlib)                   |      — |
| chi         | `go-chi/chi`               | 22,545 |
| gorilla/mux | `gorilla/mux`              | 21,845 |
| httprouter  | `julienschmidt/httprouter` | 17,136 |
| negroni     | `urfave/negroni`           |  7,529 |
| huma        | `danielgtaylor/huma`       |  4,243 |

These consume standard middleware as-is (httprouter and huma at the router /
underlying-router level). Work item: a guide showing wiring per router, no new
packages.

### Tier 1 — dedicated adapters, in priority order

| # | Framework | Repo                |  Stars | Effort | Note                                                                                   |
|---|-----------|---------------------|-------:|:------:|----------------------------------------------------------------------------------------|
| 1 | Gin       | `gin-gonic/gin`     | 88,920 |   S    | own `gin.Context` / `gin.HandlerFunc`                                                  |
| 2 | go-zero   | `zeromicro/go-zero` | 33,193 |   S    | rest middleware is `func(http.HandlerFunc) http.HandlerFunc`, thin shim                |
| 3 | Echo      | `labstack/echo`     | 32,527 |   S    | `echo.MiddlewareFunc`; `WrapMiddleware` exists but native adapter gives context access |
| 4 | Beego     | `beego/beego`       | 32,413 |   S    | own filter chain                                                                       |
| 5 | Kratos    | `go-kratos/kratos`  | 25,794 |   S    | transport middleware over HTTP and gRPC; may reuse `grpcauth`                          |
| 6 | Iris      | `kataras/iris`      | 25,571 |   S    | own context                                                                            |
| 7 | GoFrame   | `gogf/gf`           | 13,219 |   S    | own `ghttp` server                                                                     |
| 8 | Fiber     | `gofiber/fiber`     | 39,965 |   L    | fasthttp-based, no `net/http` types at all                                             |
| 9 | Hertz     | `cloudwego/hertz`   |  7,306 |   L    | own HTTP engine (netpoll), no `net/http` types                                         |

### Excluded

| Framework | Repo                 |  Stars | Reason                                                                                                                                                          |
|-----------|----------------------|-------:|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| fasthttp  | `valyala/fasthttp`   | 23,417 | HTTP engine, not a framework; the Fiber adapter covers its dominant use. A raw fasthttp middleware could later underpin Fiber/Atreugo/Gearbox at once; deferred |
| Revel     | `revel/revel`        | 13,223 | effectively unmaintained                                                                                                                                        |
| Martini   | `go-martini/martini` | 11,607 | unmaintained since ~2017; stars are historical                                                                                                                  |
| Buffalo   | `gobuffalo/buffalo`  |  8,405 | maintenance mode, development stalled                                                                                                                           |

### Watchlist (below cutoff)

Goravel (4,781), Fuego (1,753), Atreugo (1,304), Gearbox (803). Promote when
one crosses 7,000 and qualifies.

## Consequences

- Priority order is mechanical: re-run the star query, re-sort within the
  effort classes, done. No judgment calls beyond the two qualifying
  conditions.
- Nine adapters is a committed surface. Mitigation: the S class lands first
  and each S adapter is a small package wrapping the same `httpauth` core, so
  the expensive tail (Fiber, Hertz) can slip without hurting the rest.
- Fiber is #2 by stars yet ships last-but-one: its effort class dominates.
  Interim path for Fiber users: fiber's `net/http` adaptor middleware can
  wrap `httpauth.NewMiddleware` at a per-request conversion cost; worth
  documenting, not packaging.
- Star count proxies reach, not fit: Beego and Iris rank high while skewing to
  a market segment we have no adoption signal from. Accepted; the objective
  criterion is worth more than per-framework debate, and order of shipping is
  the only thing at stake.
- Fiber and Hertz adapters cannot delegate to `httpauth`'s request parsing
  (no `*http.Request`); they need their own request-canonicalization path
  against the wire spec. These two are the expensive ones.
- The snapshot goes stale. The cutoff rule, not the numbers, is the decision;
  refreshing the table does not require a new ADR.

## Alternatives considered

- **Download/dependents count (pkg.go.dev imported-by)** — closer to actual
  usage than stars, but not queryable as one sorted list and noisy for forks;
  stars are a good-enough proxy and trivially reproducible.
- **Demand-driven (build when a user asks)** — avoids speculative work but
  makes the roadmap illegible and cedes the "batteries included" first
  impression that a security library needs to compete with established auth
  middleware.
- **Everything above 1k stars** — pulls in a long tail of fasthttp
  micro-frameworks with tiny user bases each; the 7k gap keeps the set at nine
  adapters, all with distinct user populations.
