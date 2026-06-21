# 04 - Filament & Dependency Injection

> Can Laravel inject interfaces into Filament `PanelProvider`, `Page`, `Resource`, `Cluster`, and `Widget`? What actually works in Webkernel.

**See also:** [07 - Filament Runtime Authorization](../05-reference/07-filament-runtime-authorization.en.md)

---

## Short answer

**No** — you cannot use Laravel DI to force every Filament child class across packages to implement an interface on the Filament base classes themselves. Those classes are vendor code; their static authorization methods never pass through the container.

**Yes** — you can inject dependencies into **your own** `PanelProvider`, and you can centralize cross-module behaviour with **contracts + tags**, **Filament plugins**, or Webkernel’s **runtime authorization layer**.

---

## What DI can and cannot do

| Target | Constructor DI? | Method DI? | Static `canAccess` / `canView` via DI? |
| ------ | --------------- | ---------- | -------------------------------------- |
| `PanelProvider` | Yes | `panel(Panel $panel)` only receives `Panel` | N/A |
| `Page` / `Widget` / `Resource` | If resolved from container | `mount()`, `form()`, `table()` | **No** — Filament calls static methods directly |
| Third-party Filament classes | No control | No | **No** |

### Why static methods break DI

```php
// Filament internal call pattern
abort_unless(static::canAccess(), 403);
```

There is no `$app->make()` here. Injecting `WebkernelAuth` into a base class constructor does not help unless **your** class is instantiated through the container and **your** static method calls the service:

```php
public static function canAccess(): bool
{
    return app(WebkernelAuth::class)->checkClass('page', static::class);
}
```

That is exactly what `AuthorizedPage` does — but only for classes that extend it.

---

## Patterns that work in Webkernel

### 1. Runtime interception (zero touch per class)

`component-access-control` registers:

- `FilamentAuthorizationHook` (Livewire `ComponentHook`)
- `EnforceFilamentAuthorization` (panel `authMiddleware` via `Panel::configureUsing()`)

No module changes required. Covers pages, widgets, resource pages, clusters at HTTP/Livewire runtime.

→ [07 - Filament Runtime Authorization](../05-reference/07-filament-runtime-authorization.en.md)

### 2. Base classes (static navigation + runtime)

```php
// Module page
final class ReportsPage extends AuthorizedPage { }

// Module widget
final class RevenueChart extends AuthorizedWidget { }

// Module resource
final class InvoiceResource extends AuthorizedResource { }
```

Use when you want navigation items hidden **before** mount, not only 403 at runtime.

### 3. Contracts + container tags (cross-module panel composition)

For contributing to a shared panel without editing `SystemPanelProvider` for every module:

```php
interface RegistersFilamentPanel
{
    public function register(Panel $panel): void;
}

// Module ServiceProvider
$this->app->singleton(BillingPanelRegistrar::class);
$this->app->tag(BillingPanelRegistrar::class, 'filament.system');

// SystemPanelProvider::panel()
public function __construct(private readonly Container $app) {}

public function panel(Panel $panel): Panel
{
    foreach ($this->app->tagged('filament.system') as $registrar) {
        $registrar->register($panel);
    }

    return $panel->id('system')->path('system');
}
```

DI works on **your** registrar class. Filament base classes stay untouched.

### 4. Filament `Plugin` contract

```php
final class BillingPlugin implements Filament\Contracts\Plugin
{
    public function __construct(private readonly BillingService $billing) {}

    public function register(Panel $panel): void { /* ... */ }
    public function boot(Panel $panel): void { /* ... */ }
}

// In panel()
->plugin(app(BillingPlugin::class))
```

Constructor injection on the plugin class is fully supported.

### 5. `Gate::before` — resources only

```php
Gate::before(fn ($user, $ability) => false); // deny all policy checks
```

| Covers | Does not cover |
| ------ | -------------- |
| Resource `viewAny`, `create`, policies | `Page::canAccess()`, `Widget::canView()`, `canAccessPanel()` |

Use as **one layer** for resources, not as the whole solution.

---

## Decision guide

```
Need to block third-party Filament widgets/pages at runtime?
  → Runtime hook + middleware (already in component-access-control)

Need navigation hidden for your module's pages/widgets?
  → Extend AuthorizedPage / AuthorizedWidget / AuthorizedResource

Need many modules to register routes/resources on one panel?
  → Contracts + tagged registrars, or Filament Plugin

Need resource CRUD permissions?
  → DynamicPolicy + AccessControl RBAC (see 05-reference/01-access-control)
  → Gate::before is only a testing shortcut for deny-all
```

---

## Anti-patterns

| Do not | Why |
| ------ | --- |
| Patch `Filament\Pages\Page` in vendor | Lost on every update |
| Register Livewire hooks in `ServiceProvider::boot()` | Too late — see common errors doc |
| Assume `Gate::before` secures the whole panel | Pages and widgets bypass Gate |
| Put `View::addNamespace()` in `withExceptions` before `ViewServiceProvider` boots | Misleading `Target class [view] does not exist` errors |

---

## Related errors

- [02 - Target class view does not exist](../06-common-errors/02-target-class-view-does-not-exist.en.md)
- [03 - Access control deny-all not enforced](../06-common-errors/03-access-control-deny-all-not-enforced.en.md)
