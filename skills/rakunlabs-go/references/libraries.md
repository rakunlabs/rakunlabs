# Library quickstarts

## logi: logging
```go
logi.InitializeLog()                 // call once, early (or via into.WithLogger)
logi.SetLogLevel("debug")            // parse + set global slog level
slog.Info("message", "key", "value") // use std slog everywhere after init

ctx = logi.WithContext(ctx, slog.With("component", "worker"))
logi.Ctx(ctx).Info("scoped log")
```

## tell: telemetry
Embed `tell.Config` in your app config. `collector, err := tell.New(ctx, cfg.Telemetry)`;
`defer func() { _ = collector.Shutdown() }()`. An empty
`OTEL_EXPORTER_OTLP_ENDPOINT` gives noop providers, so the code stays the same.
Create instruments from `collector.MeterProvider.Meter("")`; start spans with
`otel.Tracer("").Start(ctx, name, trace.WithSpanKind(...))`. Set
`OTEL_RESOURCE_ATTRIBUTES=service.name=...` when running multiple services
locally.

## gofret: map ↔ struct conversion
```go
cfg, err := gofret.To[Config](data)          // map    -> struct
m,   err := gofret.To[map[string]any](cfg)   // struct -> map
err       = gofret.ToInto(src, &dst)         // into an existing value

c := gofret.New(gofret.Between(netip.ParseAddr), gofret.WithStrictKeys())
```
Default tag is `cfg`; options `-`, `omitempty`, `inline`, `remain`, `string`.
Defaults are lenient (weak scalar types, loose key matching); opt into
`WithStrictTypes()` / `WithStrictKeys()` / `WithErrorUnused()` when needed.

## ok: retryable HTTP client
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

## cache: generic cache
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

## query: URL → expression
Turns `?name=foo,bar&age[lt]=30&_sort=-age&_limit=10&_offset=5&_fields=id,name`
into a `*query.Query`. Operators: `eq ne gt lt gte lte like ilike nlike nilike
in nin is not kv jin njin`; `,`→`in`; `|`=OR, `&`=AND, `()` groups; `_limit
_offset _sort _fields` are reserved. Adapters convert to SQL
(`adaptergoqu.Select(q, dataset)`) and `bw` consumes it directly. Validate
field/value/limit with `query.NewValidator(...)` and
`query.ParseWithValidator(raw, v)`.

## muz: SQL migrations
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

## types: nullable DB/JSON columns
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

## bw: Badger typed buckets
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

## alan: peer discovery / locks
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

## hardloop: cron jobs and time windows
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

## tummy: time control in tests
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
