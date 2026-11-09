# Auth Kit 3: wiring

Back to [auth-kit.md](auth-kit.md). Package names below assume the layout of
[auth-kit-copy.md](auth-kit-copy.md): `authkit` (`internal/auth`, the copied
kit), `users` (`internal/auth/users`), `authstore` (`internal/store/auth`).

## Config

```go
// internal/config/config.go
import authkit "github.com/rakunlabs/<app>/internal/auth"

type Config struct {
	Server Server      `cfg:"server"`
	Auth   authkit.Config `cfg:"auth"`
}
```

Every field has a working default. A typical file:

```yaml
auth:
  external_url: https://tools.example.com/myapp   # pin in production
  base_path: /myapp
  login:
    ui:
      title: My App
  users:
    bootstrap:                 # first admin; or leave empty for first-run signup
      username: admin
      password: change-me      # from a secret loader (vault, env)
    signup: false              # true = anyone may create an account
    roles:                     # see auth-kit-roles.md
      - name: admin
        permissions: ["*"]
      - name: user
    default_roles: [user]
  oauth:
    enabled: true              # MCP / CLI clients, personal tokens
    scopes: [mcp]
  cipher_key: ...              # encrypts persisted sessions
```

Env overrides follow chu: `<APP>_AUTH_USERS_BOOTSTRAP_PASSWORD`.

| Field | Default | Meaning |
| --- | --- | --- |
| `external_url` | derived from request | Public URL incl. proxy prefix. Pins OAuth issuer + MCP resource |
| `base_path` | `/` | Mux prefix of the app |
| `login` | — | ada `auth.Config`: UI, cookie, session TTLs, MFA |
| `users.roles` | admin `*`, user | Roles and permissions |
| `users.default_roles` | `[user]` | Roles of new users |
| `users.admin_role` | `admin` | Must grant `users:write`; the last one is protected |
| `users.bootstrap.username/password` | — | First admin, created when no user exists |
| `users.signup` | `false` | Self-signup on the login page |
| `users.disable_first_user_signup` | `false` | Without bootstrap, the first visitor becomes admin unless this is set |
| `users.min_password_length` | `8` | |
| `local_label` | `Username` | Login form label |
| `disable_local` | `false` | No password login (SSO only) |
| `limiter` | 5 failures | `guard.Config`, lockout |
| `account_path` | `{base}api/account` | `"-"` disables |
| `admin_path` | `{base}api/admin/users` | `"-"` disables API and page |
| `disable_admin_ui` | `false` | Keep API, drop page |
| `oauth.enabled` | `false` | Authorization server |
| `oauth.server` | — | `oauthserver.Config` (hooks, TTLs, `Resources`, `RegistrationsPerHour`) |
| `oauth.resource_path` | `/mcp` | Protected endpoint advertised to clients |
| `oauth.subresources` | `false` | `/mcp/{x}` are separate resources |
| `oauth.scopes` | `[mcp]` | Supported and default scopes |
| `cipher_key` | — | Encrypts sessions in a persistent session store |

## Build

```go
// internal/server/server.go (or wherever the mux is built)
st, err := authstore.New(db, authstore.Config{Dialect: authstore.Postgres, TablePrefix: "auth_"})
if err != nil {
	return fmt.Errorf("auth store; %w", err)
}

sessions, err := file.New(file.Config{SessionKey: cfg.SessionKey, Path: "/var/lib/<app>/sessions"}, sessionstore.Options{})
if err != nil { // github.com/rakunlabs/ada/middleware/auth/sessionstore/file (or /redis)
	return fmt.Errorf("session store; %w", err)
}

authKit, err := authkit.New(ctx, cfg.Auth, authkit.Stores{Users: st, OAuth: st, Sessions: sessions})
if err != nil {
	return fmt.Errorf("auth; %w", err)
}
defer func() { _ = authKit.Close() }()
```

`authkit.Stores{}` keeps everything in memory: fine for a first run and tests.

`New` creates the bootstrap admin when no user exists, registers the local
login (lockout, signup), starts the OAuth server if enabled, points the MCP
metadata at it, and initializes sessions.

