# 01 - How to Build on Webkernel

Welcome.

If you're here, you probably want to build something on Webkernel.

**First reality check:**

You do not build "a Webkernel application" the way you build a normal Laravel app.

You build **modules**.

The platform already gives you:

- Auth & sessions
- Multi-panel Filament backoffice
- Business scoping
- Access control (privileges + dynamic RBAC)
- Dynamic database connection injection
- Installer + lifecycle
- Module discovery

Your job is to deliver vertical or technical capabilities as clean, contract-respecting packages.

## The Only Package Types You Create

See the full taxonomy here: [02 - Package Types](./02-package-types.en.md)

In practice, 95% of what external developers build falls into:

- `webkernel-business-module`
- `webkernel-business-module-feature`
- `webkernel-platform-module`
- `webkernel-platform-module-feature`

## Core Rules

1. Follow the contracts (see `WebkernelModule` interface in access-control thinking).
2. Never touch `config/database.php`, `config/auth.php` directly — use `InjectsConfig`.
3. Use only Schema Builder in migrations (no raw SQL, no driver-specific types).
4. Declare your permissions cleanly.
5. Do not assume a specific database engine.

## Where to start

- [02 - Package Types](./02-package-types.en.md)
- [03 - Dynamic Config](./03-dynamic-config.en.md)
- [04 - Filament & Dependency Injection](./04-filament-and-dependency-injection.en.md)
- [07 - Filament Runtime Authorization](../05-reference/07-filament-runtime-authorization.en.md)
- [06 - Common Errors](../06-common-errors/00-index.en.md)

## The spirit

Webkernel exists so that one competent developer (or a small team) can deliver serious systems without spending 80% of their time on boilerplate and glue.

Build focused modules. Let the platform handle the surface.

---

This section is for people who ship modules. If you're looking for the "why", read [02 - Vision & Intentions](../02-vision/01-vision-intentions.en.md).
