# Webkernel Internal Architecture

## Config Injection, User Blueprint, and Multi-DB

### Package map

| Package                                       | Responsibility                                 |
| --------------------------------------------- | ---------------------------------------------- |
| `webkernel/component-configuration` (StdConf) | Runtime config injection, DatabaseBootstrapper |
| `webkernel/component-user` (StdUser)          | User blueprint, auth provider registration     |
| `webkernel/component-business`                | Business as scoping unit                       |
| `webkernel/component-access-control`          | Privileges + dynamic RBAC                      |

### Main connection

The installer writes `storage/webkernel/database.json`.

`DatabaseBootstrapper` injects it as the connection named **`webkernel_primary`**.

This is the default for core platform models (User, Business, privileges, etc.).

### User inheritance

```
Illuminate\Foundation\Auth\User
  ^
Webkernel\StdUser\Models\User          (abstract base)
  ^
Webkernel\Models\User                  (empty extension point)
  ^
App\Models\User                        (your app — you can override $table / $connection)
```

Modules that need their own users usually extend `Webkernel\StdUser\Models\User` directly.

### Adding new connections from modules

Use `InjectsConfig` (see [Dynamic Config](../04-for-developers/03-dynamic-config.en.md)).

---

This document is intentionally short. The important constraints are:

- One primary connection injected at boot
- No direct editing of Laravel config files
- All core data should be happy on SQLite / MySQL / PostgreSQL with the same migrations

See also: [04.03 - Dynamic Config](../04-for-developers/03-dynamic-config.en.md) and the component packages.
