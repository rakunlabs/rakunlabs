---
name: rakunlabs-go
description: >-
  Use when building, scaffolding, or wiring a Go service/microservice with the
  rakunlabs library stack (github.com/rakunlabs/*): into, logi, chu, gofret,
  tell, ada, ok, cache, query, muz, types, bw, alan, hardloop,
  tummy. Covers the cmd/internal/_ui project layout, the canonical main.go
  run(ctx) wiring, chu config, the ada middleware stack and context handlers,
  embedding a Svelte UI, and per-library quickstarts. Trigger on "rakunlabs
  service", "new Go service with chu/ada/into", "into.Init", "chu.Load",
  "ada.New", "ada.Context", "StartWithContext", "muz.Migrate", "SQLDriver",
  "gofret.To", "hardloop.NewCron", "tummy.Now".
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
keeps the module path `github.com/worldline-go/bucket`). When you
need depth beyond this file, read the library's `README.md` / pkg.go.dev
(links at the bottom). Do not invent APIs. If a README and the code disagree,
the code wins: check the signature on pkg.go.dev.

## When to use

- Creating a new `github.com/rakunlabs/<app>` Go service from scratch.
- Wiring config loading, telemetry, an HTTP server, storage, background jobs,
  or graceful shutdown into an existing rakunlabs service.
- Using any of the libraries in the map below.

## The library map

| Library | Role | Entry point |
| --- | --- | --- |
| `into`     | App lifecycle: run a func, signals, graceful shutdown, health/kill endpoints | `into.Init(run, opts...)` |
| `logi`     | slog initializer (color on TTY, JSON otherwise) | `logi.InitializeLog()` |
| `chu`      | Layered config loader (default→file→http→env + secret backends) | `chu.Load(ctx, name, &cfg, opts...)` |
| `gofret`   | Map↔struct↔struct conversion (the decoder under `chu`) | `gofret.To[T](in)` |
| `tell`     | OpenTelemetry metrics + traces collector | `tell.New(ctx, cfg.Telemetry)` |
| `ada`      | net/http web framework, context handlers, runtime route/middleware reload | `ada.New()` |
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

## Standard project layout

Mirror `github.com/rakunlabs/kafka-console` (small and clear) or
`github.com/rakunlabs/pika` (larger, with storage and clustering):

```
<app>/
  cmd/<app>/main.go        # entrypoint: into.Init(run, ...)
  internal/
    config/config.go       # Config struct + chu.Load (this app's env prefix)
    server/
      server.go            # ada.New(), middleware stack, route groups
      embed.go             # //go:embed all:dist (built UI)
      dist/                # vite outDir; keep a placeholder committed
    service/               # business logic (transport-agnostic)
    store/ or storage/     # data layer (sql + muz, bw, ...)
  _ui/                     # Svelte + TypeScript + Tailwind frontend (pnpm)
  go.mod                   # module github.com/rakunlabs/<app>
  Makefile
  .goreleaser.yaml         # release/build via goreleaser
  AGENTS.md                # repo-specific agent rules (optional)
```

Conventions:
- `module github.com/rakunlabs/<app>`, current Go toolchain (`go 1.27`).
- `internal/` holds everything app-specific; packages depend downward
  (`server` → `service` → `store`), never the reverse.
- Build metadata is injected via ldflags into `cmd/<app>/main.go`:
  `var version, commit, date string`.
- One binary: the UI is built into `internal/server/dist` and embedded.
- ada middlewares, ada handlers and chu external loaders are **separate Go
  modules** (`github.com/rakunlabs/ada/middleware/cors`,
  `github.com/rakunlabs/chu/loader/external/loadervault`, ...). Import only the
  ones you use and let `go mod tidy` add each one.

## The canonical `main.go`

`into.Init` owns the process lifecycle. It runs `run(ctx)`, wires a logger and
a startup banner, and cancels `ctx` on SIGINT/SIGTERM so every `defer` in
`run` unwinds for a graceful shutdown. Keep `main` tiny; put all wiring in
`run`. `into.Init` never returns.

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
	"github.com/rakunlabs/<app>/internal/store"
)

// Injected at build time via -ldflags.
var (
	version = "v0.0.0"
	commit  = "-"
	date    = "-"
)

func main() {
	config.Version = version

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
	defer func() { _ = collector.Shutdown() }()

	st, err := store.New(ctx, &cfg.Store)
	if err != nil {
		return fmt.Errorf("init store; %w", err)
	}
	defer st.Close()

	svc := service.New(st)

	// Blocks until ctx is cancelled, then shuts the HTTP server down.
	if err := server.Start(ctx, cfg, svc); err != nil {
		return fmt.Errorf("start server; %w", err)
	}

	return nil
}
```

Rules:
- `defer` cleanup in reverse dependency order. The signal-cancelled `ctx`
  triggers them. Telemetry shutdown runs last, so spans/metrics flush after
  the server stops.
- Wrap every error with `fmt.Errorf("...; %w", err)`.
- Never call `os.Exit` inside `run`; return an error and let `into` handle it.
- Hand long-lived background workers the `run` ctx, not a request ctx.

### into exit codes, health and kill endpoints (v0.6+)

| Situation | Exit code |
| --- | --- |
| `run` returned `nil` | `0` |
| `run` returned an error | `1` (or `into.WithErrExitCode`) |
| Shutdown exceeded the wait timeout | `1` |
| SIGTERM / SIGINT, shutdown completed | `143` / `130` (`128+signum`) |

With systemd add `SuccessExitStatus=143` or the unit ends up `failed` after
every `systemctl stop`.

Optional management server (default `0.0.0.0:18080`):

```go
into.Init(run,
	into.WithHealthCheck(),        // GET /healthz; `<bin> --health` probes it (exec probe)
	into.WithKill(),               // POST /kill; denies everything until a check is set
	into.WithServer(into.WithServerAddress("127.0.0.1:18080")),
)

// Inside run, never after into.Init:
into.SetHealthCheck(func(ctx context.Context) error { return db.PingContext(ctx) })
into.SetKillHeaderCheck(func(h http.Header) bool { return h.Get("X-Kill-Secret") == secret })
```

Do not assign `into.HealthCheck = fn` / `into.KillHeaderCheck = fn`; those
variables were replaced by the setters in v0.6.0.

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

	mcors "github.com/rakunlabs/ada/middleware/cors"
	"github.com/rakunlabs/chu"
	"github.com/rakunlabs/chu/loader/loaderenv"
	"github.com/rakunlabs/logi"
	"github.com/rakunlabs/tell"

	// Optional secret backends: blank-import only the ones you need.
	_ "github.com/rakunlabs/chu/loader/external/loadervault"
	// _ "github.com/rakunlabs/chu/loader/external/loaderconsul"
	// _ "github.com/rakunlabs/chu/loader/external/loadergcpsecret"
	// _ "github.com/rakunlabs/chu/loader/external/loadergcpparameter"
	// _ "github.com/rakunlabs/chu/loader/external/loaderawssecrets"
	// _ "github.com/rakunlabs/chu/loader/external/loaderawsssm"
	// _ "github.com/rakunlabs/chu/loader/external/loaderazurekeyvault"

	"github.com/rakunlabs/<app>/internal/store"
)

var (
	ServiceName = "<app>"
	Version     = "v0.0.0"
)

type Config struct {
	LogLevel string `cfg:"log_level,noprefix" default:"info"`

	Server Server       `cfg:"server"`
	Store  store.Config `cfg:"store"`

	// Embed tell.Config so telemetry is configured from the same file/env.
	Telemetry tell.Config `cfg:"telemetry"`
}

type Server struct {
	Host     string     `cfg:"host"`
	Port     string     `cfg:"port" default:"8080"`
	BasePath string     `cfg:"base_path"`
	CORS     mcors.Cors `cfg:"cors"`
}

func Load(ctx context.Context) (*Config, error) {
	var cfg Config
	if err := chu.Load(ctx, ServiceName, &cfg,
		// Prefix every env var with <APP>_ (e.g. <APP>_SERVER_PORT).
		chu.WithLoaderOption(loaderenv.New(
			loaderenv.WithPrefix("<APP>_"),
		)),
		chu.WithVersion(Version),
	); err != nil {
		return nil, fmt.Errorf("load config; %w", err)
	}

	if err := logi.SetLogLevel(cfg.LogLevel); err != nil {
		return nil, fmt.Errorf("set log level %s; %w", cfg.LogLevel, err)
	}

	// MarshalMap honours log:"-" so secrets stay out of the log.
	slog.Info("loaded configuration", "config", chu.MarshalMap(cfg))

	return &cfg, nil
}
```

`chu` loads sources in order (**Default → File → HTTP → Environment**), each
overriding the previous. Notes:
- **File**: `CONFIG_FILE` env, else `<name>.{toml,yaml,yml,json}` in cwd.
- **HTTP**: `CONFIG_HTTP_ADDR` (+ `CONFIG_HTTP_SUFFIX`, `CONFIG_HTTP_QUERY`);
  sends `?version=<chu.WithVersion>` when no query is set. A pika server is
  the usual target.
- **Env**: `cfg`/`env` tags; auto-loads `.env` / `.env.local`; add more with
  `CONFIG_ENV_FILE`. The `noprefix` tag option also accepts the unprefixed var
  as a fallback (`cfg:"log_level,noprefix"` reads `<APP>_LOG_LEVEL`, then
  `LOG_LEVEL`).
- **Secret backends** (Vault, Consul, GCP Secret/Parameter, AWS Secrets/SSM,
  Azure Key Vault) are off by default. Enable each with a blank import and
  configure it with its env vars, or pass `chu.WithLoaderOption(loaderX.New(...))`.
- **Toggle any loader off** at runtime with `CONFIG_SET_<NAME>=false`
  (e.g. `CONFIG_SET_VAULT=false`) so a down backend doesn't block startup, or
  in code with `chu.WithDisableLoader(loader.NameVault)`.
- `chu.MarshalMap(cfg)` / `chu.MarshalJSON(cfg)` render config for logging.
  They take only the value. Some README snippets show a `ctx` argument, which
  does not compile.
- Decoding goes through `gofret`, so `"8080"` reaches an `int` field and
  `max_retry` / `max-retry` / `MaxRetry` all match. Teach extra types with
  `chu.WithCodecOptions(gofret.Between(netip.ParseAddr))`.

## HTTP with `ada`

`ada` is a thin net/http framework. A route accepts either a plain
`func(http.ResponseWriter, *http.Request)` or a context handler
`func(c *ada.Context) error`. Path params come from `r.PathValue` /
`c.Request.PathValue`. Mount the standard middleware stack on the root, carve
route groups, and start with `StartWithContext` so cancelling `ctx` shuts the
server down.

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
		mrecover.Middleware(),                                // panic → 500, keep process alive
		mserver.Middleware(config.ServiceName+"/"+config.Version),
		mcors.Middleware(mcors.WithConfig(cfg.Server.CORS)),
		mrequestid.Middleware(),                              // X-Request-Id
		mlog.Middleware(),                                    // structured access log
		mtelemetry.Middleware(),                              // OTEL spans/metrics per request
	)

	server.GET("/healthz", func(w http.ResponseWriter, _ *http.Request) {
		_, _ = w.Write([]byte("OK"))
	})

	root := server.Group(cfg.Server.BasePath)
	api := root.Group("/api/v1")

	api.GET("/users/{id}", func(c *ada.Context) error {
		u, err := svc.User(c.Request.Context(), c.Request.PathValue("id"))
		if err != nil {
			return err // 500 unless the error carries a status
		}
		return c.SendJSON(u)
	})

	api.POST("/users", func(c *ada.Context) error {
		var in service.User
		if err := c.Bind(&in); err != nil {
			return ada.WrapHTTPError(http.StatusBadRequest, err)
		}
		if in.Name == "" {
			return ada.NewHTTPError(http.StatusBadRequest, "name is required")
		}
		return c.SetStatus(http.StatusCreated).SendJSON(in)
	})

	if err := mountUI(root, cfg.Server.BasePath); err != nil {
		return err
	}

	return server.StartWithContext(ctx, cfg.Server.Host+":"+cfg.Server.Port)
}
```

Context handler notes:
- Return errors; status comes from `ada.NewHTTPError(code, msg)` /
  `ada.WrapHTTPError(code, err)`. A request body over a size limit becomes 413.
- Responses: `SendJSON`, `SendString`, `SendNoContent`, `SendBlob`,
  `SendFile`, `SendZip`; `SetStatus(code)` and `SetHeader(k, v)` chain.
- `c.Bind(&obj)` binds JSON body/query/header/path; body size is unlimited by
  default. Cap it with `middleware/bodylimit` or `bind.WithBodyLimit(n)`.
- `*ada.Context` is pooled and only valid until the handler returns. Copy what
  a goroutine needs; never store the context.

Route-level middleware goes as trailing args:
`api.DELETE("/users/{id}", h, requireAdmin)`. Groups take middleware too:
`server.Group("/api", authMW)`.

Other middlewares (each its own module under `ada/middleware/`): `bodylimit`,
`timeout`, `encoding` (compression), `ratelimit`, `csrf`, `forwardauth`, and
`auth` (sessions, OAuth/OIDC, password, MFA). Handlers: `handler/folder`
(static/SPA) and `handler/swagger`.

Runtime reload, without restarting:
- Routes: `server.GET(...)` after start and `server.Remove(method, path)`;
  `server.Routes()` lists them. In-flight requests keep their routing table.
- Middleware: `ada.NewSlot(mw)` (`Replace` / `Disable` / `Enable`) or
  `ada.NewPipeline()` (`Set(key, mw)` / `Remove(key)`), mounted with
  `server.Use(slot.Middleware())`.

### Embedding the Svelte UI

The frontend lives in `_ui/` (Svelte + TypeScript + Tailwind, built with
pnpm/vite) with `outDir` pointing at `internal/server/dist`. Commit a
placeholder `dist/` so `go build` works without a frontend build.

```go
// embed.go
//go:embed all:dist
var distFS embed.FS

// mountUI serves the SPA under base; unknown API paths must not fall through.
func mountUI(root *ada.Mux, base string) error {
	ui, err := fs.Sub(distFS, "dist")
	if err != nil {
		return err
	}

	f, err := folder.New(&folder.Config{ // github.com/rakunlabs/ada/handler/folder
		Index:      true,
		SPA:        true,
		PrefixPath: base,
		ETag:       folder.ETagStrong, // embed.FS has no modtime
		CacheRegex: []*folder.RegexCacheStore{
			{Regex: `-[\w-]{8}\.(js|css|woff2?)$`, CacheControl: "public, max-age=31536000, immutable"},
			{Regex: `.*`, CacheControl: "no-cache"},
		},
	})
	if err != nil {
		return fmt.Errorf("ui handler; %w", err)
	}
	f.SetFs(http.FS(ui))

	root.HandleWildcard("/", f)

	return nil
}
```

Register `api.HandleFuncWildcard("/", notFound)` on the API group so a typo in
an API path answers a JSON 404 instead of `index.html`.

## Library quickstarts

### logi: logging
```go
logi.InitializeLog()                 // call once, early (or via into.WithLogger)
logi.SetLogLevel("debug")            // parse + set global slog level
slog.Info("message", "key", "value") // use std slog everywhere after init

ctx = logi.WithContext(ctx, slog.With("component", "worker"))
logi.Ctx(ctx).Info("scoped log")
```

### tell: telemetry
Embed `tell.Config` in your app config. `collector, err := tell.New(ctx, cfg.Telemetry)`;
`defer func() { _ = collector.Shutdown() }()`. An empty
`OTEL_EXPORTER_OTLP_ENDPOINT` gives noop providers, so the code stays the same.
Create instruments from `collector.MeterProvider.Meter("")`; start spans with
`otel.Tracer("").Start(ctx, name, trace.WithSpanKind(...))`. Set
`OTEL_RESOURCE_ATTRIBUTES=service.name=...` when running multiple services
locally.

### gofret: map ↔ struct conversion
```go
cfg, err := gofret.To[Config](data)          // map    -> struct
m,   err := gofret.To[map[string]any](cfg)   // struct -> map
err       = gofret.ToInto(src, &dst)         // into an existing value

c := gofret.New(gofret.Between(netip.ParseAddr), gofret.WithStrictKeys())
```
Default tag is `cfg`; options `-`, `omitempty`, `inline`, `remain`, `string`.
Defaults are lenient (weak scalar types, loose key matching); opt into
`WithStrictTypes()` / `WithStrictKeys()` / `WithErrorUnused()` when needed.

### ok: retryable HTTP client
Retry is ON by default (4 tries, exp backoff + jitter, retries 5xx/429/timeouts).
The response body is always drained and closed after the callback.
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
Populate `ok.Config` from `chu` (`cfg:"base_url"`, `cfg:"timeout"`,
`cfg:"retry_max"`, `cfg:"tls"`, ...) and build with `cfg.New(opts...)` or
`ok.New(cfg.ToOption(), ...)`. Per-attempt timeout: `ok.WithRetryTimeout`.

### cache: generic cache
```go
c, err := cache.New[string, int](ctx,
	memory.Store,
	cache.WithStoreConfig(&memory.Config{MaxItems: 100, TTL: 10 * time.Minute}),
)
_ = c.Set(ctx, "k", 1)
v, found, err := c.Get(ctx, "k")
v, err = c.GetSet(ctx, "k", func() (int, error) { return load() })
v, found, err = c.Take(ctx, "otp:"+id) // get + delete atomically (memory, redis)
```
Redis: `rc, err := redis.NewClient(ctx, redis.ConnConfig{Address: []string{"localhost:6379"}})`
(standalone or cluster, detected), `defer rc.Close()`, then
`cache.New[K,V](ctx, redis.Store(rc), cache.WithStoreConfig(redis.Config{TTL: ..., KeyPrefix: "app_"}))`.
The caller owns and closes the client.

### query: URL → expression
Turns `?name=foo,bar&age[lt]=30&_sort=-age&_limit=10&_offset=5&_fields=id,name`
into a `*query.Query`. Operators: `eq ne gt lt gte lte like ilike nlike nilike
in nin is not kv jin njin`; `,`→`in`; `|`=OR, `&`=AND, `()` groups; `_limit
_offset _sort _fields` are reserved. Adapters convert to SQL
(`adaptergoqu.Select(q, dataset)`) and `bw` consumes it directly. Validate
field/value/limit with `query.NewValidator(...)` and
`query.ParseWithValidator(raw, v)`.

### muz: SQL migrations
Runs numbered SQL files from a directory or an embedded `embed.FS`. Use
`muz.SQLDriver` with the matching `Dialect`. Set `LockKey` when more than one
replica may migrate at once (PostgreSQL, MySQL, MSSQL; SQLite returns an error
if `LockKey` is set).
```go
//go:embed migrations
var migrationsFS embed.FS

func migrate(ctx context.Context, db *sql.DB) error {
	m := muz.Migrate{
		Path:      "migrations",
		FS:        migrationsFS,
		Extension: ".sql",
	}

	return m.Migrate(ctx, &muz.SQLDriver{
		DB:      db,
		Dialect: muz.DialectPostgres,
		Table:   "migrations",
		LockKey: "muz:postgres:public:migrations",
		Logger:  slog.Default(),
	})
}
```
Files are sorted by numeric prefix (`1_schema.sql`, `2_indexes.sql`). `Order`
prioritizes directories; `Skip` excludes paths with globs (`/test/**`).

### types: nullable DB/JSON columns
```go
type Train struct {
	Details types.Map[any]          `db:"details"` // map[string]T, NULL stays nil (generic since v0.1)
	Raw     types.RawJSON           `db:"raw"`     // like json.RawMessage + Scan/Value
	Tags    types.Slice[string]     `db:"tags"`
	Meta    types.JSON[Meta]        `db:"meta"`    // any JSON value in one column
	Price   types.Null[int64]       `db:"price"`   // sql.Null[T] that tells explicit null from missing
	At      types.Null[types.Time]  `db:"at"`
}
```
Use `decimal.Decimal` for money math; read numbers as `json.Number` or
`string` and convert.

### bw: Badger typed buckets
One struct tag set drives schema and wire name; no codegen.
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
Also `Insert`, `Update`, `Get`, `Delete`, `Walk`, `Count`, `Exists`,
`FindAndUpdate` and `*Tx` variants. Bump `bw.WithVersion[User](n)` when you
change the index/unique surface to auto-migrate. `bw/cluster` (built on
`alan`) adds leader-only writes and local reads.

### alan: peer discovery / locks
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
Raft. Fine for idempotent/recoverable work, not for non-idempotent external
effects.

### hardloop: cron jobs and time windows
```go
// Simple cron jobs.
job, err := hardloop.NewCron(hardloop.Cron{
	Name:  "cleanup",
	Func:  cleanup, // func(ctx context.Context) error
	Specs: []string{"CRON_TZ=Europe/Istanbul 0 3 * * *"},
})
job.Start(ctx)              // background
defer func() { <-job.Stop().Done() }() // Stop does not wait; Done() does

// Keep a function running only inside windows (start specs → stop specs).
loop, err := hardloop.NewLoop(
	[]string{"0 7 * * 1-5"},  // start weekdays 07:00
	[]string{"0 17 * * 1-5"}, // stop weekdays 17:00
	worker,                   // must honour ctx cancellation
)
loop.SetRestartDelay(10 * time.Second)
err = loop.Run(ctx) // blocks; return hardloop.ErrCloseLoop from worker to end it
```
Day-of-month and day-of-week combine with **AND** (crontab.guru uses OR).
Combine with `alan.LeaderLoop` when only one replica should run the job.

### tummy: time control in tests
Use `tummy.Now()` in code paths whose time tests need to control. It returns
real time until `Enable()` is called.
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
	tummy.AddDate(0, 0, 1) // calendar jump, DST-aware
}
```
Also `Pause()` / `Resume()`. The clock is process-wide: enable it before
starting goroutines that call `Now()`, and do not run tummy tests with
`t.Parallel()`.

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
auth), `github.com/rakunlabs/dataport` (SQL store, muz, hardloop jobs).
