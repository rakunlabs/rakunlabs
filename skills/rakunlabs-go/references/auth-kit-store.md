# Auth Kit 2: storage

Back to [auth-kit.md](auth-kit.md).

The kit needs two stores:

- `users.Store`: the copied users package defines it
- `oauthserver.Store` (+ optional `ClientFinder`, `ConsentStore`): imported
  from `middleware/auth/oauthserver`

The copied `authstore` package implements both on a `*sql.DB` for PostgreSQL,
SQLite and MySQL. Use it as is, or move its methods onto the app's own store.
In both cases keep its tests running against your implementation.

## Tables

Put the schema in the app's migrations (muz). Do not call `CreateTables` at
start in an app that has migrations. `authstore.Schema(dialect, prefix)` prints
the exact statements for MySQL/SQLite. PostgreSQL with prefix `auth_`:

```sql
CREATE TABLE IF NOT EXISTS auth_users (
	id TEXT PRIMARY KEY,
	username TEXT NOT NULL,
	username_key TEXT NOT NULL UNIQUE,      -- lower(username): case-insensitive login
	name TEXT NOT NULL,
	email TEXT NOT NULL,
	password_hash TEXT NOT NULL,            -- '' = SSO-only account
	roles TEXT NOT NULL,                    -- JSON array
	disabled BOOLEAN NOT NULL,
	attributes TEXT NOT NULL,               -- JSON object
	created_at BIGINT NOT NULL,             -- unix millis
	updated_at BIGINT NOT NULL
);

CREATE TABLE IF NOT EXISTS auth_clients (
	id TEXT PRIMARY KEY,
	name TEXT NOT NULL,
	redirect_uris TEXT NOT NULL,            -- JSON array
	grant_types TEXT NOT NULL,              -- JSON array
	auth_method TEXT NOT NULL,
	secret_hash TEXT NOT NULL,
	created_at BIGINT NOT NULL
);

CREATE TABLE IF NOT EXISTS auth_codes (
	hash TEXT PRIMARY KEY,
	client_id TEXT NOT NULL,
	identity TEXT NOT NULL,                 -- JSON
	redirect_uri TEXT NOT NULL,
	code_challenge TEXT NOT NULL,
	scopes TEXT NOT NULL,                   -- space separated
	resource TEXT NOT NULL,
	extra TEXT NOT NULL,                    -- JSON
	expires_at BIGINT NOT NULL
);

CREATE TABLE IF NOT EXISTS auth_tokens (
	id TEXT PRIMARY KEY,
	client_id TEXT NOT NULL,                -- '' for personal tokens
	name TEXT NOT NULL,
	subject TEXT NOT NULL,
	identity TEXT NOT NULL,                 -- JSON
	scopes TEXT NOT NULL,
	granted_scopes TEXT NOT NULL,
	resource TEXT NOT NULL,
	extra TEXT NOT NULL,
	access_hash TEXT NOT NULL UNIQUE,
	access_expires_at BIGINT NOT NULL,      -- 0 = never
	refresh_hash TEXT NULL UNIQUE,
	previous_refresh_hash TEXT NULL UNIQUE,
	refresh_expires_at BIGINT NOT NULL,
	rotated_at BIGINT NOT NULL,
	created_at BIGINT NOT NULL
);

CREATE TABLE IF NOT EXISTS auth_consents (
	subject TEXT NOT NULL,
	client_id TEXT NOT NULL,
	scopes TEXT NOT NULL,
	resource TEXT NOT NULL,
	extra_hash TEXT NOT NULL,
	created_at BIGINT NOT NULL,
	PRIMARY KEY (subject, client_id)
);

CREATE INDEX IF NOT EXISTS auth_clients_name ON auth_clients (name);
CREATE INDEX IF NOT EXISTS auth_tokens_subject ON auth_tokens (subject);
CREATE INDEX IF NOT EXISTS auth_tokens_client ON auth_tokens (client_id);
```

