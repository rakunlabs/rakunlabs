# Standard project layout

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
