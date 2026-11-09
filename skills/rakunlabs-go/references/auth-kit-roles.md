# Auth Kit 4: roles and permissions

Back to [auth-kit.md](auth-kit.md).

## Model

A role is a named list of permissions, defined in config. Users hold roles.

| Pattern | Grants |
| --- | --- |
| `*` | everything |
| `reports:read` | exactly that |
| `reports:*` | every permission starting with `reports:` |

Design the app's permissions as `<resource>:<action>`:
`reports:read`, `reports:write`, `reports:delete`, `settings:write`.

```yaml
auth:
  users:
    roles:
      - name: admin
        description: Everything, including user management
        permissions: ["*"]
      - name: analyst
        description: Reads and writes reports
        permissions: ["reports:*"]
      - name: viewer
        description: Reads reports
        permissions: ["reports:read"]
    default_roles: [viewer]
```

Built-in permissions: the admin API checks `users:read` and `users:write`.
The `admin_role` (default `admin`) must grant `users:write`.

## Checking

From simplest to most flexible:

```go
// 1. Per route
api.GET("/reports", h, authKit.RequirePermission("reports:read"))

// 2. Combinations (authz)
need := authz.Any{authKit.Users.Permission("reports:write"), authz.Role{"owner"}}
api.PUT("/reports/{id}", h, authz.Require(need))

// 3. In a handler, for rules that depend on the record
if !authKit.Users.Can(id, "reports:delete") && report.Owner != id.Subject {
	return ada.NewHTTPError(http.StatusForbidden, "not allowed")
}

// 4. A path rule table from config: authz.Rules (ada "Identity & Access" docs)
```

`Can` reads the identity's roles, which `Fresh` refreshed for this request.
Permissions are also on the identity as `users.ClaimPermissions` and in
`GET {base}api/account`, so the frontend can hide what the user can't do.

## Per-user data

Team, tenant, avatar and similar go in `User.Attributes` (JSON). Set them via
the admin API (`"attributes": {...}`) or `Users.Update`. They appear in
`Identity.Claims`. Built-in claims (`username`, `permissions`,
`ada_local_user`) always win over attributes with the same name.

## Rules already enforced (keep them)

- An actor may only give roles whose permissions they hold
  (`CreateAs`/`UpdateAs`/`DeleteAs`). A `users:*` manager can't make anyone
  admin and can't touch an admin's account (reset password, disable, delete).
- The last enabled admin who can sign in can't be disabled, deleted, demoted
  or left without a password.
- Without `users.bootstrap`, the first person to sign up becomes admin, then
  signup closes (unless `signup: true`).

When you add your own user endpoints, call the `*As` methods with the acting
identity, not `Create`/`Update`/`Delete`.

## Multiple replicas

The last-admin and first-signup checks are serialized by a mutex in the users
service: one process only. With several replicas, also enforce it in the
store, e.g. run the update in a transaction that re-counts enabled admins, or
create the bootstrap admin from a migration and set
`disable_first_user_signup: true`.
