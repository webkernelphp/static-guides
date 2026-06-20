# 06 - Versioning policy

All `webkernel/*` packages (except `runtime`, which is a separate composer
plugin with its own lifecycle) share **one version number**, released
together.

This mirrors how `illuminate/*` (Laravel components) and `filament/*`
plugins are versioned: every component bumps in lockstep, regardless of
whether its internal code changed.

## Current version

````

0.11.3

```

Defined as the canonical version in:
- `packages/framework/composer.json` (`version`)
- `packages/framework/fast-boot.php` (`WEBKERNEL_VERSION`, `WEBKERNEL_SEMVER`)
- `packages/std-functions/composer.json`
- `packages/std-svg-collection/composer.json`
- `packages/power-boards/composer.json`

## Why

- Avoids dependency-resolution headaches (`std-functions: ^0.1.0` vs
  `framework` requiring `0.11.3` — confusing and error-prone).
- Makes it trivial to know "what version of Webkernel am I running" — one
  number, checkable via `WEBKERNEL_VERSION`.
- Internal cross-package requires use the exact pinned version
  (`"webkernel/std-functions": "0.11.3"`), not loose ranges, since
  `webkernel/runtime` pulls and merges everything together at once.

## Bumping the version

When releasing a new Webkernel version, update the version string in **all**
the files listed above to the same value.
````
