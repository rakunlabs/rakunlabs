# The canonical `main.go`

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

## into exit codes, health and kill endpoints (v0.6+)

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
