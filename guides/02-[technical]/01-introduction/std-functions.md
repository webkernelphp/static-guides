# std-functions

See [`../std-functions/README.md`](../std-functions/README.md) for usage.

## Internal notes

- Namespace: `Webkernel\StdFunctions\`
- Entry point: `src/functions.php` (loaded via `autoload.files`)
- Path resolution assumes the standard monorepo layout:
  `packages/<name>/...` — two levels up from `src/` is `packages/`.
- If this package is ever moved outside the monorepo (split repo via
  `runtime`'s merge plugin), `webkernel_package()` path resolution must be
  re-verified against the new vendor directory layout.
