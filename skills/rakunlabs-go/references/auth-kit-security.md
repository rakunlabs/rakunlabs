# Auth Kit 7: security rules, production, verification

Back to [auth-kit.md](auth-kit.md).

## Rules not to break while adapting

Each line is a bug that was found and fixed in the reference. Breaking one
does not fail visibly. If your change touches the area, keep the rule and its
test.

| Rule | Where | Why |
| --- | --- | --- |
| App routes go through `authKit.API()` / `Browser()` (which include `Fresh`) | wiring | Without `Fresh` a disabled or deleted user keeps access until the session ends |
| A local identity whose user is gone is refused | `users.current`, `ClaimLocal` | Otherwise a deleted admin is treated like an SSO user and passes |
| Account and admin APIs are `SessionOnly` | `users.MountSelf/MountAdmin` | Otherwise an MCP client's token can create admins or reset passwords |
| State changes require same-origin `application/json` | `users.sameOrigin` | A sibling subdomain can post a `text/plain` form with Lax cookies |
| Admin actions go through `CreateAs/UpdateAs/DeleteAs` | users | Otherwise `users:write` can self-promote to admin or take over an admin account |
| Last enabled admin who can sign in is kept | `keepAnAdmin`, `activeAdmin` | Includes "remove the last admin's password" |
| Last-admin and first-signup checks run under the service mutex | `writeMu` | Two parallel requests can both pass the check |
| Password rehash on login re-reads under the lock | `rehash` | It can otherwise undo a concurrent disable or role change |
| Built-in claims override attributes | `Identity()` | An attribute named `permissions` could grant anything |
| Consent page uses `Session().Require()` + `Fresh` | kit `Authenticate` | A disabled user could still approve clients |
| Store rules (single-use code, CAS rotate, previous-hash lookup) | store | See [auth-kit-store.md](auth-kit-store.md) |
| `/login` → `/login/` keeps the query | `middleware/auth` | Lost `redirect_path` strands MCP users after login |

What `oauthserver` enforces on its own (don't work around it):

- Tokens only work on the resource they were issued for (audience).
- `resource` must be in `Resources` (default: issuer origin) and can't be
  added at the token step.
- A rotated-out refresh token used again after 30 s revokes the whole grant.
- Dynamic registration is rate limited per IP (60/h).
- Consent forms are CSRF-protected; custom consent JSON must be same-origin.

## Production checklist

- [ ] `external_url` is set (issuer and MCP resource don't follow the Host header).
- [ ] `login.cookie.secure: always` when TLS ends at a proxy without `X-Forwarded-Proto`.
- [ ] Bootstrap password, `cipher_key`, session key come from a secret loader; mask with `log:"-"`.
- [ ] Either `users.bootstrap` is set or first-run signup was completed by you (until then the first visitor becomes admin).
- [ ] `users.signup` is false unless self-registration is intended.
- [ ] Persistent session store + `cipher_key`.
- [ ] Tables are in migrations; `DeleteExpired` runs periodically (one replica).
- [ ] JSON/MCP routes use `API()`, pages `Browser()`, MCP routes `authz.RequireScope("mcp")`.
- [ ] Several replicas: last-admin / first-signup also enforced in the store ([auth-kit-roles.md](auth-kit-roles.md)).
- [ ] The copied tests run in CI.

## Verify

```sh
go vet ./... && go test -race ./...
```

Then with the app running (`B=http://localhost:8080`, adjust `base_path`):

```sh
# Discovery
curl -si -X POST $B/mcp | grep -i www-authenticate        # 401 + resource_metadata="…"
curl -s  $B/.well-known/oauth-protected-resource/mcp       # resource, authorization_servers
curl -s  $B/.well-known/oauth-authorization-server         # endpoints, S256

# Unauthenticated
curl -s -o /dev/null -w '%{http_code}\n' $B/api/v1/reports # 401
curl -s -o /dev/null -w '%{http_code}\n' $B/api/account     # 401
curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' $B/api/admin/users/ui/  # 303 → /login?redirect_path=…

# Login and permissions
curl -s -c jar -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"username":"admin","password":"…"}' $B/login/pass/local
curl -s -b jar $B/api/account                               # roles, permissions
```

And by hand, in a browser:

1. First run: signup form appears, the first user is admin, then signup closes.
2. The admin page creates a user, changes roles, refuses to disable the last admin.
3. A viewer gets 403 on the admin page and on write routes.
4. Disabling a user: their next request gets 401.
5. `claude mcp add --transport http app $B/mcp`: you go through login and consent, then the tools work.
