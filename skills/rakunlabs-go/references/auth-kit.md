# ada Auth Kit

Full authentication for an ada app: login page, local users, roles and
permissions, a user admin page and API, an account API, an OAuth 2.1
authorization server for MCP and CLI clients, personal access tokens, and SSO
next to local login.

`middleware/auth` (imported) provides the generic parts. Users, roles and
storage are **application code**: you copy them from the reference app
[`github.com/rakunlabs/ada/_examples/authkit`](https://github.com/rakunlabs/ada/tree/main/_examples/authkit)
into the target app and adapt them. Nothing named `users`, `kit` or `sqlstore`
exists under `middleware/auth`; do not import those paths.

## Files of this guide

Read them in order. Skip the optional ones the user did not ask for.

| Step | File | Result |
| --- | --- | --- |
| 1 | [auth-kit-copy.md](auth-kit-copy.md) | Reference code copied into `internal/auth`, imports rewritten, builds |
| 2 | [auth-kit-store.md](auth-kit-store.md) | Tables in the app's migrations; store on the app's DB (or its own data layer) |
| 3 | [auth-kit-wiring.md](auth-kit-wiring.md) | Config, `kit.New`, `Mount`, guarded routes, frontend hooks |
| 4 | [auth-kit-roles.md](auth-kit-roles.md) | The app's roles and permissions |
| 5 | [auth-kit-mcp.md](auth-kit-mcp.md) | Optional: MCP endpoint, personal tokens, sub-resources, hooks |
| 6 | [auth-kit-sso.md](auth-kit-sso.md) | Optional: OIDC / LDAP / passkey next to local login |
| 7 | [auth-kit-security.md](auth-kit-security.md) | Rules not to break while adapting, production checklist, verification |

## What goes where

| Source | Destination in the app | Notes |
| --- | --- | --- |
| `middleware/auth`, `.../oauthserver`, `.../authz`, `.../identity`, `.../strategy/*`, `.../guard`, `.../sessionstore/*` | import | `go get github.com/rakunlabs/ada/middleware/auth` |
| `_examples/authkit/users/` (+ `users/admin/` assets) | `internal/auth/users/` | Users, roles, permissions, signup, admin/account APIs, admin page |
| `_examples/authkit/store/` | `internal/store/auth/` or merged into the app's store | SQL for users + OAuth (PostgreSQL, SQLite, MySQL) |
| `_examples/authkit/kit/` | `internal/auth/` | `kit.New`, `Mount`, `API`, `Browser`, `RequirePermission` |
| `*_test.go` next to each | same place | Keep them: they check the rules that fail silently |
| `_examples/authkit/authkit.go`, `reports.go`, `index.html` | read only | Shows routes, token API, `/mcp` handler, a minimal frontend |

## Ground rules

- Copy, then adapt. Rename packages and tables to fit the app, but keep the
  behavior and the tests. If a test fails after adapting, the adaptation is
  wrong, not the test.
- Never weaken a rule in [auth-kit-security.md](auth-kit-security.md) to make
  something work.
- Run the example once to see the target behavior:
  `cd _examples && EXAMPLE=authkit go run .` and open `http://localhost:8080/`.
- Finish with `go vet ./...`, `go test -race ./...` and the curl checks at the
  end of the security file.
