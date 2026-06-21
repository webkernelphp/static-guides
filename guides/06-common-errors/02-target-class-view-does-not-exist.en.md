# 02 - Target class [view] does not exist

> `php artisan serve`, `package:discover`, or any Artisan command fails with `Target class [view] does not exist` / `Class "view" does not exist`.

---

## Symptom

```text
In Container.php line 1127:

  Target class [view] does not exist.


In Container.php line 1125:

  Class "view" does not exist
```

Often seen when running:

```bash
php artisan serve
composer dump-autoload   # during package:discover
```

The message points at the **view** service, but that is usually **not** the root cause.

---

## Root cause (most common in Webkernel)

Bootstrap fails while registering a **missing service provider** from a stale `bootstrap/cache/packages.php` entry. Laravel then tries to render the exception; `WebApp::configure()` previously called `View::addNamespace()` inside the exception handler setup **before** `ViewServiceProvider` had registered the `view` binding — which produced the misleading `view` error.

### Typical trigger

A package was **removed or renamed** (example: `webkernel/component-http` deleted) but its provider is still listed in the manifest:

```php
// bootstrap/cache/packages.php (stale fragment)
'webkernel/component-http' => [
    'providers' => [
        'Webkernel\\Component\\Http\\Providers\\HttpServiceProvider', // class gone
    ],
],
```

The real first error is:

```text
Class "Webkernel\Component\Http\Providers\HttpServiceProvider" not found
```

---

## Fix

### 1. Remove stale manifest entries

Either delete the dead package block from `bootstrap/cache/packages.php` or regenerate the manifest:

```bash
php artisan package:clear
php artisan package:discover
```

### 2. Refresh Composer lock

If `composer.lock` still references the removed package:

```bash
composer update webkernel/codebase
```

### 3. Confirm the provider class is gone from cache

```bash
grep -r "HttpServiceProvider\|component-http" bootstrap/cache/
```

No matches should remain.

### 4. Verify Artisan boots

```bash
php artisan list
php artisan serve
```

---

## Secondary issue (fixed in codebase)

`WebApp::configure()` registered view namespaces inside `withExceptions()`. That callback runs when the exception handler is first resolved — which can happen **before** `Illuminate\View\ViewServiceProvider` registers `view`.

**Rule:** register Blade/view namespaces in `Webkernel\ServiceProvider::boot()` (or any provider that runs after the view service exists), not in the exception handler setup closure.

---

## `bootstrap/app.php` note

`WebApp::configure()` already passes providers via `->withProviders([...])`. Chaining `->withProviders([])` in `bootstrap/app.php` merges an empty array and is harmless, but it is unnecessary noise. The manifest and provider class existence matter more than this chain.

---

## Prevention checklist

| Check | Action |
| ----- | ------ |
| Package removed from `packages/` | Run `composer update` + `package:discover` |
| `COMPOSER_VENDOR_DIR` unset in subprocesses | Ensure `putenv('COMPOSER_VENDOR_DIR=' . vendor_dir())` runs (see [01 - Package manifest failures](./01-resolving-laravel-package-manifest-bootstrap-failures.en.md)) |
| New clone / CI | Never commit empty `bootstrap/cache/packages.php` without running discover |

---

## Related

- [01 - Package manifest bootstrap failures](./01-resolving-laravel-package-manifest-bootstrap-failures.en.md)
- [02 - Webkernel Internals](../05-reference/02-webkernel-internals.en.md)
