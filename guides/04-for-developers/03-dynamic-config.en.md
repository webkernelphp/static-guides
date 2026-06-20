# Dynamic Configuration (Auth, Database, Mail)

## Overview

Webkernel uses **runtime config injection**. You never touch the host application's `config/auth.php`, `config/database.php`, or `config/mail.php` directly.

Instead, you implement a contract and register it in your service provider. Webkernel applies your injection before Laravel initializes the services.

## 1. Adding a custom database connection

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

Register it in your service provider's `register()` method:

```php
use Webkernel\StdConf\Bootstrappers\ConfigInjectionDispatcher;

public function register(): void
{
    $this->app->make(ConfigInjectionDispatcher::class)
        ->add(new YourDatabaseInjection());
}
```

Then in your models:

```php
protected $connection = 'your_module_db';
```

## 2. Custom auth providers

Use the event in `boot()`:

```php
$this->app['events']->dispatch(
    'webkernel.auth.register_provider',
    new YourAuthProvider()
);
```

## 3. Migration rules (stay agnostic)

Always use the schema builder. Never use raw SQL or driver-specific column types.

| Avoid                         | Use instead         |
|-------------------------------|---------------------|
| `DB::statement('CREATE ...')` | `Schema::create()`  |
| `$table->jsonb()`             | `$table->json()`    |
| `$table->tinyInteger(1)`      | `$table->boolean()` |
| `$table->mediumText()`        | `$table->text()`    |

A single migration file will work on SQLite (dev), MySQL, and PostgreSQL (prod).

## Registration order (important)

```
StdConfServiceProvider::register()
YourModule::register()                 // add your InjectsConfig here
StdConfServiceProvider::boot()         // applies DatabaseBootstrapper + injections
StdUserServiceProvider::boot()
YourModule::boot()                     // dispatch auth providers here
```

- Database and mail injections → `register()`
- Auth providers → `boot()`

---

This is foundational. Most serious modules will need at least one database injection or auth provider.
