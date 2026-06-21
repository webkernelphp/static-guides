# 07 - Filament Runtime Authorization

> How Webkernel enforces access on Filament panels without Shield, without generated policies for pages/widgets, and without editing Filament core classes.

**Package:** `webkernel/component-access-control`  
**Related:** [01 - Access Control](./01-access-control.en.md) (target RBAC design), [04 - Filament & Dependency Injection](../04-for-developers/04-filament-and-dependency-injection.en.md)

---

## Table of Contents

- [The problem](#the-problem)
- [What Filament checks natively](#what-filament-checks-natively)
- [What Webkernel adds today](#what-webkernel-adds-today)
- [The `webkernel_auth()` helper](#the-webkernel_auth-helper)
- [Deny-all test mode](#deny-all-test-mode)
- [Base classes for module authors](#base-classes-for-module-authors)
- [What you do as a module developer](#what-you-do-as-a-module-developer)
- [Troubleshooting](#troubleshooting)

---

## The problem

Filament splits authorization into several mechanisms:

| Surface | Filament mechanism | Goes through Laravel `Gate`? |
| ------- | ------------------ | ---------------------------- |
| Panel entry | `User::canAccessPanel()` | No |
| Custom pages | `static::canAccess()` | No |
| Widgets | `static::canView()` | No |
| Resources | Eloquent policies (`viewAny`, `create`, …) | Yes |
| Clusters | Delegates to child `canAccess()` | No |

Laravel dependency injection **cannot** inject an interface into Filament’s abstract base classes so that every child in every package is forced to implement it. The container resolves **instances**; Filament calls **static** methods on concrete classes.

Webkernel solves this with a **runtime layer** in `component-access-control` that does not require Filament Shield and does not patch vendor code.

---

## What Filament checks natively

### Panel (`canAccessPanel`)

Handled on the User model (you already have this). It runs **before** any page/widget/resource is reached. Webkernel does not replace it.

### Pages (`canAccess`)

Default implementation in `Filament\Pages\Concerns\CanAuthorizeAccess`:

```php
public static function canAccess(): bool
{
    return true;
}
```

Called from `mountCanAuthorizeAccess()` on mount and hydrate.

### Widgets (`canView`)

Default in `Filament\Widgets\Widget`:

```php
public static function canView(): bool
{
    return true;
}
```

Used when building dashboard widget lists and on widget hydrate.

### Resources

`Resource::canAccess()` delegates to `canViewAny()`, which uses Laravel policies / `Gate`.

---

## What Webkernel adds today

Three complementary layers ship in `AccessControlServiceProvider`:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Panel auth middleware (HTTP)                             │
│    EnforceFilamentAuthorization                             │
│    → deny-all blocks authenticated panel traffic            │
│    → guests can still reach login / register                │
├─────────────────────────────────────────────────────────────┤
│ 2. Livewire ComponentHook (runtime)                         │
│    FilamentAuthorizationHook                                │
│    → enforces webkernel_auth() on mount + hydrate           │
│    → Pages, Widgets, Resource pages, Clusters               │
│    → skips Filament\Auth\Pages\* (login, register, …)       │
├─────────────────────────────────────────────────────────────┤
│ 3. Gate::before (deny-all only)                             │
│    → blocks policy-based resource checks + navigation       │
└─────────────────────────────────────────────────────────────┘
```

### Registration details (important)

| Mechanism | Registered in | Why |
| --------- | ------------- | --- |
| `Livewire::componentHook()` | `register()` | Must run **before** `LivewireServiceProvider::boot()` calls `ComponentHookRegistry::boot()`. Registering in `boot()` is too late — hooks never fire. |
| `Panel::configureUsing()` | `register()` | Injects `EnforceFilamentAuthorization` into **every** panel’s `authMiddleware` without editing each `PanelProvider`. |
| `Gate::before()` | `boot()` | Only when `deny_all` is true. |
| `Filament::serving()` | `boot()` | Fallback for SPA / edge cases when deny-all is active. |

---

## The `webkernel_auth()` helper

Loaded from `load.component-access-control.functions.php`.

```php
// Explicit (what you would write inside a static method)
webkernel_auth('widget', __NAMESPACE__, __DIR__);

// FQCN shorthand — resolves namespace + directory via Reflection
webkernel_auth('page', static::class);
webkernel_auth('resource', MyResource::class);
```

### Component types

| `$type` | Filament target |
| ------ | --------------- |
| `page` | `Filament\Pages\Page` (not auth simple pages) |
| `widget` | `Filament\Widgets\Widget` |
| `resource` | Resource class or resource page class |
| `cluster` | `Filament\Clusters\Cluster` |

### Service behind the helper

`Webkernel\Component\AccessControl\Auth\WebkernelAuth`:

- `check($type, $namespaceOrClass, $directory = null): bool`
- `checkClass($type, $class): bool`
- `deniesAll(): bool`

When full RBAC is implemented, `check()` will resolve module slug from namespace/directory and query the dynamic permission registry. Today it supports **`deny_all`** mode and a permissive default when deny-all is off.

---

## Deny-all test mode

Use this to verify the wiring — not for production.

### Configuration

```php
// packages/component-access-control/config/access-control.php
'deny_all' => env('WEBKERNEL_ACCESS_DENY_ALL', true),
```

```env
# .env — turn off when done testing
WEBKERNEL_ACCESS_DENY_ALL=false
```

### Expected behaviour

| Step | Expected |
| ---- | -------- |
| Visit `/system` as guest | Login page loads |
| Log in | **403** on any authenticated panel route |
| Dashboard, widgets, resources | **403** (or hidden from nav for resources) |

### Verify from CLI

```bash
php artisan config:clear

php -r "
require 'third_party/autoload.php';
\$app = require 'bootstrap/app.php';
\$app->make(Illuminate\Contracts\Console\Kernel::class)->bootstrap();
echo 'deny_all: ' . var_export(config('access-control.deny_all'), true) . PHP_EOL;
echo 'dashboard: ' . var_export(webkernel_auth('page', Filament\Pages\Dashboard::class), true) . PHP_EOL;
"
```

With deny-all active, `dashboard` must print `false`.

---

## Base classes for module authors

Optional but recommended for **navigation** visibility (static `canAccess` / `canView`):

| Base class | Namespace | Overrides |
| ---------- | --------- | --------- |
| `AuthorizedPage` | `Webkernel\Component\AccessControl\Filament\Pages` | `canAccess()` |
| `AuthorizedWidget` | `Webkernel\Component\AccessControl\Filament\Widgets` | `canView()` |
| `AuthorizedResource` | `Webkernel\Component\AccessControl\Filament\Resources` | `canAccess()` |

```php
use Webkernel\Component\AccessControl\Filament\Pages\AuthorizedPage;

final class PayrollSettingsPage extends AuthorizedPage
{
    // canAccess() already delegates to webkernel_auth('page', static::class)
}
```

Third-party Filament widgets (e.g. `AccountWidget`) do not extend these bases. They are still blocked at runtime by `FilamentAuthorizationHook` when `webkernel_auth()` denies access.

---

## What you do as a module developer

### Minimum (zero boilerplate in each class)

1. Ensure `webkernel/component-access-control` is installed (pulled in by `webkernel/codebase`).
2. Build Filament pages/widgets/resources as usual.
3. Runtime enforcement applies automatically via the hook + panel middleware.

### Recommended (clean navigation + static checks)

1. Extend `AuthorizedPage` / `AuthorizedWidget` / `AuthorizedResource`.
2. Declare module permissions via `AccessControl::register()` when the RBAC layer is complete (see [01 - Access Control](./01-access-control.en.md)).

### Do not

- Rely on `Gate::before` alone — it does not cover pages or widgets.
- Register Livewire hooks in `boot()` of your own provider — same timing bug as below.
- Add `canAccess()` copies in every class once bases or the hook cover your case.

---

## Troubleshooting

| Symptom | Likely cause | Doc |
| ------- | ------------ | --- |
| `webkernel_auth()` returns `false` but UI still loads | Hook registered too late | [03 - Deny-all not enforced](../06-common-errors/03-access-control-deny-all-not-enforced.en.md) |
| `Target class [view] does not exist` on `artisan serve` | Stale package manifest / missing provider class | [02 - Target class view](../06-common-errors/02-target-class-view-does-not-exist.en.md) |
| Resources visible, pages not blocked | Only `Gate::before` active, no hook/middleware | This document — [What Webkernel adds](#what-webkernel-adds-today) |

---

## Future work

The [01 - Access Control](./01-access-control.en.md) document describes the full privilege + business + RBAC design. `WebkernelAuth::check()` is the integration point where that design will land. The runtime hook and middleware stay; only the decision inside `check()` becomes module-aware.
