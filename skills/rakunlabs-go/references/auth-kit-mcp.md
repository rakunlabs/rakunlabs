# Auth Kit 5: MCP, personal tokens, hooks (optional)

Back to [auth-kit.md](auth-kit.md). Needs `auth.oauth.enabled: true`.

## What clients see

An MCP client (`claude mcp add --transport http app https://…/mcp`, Cursor,
the MCP Inspector) does this on its own:

1. Calls `/mcp`, gets `401` with
   `WWW-Authenticate: Bearer resource_metadata="…/.well-known/oauth-protected-resource/mcp"`.
2. Reads the resource metadata → the authorization server → its
   `/.well-known/oauth-authorization-server` metadata.
3. Registers itself (`POST {base}oauth/register`, RFC 7591).
4. Opens the browser at `{base}oauth/authorize` with PKCE and `resource`.
   The user signs in on the login page and approves on the consent page
   ("don't ask again" is remembered per client).
5. Exchanges the code at `{base}oauth/token`, then calls `/mcp` with the
   bearer token and refreshes it as needed.

The app only has to mount the kit and guard the MCP route:

```go
mcpGuard := append(authKit.API(), authz.RequireScope("mcp"))
mux.HandleWithMethod("", cfg.Auth.BasePath+"/mcp", mcpHandler, mcpGuard...)
// or a group with HandleWildcard for /mcp/...
```

Use the official SDK (`github.com/modelcontextprotocol/go-sdk`) as
`mcpHandler`. Inside it, `identity.FromContext(ctx)` is the user with
current roles; `oauthserver.Info(id)` has the token ID, client ID, scopes,
resource and `Extra`.

Tokens are opaque (`oat_…` access, `ort_…` refresh), stored only as hashes.
There is no JWT / ID token.

## Personal access tokens

Same server, long-lived tokens for scripts; they work on every route behind
`authKit.API()` and stop working when the user is disabled.

```go
api.POST("/tokens", func(c *ada.Context) error {
	var in struct {
		Name string `json:"name"`
		Days int    `json:"days"`
	}
	if err := c.Bind(&in); err != nil {
		return ada.WrapHTTPError(http.StatusBadRequest, err)
	}

	raw, t, err := authKit.OAuth.IssueToken(c.Request.Context(), oauthserver.IssueRequest{
		Name:     in.Name,
		Identity: identity.FromContext(c.Request.Context()),
		Scopes:   []string{"mcp"},
		TTL:      time.Duration(in.Days) * 24 * time.Hour, // 0 = never expires
	})
	if err != nil {
		return err
	}

	return c.SetStatus(http.StatusCreated).SendJSON(map[string]any{"id": t.ID, "token": raw}) // shown once
})

api.GET("/tokens", func(c *ada.Context) error {
	list, err := st.TokensBySubject(c.Request.Context(), identity.FromContext(c.Request.Context()).Subject)
	if err != nil {
		return err
	}
	return c.SendJSON(list) // never includes raw tokens
})

api.DELETE("/tokens/{id}", func(c *ada.Context) error {
	ctx, me := c.Request.Context(), identity.FromContext(c.Request.Context())
	list, err := st.TokensBySubject(ctx, me.Subject)
	if err != nil {
		return err
	}
	for _, t := range list {
		if t.ID == c.Request.PathValue("id") { // only the caller's own tokens
			if t.ClientID != "" {
				_ = st.DeleteConsent(ctx, me.Subject, t.ClientID) // ask again next time
			}
			if err := st.DeleteToken(ctx, t.ID); err != nil {
				return err
			}
			return c.SendNoContent()
		}
	}
	return ada.NewHTTPError(http.StatusNotFound, "no such token")
})
```

The token routes are under `authKit.API()`, so a token could manage tokens.
If that is unwanted, add `users.SessionOnly()` to them.

Registered OAuth clients for admins:

```go
admin := base.Group("/api/v1/admin", append(authKit.API(), users.SessionOnly(), authKit.RequirePermission("users:write"))...)
admin.GET("/oauth/clients", func(c *ada.Context) error {
	list, err := st.ListClients(c.Request.Context())
	if err != nil {
		return err
	}
	return c.SendJSON(list)
})
admin.DELETE("/oauth/clients/{id}", func(c *ada.Context) error {
	return st.DeleteClient(c.Request.Context(), c.Request.PathValue("id"))
})
```

## One MCP endpoint per resource

For `/mcp/{connection}` limit each token to the resource it was approved for:

```yaml
auth:
  oauth:
    enabled: true
    subresources: true
```

```go
cfg.Auth.OAuth.Server.Hooks.Authorize = func(ctx context.Context, req *oauthserver.AuthorizeRequest) error {
	conn := lastSegmentAfter(req.Resource, "/mcp/")
	if conn == "" {
		return nil // whole /mcp
	}
	if !canUse(ctx, req.Identity.Subject, conn) {
		return &oauthserver.Error{Code: "invalid_target", Description: "no access to " + conn}
	}
	req.Extra = map[string]any{"connection": conn}
	return nil
}
// set before authkit.New(ctx, cfg.Auth, ...)
```

The server already refuses a `/mcp/shop` token on `/mcp/admin` or any other
path (audience check). In the handler you can still compare `Extra`:

```go
if info := oauthserver.Info(id); info != nil {
	if c, _ := info.Extra["connection"].(string); c != "" && c != r.PathValue("conn") {
		return ada.NewHTTPError(http.StatusForbidden, "token is for another connection")
	}
}
```

## Hooks

`oauthserver.Config.Hooks` (set under `cfg.Auth.OAuth.Server.Hooks`):

| Hook | Runs | Use |
| --- | --- | --- |
| `Authorize` | Before consent, user known | Narrow scopes, restrict resources, attach `Extra` |
| `Identity` | Code exchange, refresh, every request | Preset by the kit to `users.IdentityHook()`; keep it unless users live elsewhere |
| `TokenTTL` | Issuing | Per-client/user lifetimes; negative refresh TTL = no refresh token |

Other useful `oauthserver.Config` fields: `Resources` (allowed resource
identifiers; default the issuer origin), `RegistrationsPerHour` (default 60
per IP), `DisableRegistration`, `TrustedProxies`, `ConsentURL` (own consent
page, JSON flow).
