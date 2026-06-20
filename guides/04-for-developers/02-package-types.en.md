# 02 - Webkernel Package Types

This is the real taxonomy. Everything is classified this way.

Defined in `Webkernel\StdLifecycle\Installer\SLCPackageType`.

## Infrastructure & Core (platform level)

| Type        | Composer type         | Purpose                                                                               |
| ----------- | --------------------- | ------------------------------------------------------------------------------------- |
| `Assets`    | `webkernel-assets`    | Asset bundles                                                                         |
| `Component` | `webkernel-component` | Foundational capability (user, access-control, business, configuration, plugin, etc.) |
| `DevTool`   | `webkernel-devtool`   | Development tools                                                                     |
| `Stdlib`    | `webkernel-stdlib`    | Pure PHP standard library (no Laravel)                                                |
| `Element`   | `webkernel-element`   | Reusable UI / Filament elements                                                       |
| `Agent`     | `webkernel-agent`     | Agentic / AI worker packages                                                          |
| `Ffi`       | `webkernel-ffi`       | Native extensions via FFI                                                             |

These are mostly built by the core team.

## Business Architecture (what most domain developers build)

| Type                    | Composer type                       | Description                                                                          |
| ----------------------- | ----------------------------------- | ------------------------------------------------------------------------------------ |
| `BusinessModule`        | `webkernel-business-module`         | Autonomous business domain module (payroll, CRM, inventory, training, compliance...) |
| `BusinessModuleFeature` | `webkernel-business-module-feature` | Additional feature attached to a business module. Requires `extra.webkernel.module`  |

**Business modules** are scoped to a Business. They get business-aware permissions, activation, etc.

## Platform Architecture (technical / cross-cutting)

| Type                    | Composer type                       | Description                                                                                  |
| ----------------------- | ----------------------------------- | -------------------------------------------------------------------------------------------- |
| `PlatformModule`        | `webkernel-platform-module`         | Core platform or technical infrastructure (logs, billing, identity providers, monitoring...) |
| `PlatformModuleFeature` | `webkernel-platform-module-feature` | Feature attached to a platform module                                                        |

## How features work

Feature packages **must** declare their parent module in `composer.json`:

```json
"extra": {
    "webkernel": {
        "module": "your-parent-module-slug"
    }
}
```

## Why this taxonomy exists

- It makes module composition predictable.
- It allows the platform (and agents) to reason about dependencies and scoping.
- Business modules are isolated by business. Platform modules are global.
- Features are always attached — no orphan code.

This is not the usual "just make a Laravel package" free-for-all. The constraints are the point.

---

**Developers almost only ever create the four business/platform module types.**

Everything else is foundational platform work.
