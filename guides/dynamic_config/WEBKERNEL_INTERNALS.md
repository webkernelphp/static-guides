# Webkernel Internal Architecture

## Config Injection, User Blueprint, and Multi-DB

---

### Package map

| Package                   | Responsibility                                                                                   |
| ------------------------- | ------------------------------------------------------------------------------------------------ |
| `webkernel/std-conf`      | Runtime config injection engine. DatabaseBootstrapper. ConfigInjectionDispatcher.                |
| `webkernel/std-user`      | User blueprint (abstract Eloquent + Filament). AuthConfigurator. RegistersAuthProvider contract. |
| `webkernel/std-lifecycle` | vendor:publish interception, CommandReplacer.                                                    |

---

### Config injection lifecycle

```
Application boot
  |
  +-- StdConfServiceProvider::register()
  |     Binds ConfigInjectionDispatcher as singleton.
  |
  +-- [All module service providers]::register()
  |     Each calls dispatcher->add(new TheirInjection()).
  |     Database and mail injections only. Auth goes in boot().
  |
  +-- StdConfServiceProvider::boot()
  |     1. DatabaseBootstrapper reads storage/webkernel/database.json
  |        and sets database.connections.webkernel_primary.
  |     2. dispatcher->apply() iterates all queued InjectsConfig objects
  |        and calls config->set() for each key.
  |
  +-- StdUserServiceProvider::boot()
  |     AuthConfigurator sets auth.providers.users.model = App\Models\User.
  |     Registers webkernel.auth.register_provider event listener.
  |
  +-- [All module service providers]::boot()
        Modules dispatch webkernel.auth.register_provider with their
        RegistersAuthProvider instance. AuthConfigurator injects the
        new provider and optional guard into the auth config.
```

---

### User inheritance chain

```
Illuminate\Foundation\Auth\User (Laravel Authenticatable)
  ^
  |
Webkernel\StdUser\Models\User   (abstract, Filament contracts, Gravatar, $table = 'users')
  ^
  |
Webkernel\Models\User           (empty, distributed with webkernel/models or webkernel/std-user)
  ^
  |
App\Models\User                 (host application, can override $table, $connection, $fillable)
  ^
  |
YourModule\Models\YourUser      (module-level, own $table, $connection, canAccessPanel)
```

Modules extend `Webkernel\StdUser\Models\User` directly when they need a fully
independent user type (agent, customer, system account). They extend
`App\Models\User` only when they want to add behaviour to the primary app owner user.

---

### DatabaseBootstrapper state file

Written by the installer screen at: `storage/webkernel/database.json`

```json
{
    "driver": "pgsql",
    "database": "webkernel_prod",
    "host": "127.0.0.1",
    "port": "5432",
    "username": "wk",
    "password": "secret"
}
```

The bootstrapper reads this once per request boot. It does not cache.
If the file does not exist, the bootstrapper is a no-op (installer not yet run).

---

### Adding a new supported DB driver

1. Add the driver string to `SUPPORTED_DRIVERS` in `DatabaseBootstrapper`.
2. Add a `match` arm in `buildConnectionConfig()`.
3. Document the installer field requirements.

ClickHouse and other non-relational engines will not use DatabaseBootstrapper.
They will use direct `InjectsConfig` implementations in their respective modules,
loaded only when the module is active. Auth-incompatible drivers must be flagged
in the module's `composer.json` under `extra.webkernel.auth_capable = false`.

---

### Blocking vendor:publish for protected config files

CommandReplacer intercepts `vendor:publish`. When the `--tag` or `--provider`
targets a Webkernel-protected config (auth, database, mail), the replacer can:

- Block the command entirely and print a message.
- Substitute a Webkernel-managed stub instead.

This prevents developers from accidentally overwriting the thin proxy files that
Webkernel expects at `config/auth.php`, `config/database.php`, etc.

---

### Migration strategy

All Webkernel and module migrations use only Laravel Schema Builder methods.
Raw SQL and driver-specific column types are banned. This guarantees that a
single migration file works on SQLite (dev/test), MySQL, and PostgreSQL (prod)
with zero modification.

For post-relational engines (ClickHouse, etc.), modules ship a dedicated
migration runner or schema initializer that is invoked only when that connection
is active. These are not standard Laravel migrations.