## Mount and guard

```go
// /login, /logout, /login/me, /oauth/*, /.well-known/*,
// {base}api/account, {base}api/admin/users (+ /ui/)
authKit.Mount(mux)

base := mux.Group(cfg.Auth.BasePath)

// JSON API: 401 instead of a redirect.
api := base.Group("/api/v1", authKit.API()...)
api.GET("/reports", listReports, authKit.RequirePermission("reports:read"))
api.DELETE("/reports/{id}", deleteReport, authKit.RequirePermission("reports:delete"))
api.GET("/settings", getSettings, authKit.RequireRole("admin"))

// MCP: OAuth and personal tokens are accepted here.
mcp := base.Group("/mcp", append(authKit.API(), authz.RequireScope("mcp"))...)

// Pages: unauthenticated browsers are redirected to the login page.
pages := base.Group("/app", authKit.Browser()...)
```

- `API()` = `session.DisableRedirect()` + `Require()`. Use it for JSON and MCP.
- `Browser()` = `Require()`. Use it for pages.
- `Require()` = `auth.Require()` + `users.Fresh(true)`: every request re-reads
  the user, so disable/delete/role changes apply immediately.
- Never put app routes behind `auth.Require()` alone; always go through the
  kit chains, or `Fresh` is skipped.

In a handler:

```go
id := identity.FromContext(r.Context())
id.Subject                                    // user ID (stable)
id.Name, id.Email, id.Roles
identity.Claim[string](id, users.ClaimUsername)
authKit.Users.Can(id, "reports:delete")       // check in code
oauthserver.Info(id)                          // non-nil: the request came with a token
```

## Mounted routes

| Route | Who | What |
| --- | --- | --- |
| `GET {base}login/` | anyone | Login page; signup on first run or when `signup: true` |
| `GET {base}login/me` | signed in | Current identity |
| `POST {base}logout` | signed in | Sign out |
| `GET/PUT {base}api/account` | session | Own profile `{id, username, name, email, roles, permissions, has_password}`; PUT `{name, email}` |
| `POST {base}api/account/password` | session | `{current, password}` |
| `GET {base}api/admin/users` | `users:read` | List |
| `POST {base}api/admin/users` | `users:write` | `{username, password, name, email, roles, disabled, attributes}` |
| `GET/PATCH/DELETE {base}api/admin/users/{id}` | read / write | PATCH takes any field or `password` |
| `GET {base}api/admin/users/roles` | `users:read` | Roles and permissions |
| `GET {base}api/admin/users/ui/` | `users:read` | Built-in admin page |
| `/.well-known/…`, `{base}oauth/…` | clients | OAuth server, when `oauth.enabled` |

Account and admin routes accept browser sessions only (tokens get 403) and
state-changing calls must be same-origin `application/json`.

## Frontend (SPA)

No auth code of its own is needed:

| Need | Do |
| --- | --- |
| Who is signed in | `GET {base}api/account` |
| 401 | `location.href = "{base}login/?redirect_path=" + encodeURIComponent(location.pathname + location.search + location.hash)` |
| Sign out | `POST {base}logout`, then reload |
| Hide buttons | Check `permissions` with the same rules (`*`, `x:*`) — the server still enforces |
| Change password | `POST {base}api/account/password` |
| Manage users | Link to `{base}api/admin/users/ui/`, or build a screen on the admin API |
| Requests | Send `Content-Type: application/json`; same origin, cookies included |

Theme the login page with `login.ui.theme` (or `login.ui.custom_css_url`).
To serve your own login page, set `login.ui.external_folder: true` and mount
your handler at `{base}login/` (see the ada Login UI docs).

## Behind a reverse proxy

- Set `external_url` to the public URL including the prefix.
- If the proxy strips the prefix, keep `base_path: /` and set
  `oauth.server.trusted_proxies`; the admin page uses relative URLs and
  keeps working.
- If TLS ends at a proxy that sends no `X-Forwarded-Proto`, set
  `login.cookie.secure: always`.
