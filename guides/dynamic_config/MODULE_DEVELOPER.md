# Webkernel Module Development Guide

## Auth, Database, and Mail Extensions

---

### Overview

Webkernel uses runtime config injection. You never touch the host application's
`config/auth.php`, `config/database.php`, or `config/mail.php` directly.
Instead, you implement a contract and register it in your service provider.
Webkernel applies your injection before Laravel's auth, DB, and mail services
initialize.

---

### 1. Adding a custom auth guard and user model

Implement `Webkernel\StdUser\Contracts\RegistersAuthProvider` in your module:

```php
namespace YourModule\Config;

use Webkernel\StdUser\Contracts\RegistersAuthProvider;
use YourModule\Models\YourUser;

final class YourAuthProvider implements RegistersAuthProvider
{
    public function providerName(): string
    {
        return 'your_module_users';
    }

    public function modelClass(): string
    {
        return YourUser::class;
    }

    public function guard(): ?array
    {
        return [
            'name'   => 'your_guard',
            'driver' => 'session',
        ];
    }
}
```

Dispatch it in your service provider `boot()`:

```php
use Webkernel\StdUser\Config\YourAuthProvider;

public function boot(): void
{
    $this->app['events']->dispatch(
        'webkernel.auth.register_provider',
        new YourAuthProvider()
    );
}
```

Result: `auth.providers.your_module_users` and `auth.guards.your_guard`
are available at runtime. No file was touched.

---

### 2. Adding a custom database connection

Implement `Webkernel\StdConf\Contracts\InjectsConfig`:

```php
namespace YourModule\Config;

use Webkernel\StdConf\Contracts\InjectsConfig;

final class YourDatabaseInjection implements InjectsConfig
{
    public function configScope(): string
    {
        return 'database';
    }

    public function configValues(): array
    {
        return [
            'database.connections.your_module_db' => [
                'driver'   => env('YOUR_MODULE_DB_DRIVER', 'sqlite'),
                'database' => env('YOUR_MODULE_DB_DATABASE', database_path('your_module.sqlite')),
                'prefix'   => 'ym_',
            ],
        ];
    }
}
```

Register it in your service provider `register()` (not `boot()`):

```php
use Webkernel\StdConf\Bootstrappers\ConfigInjectionDispatcher;
use YourModule\Config\YourDatabaseInjection;

public function register(): void
{
    $this->app->make(ConfigInjectionDispatcher::class)
        ->add(new YourDatabaseInjection());
}
```

Then point your model at it:

```php
protected $connection = 'your_module_db';
```

---

### 3. Adding a custom mailer

Same contract, different scope:

```php
public function configScope(): string
{
    return 'mail';
}

public function configValues(): array
{
    return [
        'mail.mailers.your_module_mailer' => [
            'transport' => 'smtp',
            'host'      => env('YOUR_MODULE_MAIL_HOST', '127.0.0.1'),
            'port'      => env('YOUR_MODULE_MAIL_PORT', 587),
            'username'  => env('YOUR_MODULE_MAIL_USERNAME'),
            'password'  => env('YOUR_MODULE_MAIL_PASSWORD'),
        ],
    ];
}
```

Use it in a Mailable or Notification:

```php
Mail::mailer('your_module_mailer')->to($recipient)->send(new YourMail());
```

---

### 4. Extending the user model

Your module user must extend `Webkernel\StdUser\Models\User`:

```php
namespace YourModule\Models;

use Filament\Panel;
use Webkernel\StdUser\Models\User as StdUser;

class YourUser extends StdUser
{
    protected $table = 'your_module_users';
    protected $connection = 'your_module_db';

    public function canAccessPanel(Panel $panel): bool
    {
        return $panel->getId() === 'your-panel-id';
    }
}
```

The migration for `your_module_users` is a plain Laravel migration.
Use only schema builder methods (no raw SQL) to stay agnostic across
SQLite, MySQL, and PostgreSQL.

---

### 5. Migration rules (stay agnostic)

Always use the schema builder. Never use raw SQL or driver-specific column types.

| Avoid                         | Use instead         |
| ----------------------------- | ------------------- |
| `DB::statement('CREATE ...')` | `Schema::create()`  |
| `$table->jsonb()`             | `$table->json()`    |
| `$table->tinyInteger(1)`      | `$table->boolean()` |
| `$table->mediumText()`        | `$table->text()`    |

A single migration file will run correctly on SQLite (dev), MySQL, and PostgreSQL (prod).

---

### Provider registration order

```
StdConfServiceProvider::register()    <-- binds ConfigInjectionDispatcher singleton
YourModule::register()                <-- calls dispatcher->add(new YourDatabaseInjection())
StdConfServiceProvider::boot()        <-- calls DatabaseBootstrapper, then dispatcher->apply()
StdUserServiceProvider::boot()        <-- sets auth.providers.users.model
YourModule::boot()                    <-- dispatches webkernel.auth.register_provider
```

Register database injections in `register()`.
Register auth providers in `boot()`.
