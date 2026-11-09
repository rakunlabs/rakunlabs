# Config with `chu`

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
