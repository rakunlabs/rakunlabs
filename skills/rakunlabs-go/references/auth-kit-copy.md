# Auth Kit 1: copy the reference code

Back to [auth-kit.md](auth-kit.md).

## Get the source

Use a local checkout if there is one (`../ada`), otherwise:

```sh
git clone --depth 1 https://github.com/rakunlabs/ada /tmp/ada
SRC=/tmp/ada/_examples/authkit
```

## Copy

```sh
APP=github.com/rakunlabs/<app>

mkdir -p internal/auth internal/store/auth
cp -r $SRC/users      internal/auth/users     # includes users/admin/ (embedded page)
cp    $SRC/kit/*.go   internal/auth/
cp    $SRC/store/*.go internal/store/auth/

# Package names: kit -> authkit, store -> authstore. The imports get the old
# names as aliases, so no code body has to change.
sed -i 's/^package kit$/package authkit/; s/^package kit_test$/package authkit_test/' internal/auth/*.go
sed -i 's/^package store$/package authstore/; s/^package store_test$/package authstore_test/' internal/store/auth/*.go

grep -rl 'ada/_examples/authkit' internal | xargs sed -i -E \
  -e "s#^(\s*)\"github.com/rakunlabs/ada/_examples/authkit/users\"#\1\"$APP/internal/auth/users\"#" \
  -e "s#^(\s*)\"github.com/rakunlabs/ada/_examples/authkit/store\"#\1store \"$APP/internal/store/auth\"#" \
  -e "s#^(\s*)\"github.com/rakunlabs/ada/_examples/authkit/kit\"#\1kit \"$APP/internal/auth\"#"
```

Do not name the wiring package `auth`: it imports
`github.com/rakunlabs/ada/middleware/auth`, which is `auth` too.

Then:

```sh
go get github.com/rakunlabs/ada/middleware/auth
go get modernc.org/sqlite   # the store tests use SQLite
go mod tidy
go vet ./... && go test -race ./internal/...
```

All tests must pass before you change anything. That is the baseline.

In app code import them as:

```go
import (
	authkit "github.com/rakunlabs/<app>/internal/auth"
	"github.com/rakunlabs/<app>/internal/auth/users"
	authstore "github.com/rakunlabs/<app>/internal/store/auth"
)
```

## Choosing names

- If the app already has `internal/auth` for something else, use
  `internal/authkit` as the directory. Keep one package for the wiring and
  one for users.
- If the app already has a store package with a `*sql.DB`, the auth store can
  become methods on it instead of a separate package; see
  [auth-kit-store.md](auth-kit-store.md).
- `users/admin/` holds `index.html`, `admin.js`, `admin.css`, embedded by
  `users/ui.go`. Keep it or drop the admin page (`DisableAdminUI`) when the
  app builds its own screen on the admin API.

## What you can delete

| Not needed | Delete |
| --- | --- |
| No MCP / OAuth clients | Keep the store's OAuth half anyway if you want personal tokens; otherwise drop `oauth*` tables and methods and leave `OAuth.Enabled` false |
| Only PostgreSQL | The `MySQL`/`SQLite` branches of the store (keep SQLite if tests use it) |
| Own admin UI | `users/ui.go`, `users/admin/`, and the `MountAdminUI` call in `Mount` |