MySQL: `VARCHAR(191)` for keys and indexed columns, indexes inside
`CREATE TABLE` (MySQL has no `CREATE INDEX IF NOT EXISTS`).

Only hashes reach the database: codes, access/refresh tokens and client
secrets are stored as SHA-256, never in clear.

## Contract: rules that fail silently

If you rewrite the store (another library, `bw`, an ORM), each rule below
keeps a security property. Breaking one still lets the app run normally, and
the hole stays open. The copied tests check all of them: point them at your
implementation.

### oauthserver.Store

| Method | Must |
| --- | --- |
| any lookup | Return `oauthserver.ErrNotFound` for a missing record. Any other error means an outage (500) |
| `TakeCode` | Read and delete in one transaction. If the `DELETE` affected 0 rows, return `ErrNotFound`. Otherwise two parallel exchanges of one code both get tokens |
| `RotateToken(old, t)` | `UPDATE … WHERE id = ? AND refresh_hash = ?`. If 0 rows changed, return `oauthserver.ErrConflict`. Otherwise parallel refreshes mint several token chains |
| `TokenByRefresh(h)` | Match `refresh_hash = h OR previous_refresh_hash = h`. Otherwise replayed (stolen) refresh tokens are not detected |
| `SaveToken`/`RotateToken` | Persist every `Token` field, including `GrantedScopes`, `PreviousRefreshHash`, `RotatedAt` |
| `DeleteToken` | Removing the row must make its access and refresh hashes unknown |
| `ClientsByName` (optional) | Lets the same MCP client reuse its registration instead of creating one per sign-in |
| `SaveConsent`/`GetConsent` (optional) | Upsert on `(subject, client_id)`; store `ExtraHash` |

### users.Store

| Method | Must |
| --- | --- |
| `GetUserByUsername` | Case-insensitive (`username_key = lower(?)`) |
| `CreateUser`/`UpdateUser` | Return `users.ErrExists` on a taken username (unique violation) |
| lookups | `users.ErrNotFound` for missing users |
| `UpdateUser` | Replace every field (roles, disabled, password hash, attributes) |

### Encoding

- `Identity` and `Extra` are JSON. After decoding, `[]string` claims come back
  as `[]any`. Convert `users.ClaimPermissions` back to `[]string` (the copied
  store does), and any claim your code type-asserts.
- Times as Unix millis keep one schema for every dialect. Zero time = `0`.

### SQLite

SQLite answers a second writer with `SQLITE_BUSY` instead of waiting.
Serialize writes in the store (the copied store has a mutex) and open the DB
with `?_pragma=busy_timeout(5000)&_pragma=journal_mode(WAL)`.

## Extra queries the app needs

The copied store also has these. They are not part of the interfaces, so
keep them if you rewrite:

| Method | Used by |
| --- | --- |
| `TokensBySubject(subject)` | "My tokens" page; revoking one token |
| `DeleteConsent(subject, clientID)` | Revoking a client's token so it asks again |
| `ListClients`, `DeleteClient(id)` | Admin page for registered OAuth clients |
| `DeleteSubject(subject)` | Remove all OAuth data of a deleted user |
| `DeleteExpired(now)` | Periodic cleanup |

Cleanup job (one replica only with `alan.LeaderLoop`):

```go
job, _ := hardloop.NewCron(hardloop.Cron{
	Name:  "auth-cleanup",
	Specs: []string{"*/15 * * * *"},
	Func:  func(ctx context.Context) error { return st.DeleteExpired(ctx, time.Now()) },
})
job.Start(ctx)
```

## Testing against real databases

The copied store tests run on SQLite. To run them against PostgreSQL and
MySQL as well, add backends in `backends()` (`store_test.go`) with the
`pgx/v5/stdlib` and `go-sql-driver/mysql` drivers and a DSN from env. Each
backend gets fresh tables under a unique prefix. Run `race_test.go`
(concurrent `TakeCode`/`RotateToken`) on every dialect you ship.
