---
name: rakunlabs-go
description: >-
  Use when building, scaffolding, or wiring a Go service/microservice with the
  rakunlabs library stack (github.com/rakunlabs/*): into, logi, chu, gofret,
  tell, ada, ok, cache, query, muz, types, bw, alan, hardloop,
  tummy. Covers the cmd/internal/_ui project layout, the canonical main.go
  run(ctx) wiring, chu config, the ada middleware stack and context handlers,
  embedding a Svelte UI, per-library quickstarts, and the ada Auth Kit (login
  page, local users, roles and permissions, user admin page, OAuth 2.1 / MCP
  authorization server, personal access tokens, SSO). Trigger on "rakunlabs
  service", "new Go service with chu/ada/into", "into.Init", "chu.Load",
  "ada.New", "ada.Context", "StartWithContext", "muz.Migrate", "SQLDriver",
  "gofret.To", "hardloop.NewCron", "tummy.Now", "add ada Auth Kit", "add
  auth/login/users/roles to an ada app", "MCP OAuth", "oauthserver".
metadata:
  org: rakunlabs
  language: go
---

# rakunlabs Go service skill

How to build a Go service the rakunlabs way. The ecosystem is a set of small,
single-purpose libraries that compose into a service, plus a family of
applications (pika, kafka-console, dataport, kutu, calendar, at) that all share
the same shape: one binary, config from `chu`, an `ada` HTTP server, and an
embedded Svelte UI.

All packages live under `github.com/rakunlabs/<name>` (exception: `bucket`
keeps the module path `github.com/worldline-go/bucket`). Do not invent APIs.
If a README and the code disagree, the code wins: check the signature on
pkg.go.dev.

This file is the index. Read only the reference files the task needs.

## When to use

- Creating a new `github.com/rakunlabs/<app>` Go service from scratch.
- Wiring config loading, telemetry, an HTTP server, storage, background jobs,
  or graceful shutdown into an existing rakunlabs service.
- Adding authentication to an ada app ("add ada Auth Kit").
- Using any of the libraries in the map below.

## References

| Task | Read |
| --- | --- |
| Directory layout of a new service | [references/layout.md](references/layout.md) |
| `main.go`, `into.Init`, exit codes, health/kill endpoints | [references/main-go.md](references/main-go.md) |
| Config struct, `chu.Load`, env prefix, secret backends | [references/config-chu.md](references/config-chu.md) |
| ada server, middleware stack, context handlers, embedded UI | [references/http-ada.md](references/http-ada.md) |
| logi, tell, gofret, ok, cache, query, muz, types, bw, alan, hardloop, tummy | [references/libraries.md](references/libraries.md) |
| **Auth Kit**: login, users, roles, admin page, OAuth/MCP, tokens, SSO | [references/auth-kit.md](references/auth-kit.md) (start here, it links the rest) |

## The library map

| Library | Role | Entry point |
| --- | --- | --- |
| `into`     | App lifecycle: run a func, signals, graceful shutdown, health/kill endpoints | `into.Init(run, opts...)` |
| `logi`     | slog initializer (color on TTY, JSON otherwise) | `logi.InitializeLog()` |
| `chu`      | Layered config loader (default→file→http→env + secret backends) | `chu.Load(ctx, name, &cfg, opts...)` |
| `gofret`   | Map↔struct↔struct conversion (the decoder under `chu`) | `gofret.To[T](in)` |
| `tell`     | OpenTelemetry metrics + traces collector | `tell.New(ctx, cfg.Telemetry)` |
| `ada`      | net/http web framework, context handlers, runtime route/middleware reload | `ada.New()` |
| `ada/middleware/auth` | Login UI, sessions, strategies, OAuth 2.1 server | `auth.New(cfg)` / `oauthserver.New(cfg, store)` |
| `ok`       | Retryable HTTP client | `ok.New(opts...)` |
| `cache`    | Generic cache over memory/redis | `cache.New[K,V](ctx, store, opts...)` |
| `query`    | URL query string → filter/sort/paging expression | `query.Parse(rawQuery, opts...)` |
| `muz`      | SQL migration runner (PostgreSQL, MySQL, SQLite, MSSQL) | `muz.Migrate{...}.Migrate(ctx, driver)` |
| `types`    | Nullable JSON/SQL column types (`Map`, `JSON`, `Null`, `Slice`, `Time`) | `types.Null[T]` |
| `bw`       | BadgerDB typed buckets + `query` engine (+ `bw/cluster`) | `bw.Open(path)` |
| `alan`     | QUIC peer discovery, distributed locks, leader election | `alan.New(alan.Config{...})` |
| `hardloop` | Cron jobs and functions that run inside cron time windows | `hardloop.NewCron(...)` / `hardloop.NewLoop(...)` |
| `tummy`    | Test helper for deterministic time-dependent code | `tummy.Enable()` / `tummy.Now()` |

Smaller helpers: `mapx` (map merge/compare), `contextx` (typed context
values), `bucket` (bounded goroutine batch processing).

## Conventions & gotchas

- **One env prefix per app** (`<APP>_`) via `loaderenv.WithPrefix`.
- **Mask secrets** with `log:"-"`; never log a raw config struct, use
  `chu.MarshalMap(cfg)`.
- **Embed `tell.Config`** in the app config so telemetry shares the config
  pipeline.
- **`defer` order in `run`** = graceful shutdown order; telemetry last.
- **Start ada with `StartWithContext(ctx, addr)`**, not `Start`, so shutdown
  follows the `run` ctx.
- **Module path** is always `github.com/rakunlabs/<app>`; keep packages under
  `internal/` and dependencies pointing downward.
- **Errors** wrap with `%w` and a short `lower-case; %w` prefix.
- **Sub-modules**: ada middlewares/handlers and chu external loaders each need
  their own `require` line; run `go mod tidy` after adding imports.
- **Verify** with `go vet ./...` and `go test ./...` before finishing.

## Deep-dive references

- into     · https://pkg.go.dev/github.com/rakunlabs/into
- logi     · https://pkg.go.dev/github.com/rakunlabs/logi
- chu      · https://pkg.go.dev/github.com/rakunlabs/chu
- gofret   · https://pkg.go.dev/github.com/rakunlabs/gofret
- tell     · https://pkg.go.dev/github.com/rakunlabs/tell
- ada      · https://rakunlabs.github.io/ada/
- ok       · https://pkg.go.dev/github.com/rakunlabs/ok
- cache    · https://pkg.go.dev/github.com/rakunlabs/cache
- query    · https://pkg.go.dev/github.com/rakunlabs/query
- muz      · https://pkg.go.dev/github.com/rakunlabs/muz
- types    · https://pkg.go.dev/github.com/rakunlabs/types
- bw       · https://pkg.go.dev/github.com/rakunlabs/bw
- alan     · https://pkg.go.dev/github.com/rakunlabs/alan
- hardloop · https://pkg.go.dev/github.com/rakunlabs/hardloop
- tummy    · https://pkg.go.dev/github.com/rakunlabs/tummy

Reference implementations: `github.com/rakunlabs/kafka-console` (compact
server + embedded UI), `github.com/rakunlabs/pika` (bw storage, alan cluster,
auth), `github.com/rakunlabs/dataport` (SQL store, muz, hardloop jobs),
`github.com/rakunlabs/ada/_examples/authkit` (Auth Kit).
