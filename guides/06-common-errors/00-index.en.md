# 06 - Common Errors

Quick index for operational failures seen on real Webkernel projects.

| # | Topic | When to read |
| - | ----- | ------------ |
| [01](./01-resolving-laravel-package-manifest-bootstrap-failures.en.md) | Package manifest / empty `bootstrap/cache/packages.php` | Routes or providers missing after `composer install` |
| [02](./02-target-class-view-does-not-exist.en.md) | `Target class [view] does not exist` | Artisan crashes; often stale provider after package removal |
| [03](./03-access-control-deny-all-not-enforced.en.md) | Deny-all config true but panel still open | Filament access control testing |

**Rule of thumb:** read the **root error** in the stack trace (first exception), not the last wrapper message.
