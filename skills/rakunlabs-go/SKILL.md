---
name: rakunlabs-go
description: >-
  Use when building, scaffolding, or wiring a Go service/microservice with the
  rakunlabs library stack — into, logi, chu, tell, ada, ok, cache, query, muz,
  bw, alan, tummy (github.com/rakunlabs/*). Covers the cmd/internal project layout, the
  canonical main.go run(ctx) wiring, chu config, the ada middleware stack, and
  per-library quickstarts. Trigger on "rakunlabs service", "new Go service with
  chu/ada/into", "into.Init", "chu.Load", "ada.New", "muz.Migrate", "SQLDriver", "tummy.Now".
metadata:
  org: rakunlabs
  language: go
---

# rakunlabs Go service skill

How to build a Go service the rakunlabs way. The ecosystem is a set of small,
single-purpose libraries that compose into a service. This skill captures the
standard project layout, the canonical `main.go` wiring, and a copy-paste
quickstart for each library.

All packages live under `github.com/rakunlabs/<name>`. When you need depth
beyond this file, read the library's `README.md` / pkg.go.dev (links at the
bottom) — do not invent APIs.

## When to use

- Creating a new `github.com/rakunlabs/<app>` Go service from scratch.
- Wiring config loading, telemetry, an HTTP server, storage, or graceful
  shutdown into an existing rakunlabs service.
- Using any of: `into`, `logi`, `chu`, `tell`, `ada`, `ok`, `cache`, `query`,
  `muz`, `bw`, `alan`, `tummy`.

## The library map

| Library | Role | Entry point |
| --- | --- | --- |
| `into`  | App lifecycle: run a func, handle signals, graceful shutdown | `into.Init(run, opts...)` |
| `logi`  | slog initializer (color on TTY, JSON otherwise) | `logi.InitializeLog()` |
| `chu`   | Layered config loader (default→file→http→env + secret backends) | `chu.Load(ctx, name, &cfg, opts...)` |
| `tell`  | OpenTelemetry metrics + traces collector | `tell.New(ctx, cfg.Telemetry)` |
| `ada`   | net/http web framework with runtime middleware reload | `ada.New()` |
| `ok`    | Retryable HTTP client | `ok.New(opts...)` |
| `cache` | Generic cache over memory/redis | `cache.New[K,V](ctx, store, opts...)` |
| `query` | URL query string → filter/sort/paging expression | `query.Parse(rawQuery, opts...)` |
| `muz`   | SQL migration runner for PostgreSQL-style databases | `muz.Migrate{...}.Migrate(ctx, driver)` |
| `bw`    | BadgerDB typed buckets + `query` engine (+ `bw/cluster`) | `bw.Open(path)` |
| `alan`  | QUIC peer discovery, distributed locks, leader election | `alan.New(alan.Config{...})` |
| `tummy` | Test helper for deterministic time-dependent code | `tummy.Enable()` / `tummy.Now()` |

## Standard project layout

Mirror `github.com/rakunlabs/pika`:

```
<app>/
  cmd/<app>/main.go        # entrypoint: into.Init(run, ...)
  internal/
    config/config.go       # Config struct + chu.Load (this app's env prefix)
    server/server.go       # ada.New(), middleware stack, route groups
    service/               # business logic (transport-agnostic)
    storage/               # data layer (bw, sql, etc.)
  go.mod                   # module github.com/rakunlabs/<app>
  Makefile
  .goreleaser.yaml         # release/build via goreleaser
  AGENTS.md                # repo-specific agent rules (optional)
```

Conventions:
- `module github.com/rakunlabs/<app>`, current Go toolchain.
- `internal/` holds everything app-specific; packages depend downward
  (`server` → `service` → `storage`), never the reverse.
- Build metadata is injected via ldflags into `cmd/<app>/main.go`:
  `var version, commit, date string`.

## The canonical `main.go`

`into.Init` owns the process lifecycle. It runs `run(ctx)`, wires a logger and
a startup banner, and cancels `ctx` on SIGINT/SIGTERM so every `defer` in
`run` unwinds for a graceful shutdown. Keep `main` tiny; put all wiring in
`run`.

```go
package main

import (
	"context"
	"fmt"

	"github.com/rakunlabs/into"
	"github.com/rakunlabs/logi"
	"github.com/rakunlabs/tell"

	"github.com/rakunlabs/<app>/internal/config"
	"github.com/rakunlabs/<app>/internal/server"
	"github.com/rakunlabs/<app>/internal/service"
	"github.com/rakunlabs/<app>/internal/storage"
)

// Injected at build time via -ldflags.
var (
	version = "v0.0.0"
	commit  = "-"
	date    = "-"
)

func main() {
	into.Init(run,
		into.WithLogger(logi.InitializeLog(logi.WithCaller(false))),
		into.WithMsgf("%s version:[%s] commit:[%s] date:[%s]",
			config.ServiceName, version, commit, date),
	)
}

func run(ctx context.Context) error {
	cfg, err := config.Load(ctx)
	if err != nil {
		return err
	}

	// Telemetry first so everything downstream is observable.
	collector, err := tell.New(ctx, cfg.Telemetry)
	if err != nil {
		return fmt.Errorf("init telemetry; %w", err)
	}
	defer collector.Shutdown()

	// Storage.
	store, err := storage.New(ctx, &cfg.Storage)
	if err != nil {
		return fmt.Errorf("init storage; %w", err)
	}
	defer store.Close()

	// Service (business logic) wraps storage.
	svc := service.New(store)

	// Server blocks until ctx is cancelled, then shuts down.
	if err := server.Start(ctx, cfg, svc); err != nil {
		return fmt.Errorf("start server; %w", err)
	}

	return nil
}
```

Rules:
- `defer` cleanup in reverse dependency order — the signal-cancelled `ctx`
  triggers them. Telemetry shutdown last, so spans/metrics flush after the
  server stops.
- Wrap every error with `fmt.Errorf("...; %w", err)`.
- Never call `os.Exit` inside `run`; return an error and let `into` handle it.

## Config with `chu`

One `Config` struct per app, populated by `chu.Load`. Tags: `cfg:"..."` names
the field across all loaders, `default:"..."` provides defaults, `log:"-"`
masks secrets from the config log line.

```go
package config

import (
	"context"
	"fmt"
	"log/slog"

	"github.com/rakunlabs/chu"
	"github.com/rakunlabs/chu/loader/loaderenv"
	"github.com/rakunlabs/logi"
	"github.com/rakunlabs/tell"

	// Optional secret backends — blank-import only the ones you need.
	_ "github.com/rakunlabs/chu/loader/external/loadervault"
	// _ "github.com/rakunlabs/chu/loader/external/loaderconsul"
	// _ "github.com/rakunlabs/chu/loader/external/loadergcpsecret"
	// _ "github.com/rakunlabs/chu/loader/external/loadergcpparameter"

	"github.com/rakunlabs/<app>/internal/storage"
)

var (
	ServiceName = "<app>"
	Version     = "v0.0.0"
)

type Config struct {
	LogLevel string `cfg:"log_level" default:"info"`

	Storage storage.Config `cfg:"storage"`
	Server  Server         `cfg:"server"`

	// Embed tell.Config so telemetry is configured from the same file/env.
	Telemetry tell.Config `cfg:"telemetry"`
}

type Server struct {
	Host string `cfg:"host"`
	Port string `cfg:"port" default:"8080"`
}

func Load(ctx context.Context) (*Config, error) {
	var cfg Config
	if err := chu.Load(ctx, ServiceName, &cfg,
		// Prefix every env var with APP_ (e.g. APP_SERVER_PORT).
		chu.WithLoaderOption(loaderenv.New(
			loaderenv.WithPrefix("<APP>_"),
		)),
		chu.WithVersion(Version),
	); err != nil {
		return nil, err
	}

	if err := logi.SetLogLevel(cfg.LogLevel); err != nil {
		return nil, fmt.Errorf("set log level %s: %w", cfg.LogLevel, err)
	}

	// MarshalMap honours log:"-" so secrets stay out of the log.
	slog.Info("loaded configuration", "config", chu.MarshalMap(cfg))

	return &cfg, nil
}
```

`chu` loads sources in order — **Default → File → HTTP → Environment** — each
overriding the previous. Notes:
- **File**: `CONFIG_FILE` env, else `<name>.{toml,yaml,yml,json}` in cwd.
- **Env**: `cfg`/`env` tags; auto-loads `.env` / `.env.local`; add more with
  `CONFIG_ENV_FILE`. Use the `noprefix` tag option for a field to also accept
  the unprefixed var as a fallback (`cfg:"log_level,noprefix"`).
- **Secret backends** (Vault, Consul, GCP Secret/Parameter, AWS Secrets/SSM,
  Azure Key Vault) are off by default — enable each via blank import and
  configure via its env vars.
- **Toggle any loader off** with `CONFIG_SET_<NAME>=false`
  (e.g. `CONFIG_SET_VAULT=false`) so a down backend doesn't block startup.
- `chu.MarshalMap(cfg)` / `chu.MarshalJSON(cfg)` render config for logging.

## HTTP with `ada`

`ada` is a thin net/http framework: handlers are plain
`func(http.ResponseWriter, *http.Request)`, path params via `r.PathValue`.
Mount the standard middleware stack on the root, then carve route groups.

```go
package server

import (
	"context"
	"net/http"

	"github.com/rakunlabs/ada"
	mcors "github.com/rakunlabs/ada/middleware/cors"
	mlog "github.com/rakunlabs/ada/middleware/log"
	mrecover "github.com/rakunlabs/ada/middleware/recover"
	mrequestid "github.com/rakunlabs/ada/middleware/requestid"
	mserver "github.com/rakunlabs/ada/middleware/server"
	mtelemetry "github.com/rakunlabs/ada/middleware/telemetry"

	"github.com/rakunlabs/<app>/internal/config"
	"github.com/rakunlabs/<app>/internal/service"
)

func Start(ctx context.Context, cfg *config.Config, svc *service.Service) error {
	server := ada.New()
	server.Use(
		mrecover.Middleware(),            // panic → 500, keep process alive
		mserver.Middleware(config.ServiceName),
		mcors.Middleware(),
		mrequestid.Middleware(),          // X-Request-Id
		mlog.Middleware(),                // structured access log
		mtelemetry.Middleware(),          // OTEL spans/metrics per request
	)

	server.GET("/healthz", func(w http.ResponseWriter, _ *http.Request) {
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte("OK"))
	})

	api := server.Group("/api/v1")
	api.GET("/hello/{user}", func(w http.ResponseWriter, r *http.Request) {
		_, _ = w.Write([]byte("Hello, " + r.PathValue("user")))
	})

	// Honour ctx cancellation for graceful shutdown if your ada version
	// exposes a context-aware start; otherwise wire server.Shutdown to ctx.
	return server.Start(cfg.Server.Host + ":" + cfg.Server.Port)
}
```

Runtime middleware reload (swap/enable/disable without restart) via
`ada.NewSlot` / `ada.NewPipeline` — see the ada guide when you need it.

## Library quickstarts

### into — lifecycle
`into.Init(run, opts...)` runs `run(ctx context.Context) error`. `ctx` is
cancelled on SIGINT/SIGTERM. Options seen in practice:
`into.WithLogger(<slog.Logger>)`, `into.WithMsgf(format, args...)` (startup
banner). Do all setup/teardown inside `run` with `defer`.

### logi — logging
```go
logi.InitializeLog()                 // call once, early (or via into.WithLogger)
logi.SetLogLevel("debug")            // parse + set global slog level
slog.Info("message", "key", "value") // use std slog everywhere after init

ctx = logi.WithContext(ctx, slog.With("component", "worker"))
logi.Ctx(ctx).Info("scoped log")
```

### tell — telemetry
Embed `tell.Config` in your app config. `collector, err := tell.New(ctx, cfg.Telemetry)`;
`defer collector.Shutdown()`. Empty `OTEL_EXPORTER_OTLP_ENDPOINT` → noop
providers (code stays identical). Create instruments from
`collector.MeterProvider.Meter("")`; start spans with
`otel.Tracer("").Start(ctx, name, trace.WithSpanKind(...))`. Set
`OTEL_RESOURCE_ATTRIBUTES=service.name=...` when running multiple services
locally.

### ok — retryable HTTP client
Retry is ON by default (4 tries, exp backoff + jitter, retries 5xx/429/timeouts).
The response body is always drained+closed after the callback.
```go
client, err := ok.New(
	ok.WithBaseURL("https://api.example.com/v1"),
	ok.WithTimeout(30 * time.Second),
	ok.WithHeaderSet("Authorization", "Bearer "+token),
)

req, _ := http.NewRequestWithContext(ctx, http.MethodGet, "/users", nil)
var users []User
err = client.Do(req, ok.ResponseFuncJSON(&users)) // pass nil to only check 2xx
```
Populate `ok.Config` from `chu` (`cfg:"..."` tags) and convert with
`cfg.ToOption()` / `cfg.New(...)`.

### cache — generic cache
```go
c, err := cache.New[string, int](ctx,
	memory.Store,
	cache.WithStoreConfig(&memory.Config{MaxItems: 100, TTL: 10 * time.Minute}),
)
_ = c.Set(ctx, "k", 1)
v, ok, err := c.Get(ctx, "k")
v, err = c.GetSet(ctx, "k", func() (int, error) { return load() })
```
Redis store: `redis.Store(redisClient)` + `cache.WithStoreConfig(redis.Config{TTL: ...})`.
Interface: `Get`, `Set`, `Delete`, `GetSet`.

### query — URL → expression
Turns `?name=foo,bar&age[lt]=30&_sort=-age&_limit=10&_offset=5&_fields=id,name`
into a `*query.Query`. Operators: `eq ne gt lt gte lte like ilike nlike nilike
in nin is not kv jin njin`; `,`→`in`; `|`=OR, `&`=AND, `()` groups; `_limit
_offset _sort _fields` are reserved. Adapters convert to SQL
(`adaptergoqu.Select`) and `bw` consumes it directly. Validate field/value/
limit with `query.NewValidator(...)` and `query.WithValidator`.

### muz — SQL migrations
Runs numbered SQL migration files from a filesystem or embedded `embed.FS`.
Use `muz.SQLDriver` for SQL databases including PostgreSQL, MySQL, SQLite, and
MSSQL. Set `Dialect` to match the target database and set `LockKey` when more
than one process may run migrations concurrently.
```go
//go:embed migrations
var migrationsFS embed.FS

func migrate(ctx context.Context, db *sql.DB) error {
	m := muz.Migrate{
		Path:      "migrations",
		FS:        migrationsFS,
		Extension: ".sql",
	}

	driver := &muz.SQLDriver{
		DB:      db,
		Dialect: muz.DialectPostgres,
		Table:   "migrations",
		LockKey: "muz:postgres:public:migrations",
		Logger:  slog.Default(),
	}

	return m.Migrate(ctx, driver)
}
```
Migration files are sorted by numeric prefix (`1_schema.sql`,
`2_indexes.sql`) and executed in order. Use `Order` to prioritize directories
and `Skip` to exclude directories/files with glob patterns.

### bw — Badger typed buckets
One struct tag set drives schema + wire name; no codegen.
```go
type User struct {
	ID    string `bw:"id,pk"`
	Name  string `bw:"name,index"`
	Email string `bw:"email,unique"`
	Bio   string `bw:"-"`            // never serialized
}

db, _ := bw.Open("/var/lib/<app>")
defer db.Close()
users, _ := bw.RegisterBucket[User](db, "users")

q, _ := query.Parse("name=Tarık|age[gt]=29&_sort=-age&_limit=10")
got, _ := users.Find(ctx, q)
```
Bump `bw.WithVersion[User](n)` when you change the index/unique surface to
auto-migrate. `bw/cluster` (built on `alan`) adds leader-only writes + local
reads with `NotifySync`.

### alan — peer discovery / locks
QUIC-based discovery with TLS 1.3 (optional pre-shared key for admission).
```go
a, _ := alan.New(alan.Config{DNSAddr: "<app>-headless.ns.svc.cluster.local", Port: 7946, Replicas: 3})
go a.Start(ctx)

// Leader-elected singleton work (cron, indexer, scheduler):
_ = a.LeaderLoop(ctx, "scheduler", 5*time.Second, func(ctx context.Context) error {
	return runCron(ctx)
})
```
Also: `Send`/`Handle`, `SendAndWaitReply`, `Lock`/`TryLock`/`Unlock`,
`SendStream`/`HandleStream` for large payloads. Best-effort coordination, not
Raft — fine for idempotent/recoverable work, not for non-idempotent external
effects.

### tummy — time control in tests
Use `tummy` when code depends on the current time and tests need deterministic
time travel. Enable it in test setup, set the clock, then call `tummy.Now()` in
code paths you want to control.
```go
func TestExpiresAt(t *testing.T) {
	tummy.Enable()
	t.Cleanup(tummy.Disable)

	tummy.SetTime(time.Date(2026, 5, 31, 12, 0, 0, 0, time.UTC))

	got := expiresAt(tummy.Now(), 2*time.Hour)
	if !got.Equal(time.Date(2026, 5, 31, 14, 0, 0, 0, time.UTC)) {
		t.Fatalf("expiresAt() = %s", got)
	}

	tummy.AddDuration(30 * time.Minute)
	_ = tummy.Now()
}
```
Also available: `tummy.AddDate(years, months, days)` for calendar jumps.

## Conventions & gotchas

- **One env prefix per app** (`<APP>_`) via `loaderenv.WithPrefix`.
- **Mask secrets** with `log:"-"`; never log a raw config struct — use
  `chu.MarshalMap`.
- **Embed `tell.Config`** in the app config so telemetry shares the config
  pipeline.
- **`defer` order in `run`** = graceful shutdown order; telemetry last.
- **Module path** is always `github.com/rakunlabs/<app>`; keep packages under
  `internal/` and dependencies pointing downward.
- **Errors** wrap with `%w` and a short `lower-case; %w` prefix.

## Deep-dive references

- into  — https://pkg.go.dev/github.com/rakunlabs/into
- logi  — https://pkg.go.dev/github.com/rakunlabs/logi
- chu   — https://pkg.go.dev/github.com/rakunlabs/chu
- tell  — https://pkg.go.dev/github.com/rakunlabs/tell
- ada   — https://rakunlabs.github.io/ada/
- ok    — https://pkg.go.dev/github.com/rakunlabs/ok
- cache — https://pkg.go.dev/github.com/rakunlabs/cache
- query — https://pkg.go.dev/github.com/rakunlabs/query
- muz   — https://pkg.go.dev/github.com/rakunlabs/muz
- bw    — https://pkg.go.dev/github.com/rakunlabs/bw
- alan  — https://pkg.go.dev/github.com/rakunlabs/alan
- tummy — https://pkg.go.dev/github.com/rakunlabs/tummy

Reference implementation: `github.com/rakunlabs/pika`.
