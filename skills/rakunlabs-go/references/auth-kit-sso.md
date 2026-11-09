# Auth Kit 6: SSO (optional)

Back to [auth-kit.md](auth-kit.md).

Any `middleware/auth` strategy can sit next to the local login; the login page
shows a button for each. Register it with `authkit.WithStrategy` (the copied
kit's option):

```go
import authoauth2 "github.com/rakunlabs/ada/middleware/auth/strategy/oauth2"

kc, err := authoauth2.NewWithContext(ctx, "keycloak", authoauth2.Config{
	IssuerURL:    "https://kc.example.com/realms/main",
	ClientID:     cfg.SSO.ClientID,
	ClientSecret: cfg.SSO.ClientSecret,
	Scopes:       []string{"openid", "email", "profile"},
}, authoauth2.Options{
	Label:           "Sign in with Keycloak",
	CallbackBaseURL: "https://tools.example.com", // origin only: scheme + host, no path
})
if err != nil {
	return fmt.Errorf("sso; %w", err)
}

authKit, err := authkit.New(ctx, cfg.Auth, stores, authkit.WithStrategy(kc))
```

The callback path (with `base_path`) is filled in by `middleware/auth` when it
mounts the strategy. Register
`https://tools.example.com{base}login/callback/keycloak` as the redirect URI
at the IdP (`{base}` is `base_path`, the last segment is the strategy name).

Other strategies under `middleware/auth/strategy/`: `ldap`, `passkey`,
`apikey`, `bearer`, `magiclink`, `header`, `mtls`, `basic`. See the ada
"Strategies" docs for each one's config.

## How SSO identities behave

- An SSO identity is not a local user. Its subject is the upstream `sub`.
  `users.Fresh` and the OAuth identity hook pass it through unchanged, so SSO
  users can use the app and MCP clients.
- Local identities carry `users.ClaimLocal` (`users.IsLocal(id)`), so a
  deleted local user is refused instead of being mistaken for SSO.
- With any extra strategy registered, the kit sets `users.external_login`:
  an admin without a password still counts as someone who can sign in.
- `disable_local: true` removes the password form (SSO only). Users are still
  managed locally if you map SSO identities to them.

## Giving SSO users local roles

SSO users have only the roles the IdP put in the identity. To manage their
roles in the app, map them to local accounts in a strategy wrapper:

```go
// mapped wraps an SSO strategy so its identities become local users.
type mapped struct {
	strategy.Authenticator // Name, Descriptor, Logout come from the SSO strategy
	users *users.Service
}

func (m mapped) Login(w http.ResponseWriter, r *http.Request) (*identity.Identity, strategy.Outcome, error) {
	id, out, err := m.Authenticator.Login(w, r)
	if err != nil || out != strategy.OutcomeContinue || id == nil {
		return id, out, err // redirect to the IdP, or a failure already written
	}

	if id.Email == "" {
		http.Error(w, "the identity provider sent no verified email", http.StatusForbidden)
		return nil, strategy.OutcomeFailed, nil
	}

	ctx := r.Context()
	u, err := m.users.Store().GetUserByUsername(ctx, id.Email)
	if errors.Is(err, users.ErrNotFound) {
		// First login: create an SSO-only account (empty password).
		u, err = m.users.Create(ctx, users.CreateRequest{Username: id.Email, Name: id.Name, Email: id.Email})
	}
	if err != nil {
		return nil, strategy.OutcomeContinue, err
	}
	if u.Disabled {
		http.Error(w, "account disabled", http.StatusForbidden)
		return nil, strategy.OutcomeFailed, nil
	}

	local := m.users.Identity(u) // local roles, IsLocal = true
	local.Provider = id.Provider // keep the SSO provider for logout

	return local, strategy.OutcomeContinue, nil
}

// SetCallbackBasePath forwards the callback path the middleware assigns.
func (m mapped) SetCallbackBasePath(p string) {
	if b, ok := m.Authenticator.(strategy.CallbackBinder); ok {
		b.SetCallbackBasePath(p)
	}
}
```

Register `mapped{Authenticator: kc, users: …}` instead of `kc`. The users
service is created inside `authkit.New`, so either build the users service
yourself before `New`, or add an option to the copied kit that wraps extra
strategies after `users.New` (the kit is your code now).

Usernames are 2–64 letters, digits and `. _ - @ +`, so most emails fit. If
your IdP may send an email that collides with an existing local account,
decide whether linking is allowed: an attacker who controls that email at the
IdP gets the account.

Link by a verified email only (`authoauth2.Options{EmailVerifyCheck: true}`),
otherwise anyone who can set that email at the IdP takes over the account.
