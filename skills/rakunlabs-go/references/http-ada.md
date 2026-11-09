# HTTP with `ada`

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
(static/SPA) and `handler/swagger`. For a full login + users + roles + MCP
OAuth stack, follow [auth-kit.md](auth-kit.md).

Runtime reload, without restarting:
- Routes: `server.GET(...)` after start and `server.Remove(method, path)`;
  `server.Routes()` lists them. In-flight requests keep their routing table.
- Middleware: `ada.NewSlot(mw)` (`Replace` / `Disable` / `Enable`) or
  `ada.NewPipeline()` (`Set(key, mw)` / `Remove(key)`), mounted with
  `server.Use(slot.Middleware())`.

## Embedding the Svelte UI

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
