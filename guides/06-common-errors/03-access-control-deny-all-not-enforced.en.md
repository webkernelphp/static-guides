# 03 - Access control deny-all not enforced

> `WEBKERNEL_ACCESS_DENY_ALL=true` (or default `deny_all` in config) but `/system` dashboard still loads after login.

---

## Symptom

- `config('access-control.deny_all')` is `true`
- `webkernel_auth('page', Filament\Pages\Dashboard::class)` returns `false` in CLI
- Authenticated user still sees the Filament dashboard at `/system`

Authorization config looks correct; UI does not match.

---

## Root cause

### 1. Livewire hook registered too late (primary bug)

`Livewire::componentHook()` was called in `AccessControlServiceProvider::boot()`.

Livewire wires all component hooks inside `LivewireServiceProvider::boot()`:

```php
// LivewireServiceProvider::boot() — simplified
foreach ($features as $feature) {
    app('livewire')->componentHook($feature);
}
ComponentHookRegistry::boot(); // registers mount/hydrate listeners — once
```

Provider order: all `register()` methods run first, then all `boot()` methods. If your hook is only added during your `boot()` **after** Livewire already called `ComponentHookRegistry::boot()`, it sits in the hook list but **no mount/hydrate listener is attached**.

Filament pages keep their default `canAccess(): true`. Nothing calls `webkernel_auth()`.

### 2. `Gate::before` alone is insufficient

`Gate::before(fn () => false)` only affects **policy / Gate** checks (mostly resources). It does not intercept:

- `Filament\Pages\Page::canAccess()`
- `Filament\Widgets\Widget::canView()`
- Dashboard rendering for the default Filament page

---

## Fix (implemented)

`AccessControlServiceProvider` now:

1. Registers `Livewire::componentHook(FilamentAuthorizationHook::class)` in **`register()`**
2. Adds `EnforceFilamentAuthorization` to every panel via `Panel::configureUsing()` in **`register()`**
3. Keeps `Gate::before` + `Filament::serving()` in **`boot()`** as deny-all extras

### Verify middleware is on the panel

```bash
php -r "
require 'third_party/autoload.php';
\$app = require 'bootstrap/app.php';
\$app->make(Illuminate\Contracts\Console\Kernel::class)->bootstrap();
print_r(Filament\Facades\Filament::getPanel('system')->getAuthMiddleware());
"
```

Expect `Webkernel\Component\AccessControl\Filament\Http\Middleware\EnforceFilamentAuthorization` in the list.

---

## Expected behaviour after fix

| Step | Result |
| ---- | ------ |
| Guest visits `/system` | Login page |
| User logs in | **403** |
| Direct URL to dashboard | **403** |

```bash
php artisan config:clear
# restart php artisan serve if running
```

---

## If it still fails

| Check | Command / action |
| ----- | ---------------- |
| Config cached with old value | `php artisan config:clear` |
| `component-access-control` not discovered | `grep access-control bootstrap/cache/packages.php` |
| Old PHP process | Restart `artisan serve` / Octane / FPM workers |
| Custom panel without Filament auth middleware stack | Ensure panel is built through `Panel::make()` so `configureUsing` applies |

---

## Related

- [07 - Filament Runtime Authorization](../05-reference/07-filament-runtime-authorization.en.md)
- [04 - Filament & Dependency Injection](../04-for-developers/04-filament-and-dependency-injection.en.md)
