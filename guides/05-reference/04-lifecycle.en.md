# Webkernel Standard Lifecycle — Package Types

This package defines the Composer package types recognized by the Webkernel
installer (`SLCPackageType`), and the logic that resolves where each package
type gets installed (`SLCInstallerLocations`, `SLCBaseInstaller`).

## The taxonomy

```php
// inside Webkernel\StdLifecycle\Installer\SLCPackageType::class

enum SLCPackageType: string
{
    case Module          = 'webkernel-module';
    case ModuleFeature   = 'webkernel-module-feature';
    case Ffi             = 'webkernel-ffi';

    case Stdlib  = 'webkernel-stdlib';
    case Engine  = 'webkernel-engine';
    case Element = 'webkernel-element';
    case Agent   = 'webkernel-agent';
    case Assets  = 'webkernel-assets';
    case DevTool = 'webkernel-devtool';
}
```

Nine types. They are not all equal, and that inequality is intentional.

## Two tiers of meaning

### Tier 1 — types with a real installer contract

`Module`, `ModuleFeature`, and `Ffi` are the only types that currently change
**where** a package is installed and, for `ModuleFeature`, **what is
required** for it to install at all.

- `webkernel-module` → installed under `modules/{vendor}/{name}/`
- `webkernel-module-feature` → installed under
  `modules/{parentVendor}/{parentName}/features/{vendor}-{name}/`, and
  **requires** `extra.webkernel.module` in the package's `composer.json`.
  If it's missing, the installer throws. This is a real guard, enforced
  at install time, not a convention.
- `webkernel-ffi` → installed under `ffi/{vendor}/{name}/`

If you declare a package as `webkernel-module-feature` without declaring its
parent module, installation fails. That's by design.

### Tier 2 — types that are classification only

`Stdlib`, `Engine`, `Element`, `Agent`, `Assets`, and `DevTool` all resolve to
the exact same install path: the default Composer `vendor/{vendor}/{name}/`.
None of them currently change installer behavior. None of them are currently
verified against anything.

**This means the type, for these six cases, is a declaration of intent —
not a technical contract.**

Nothing today stops a developer from tagging a package `webkernel-agent`
when it has nothing to do with an autonomous worker. There is no guard
checking that an `Engine` actually behaves like a foundational capability,
or that `Assets` actually only contains static files. The installer accepts
the type, files it under `vendor/`, and moves on.

This is the opposite of `ModuleFeature`, which is enforced. The asymmetry is
real and worth stating plainly so nobody loses time looking for a behavioral
difference between, say, `Engine` and `Stdlib` that does not exist yet.

## Why keep nine types if six of them do nothing differently

Because the type still has value as a **label read by humans, tools, and
agents** without needing to open `extra.webkernel` and parse nested JSON.
`composer show -t webkernel-agent` or a registry scanner can filter the
ecosystem by declared nature directly from the package type. That's worth
something even without enforcement.

The decision to keep this classification granular, rather than collapsing
everything into a single `webkernel-stdlib` with a `kind` attribute, is
deliberate: it's a bet that explicit types are more discoverable than a
buried metadata field, even at the cost of not being verified — for now.

## What changes later: distribution-time verification

The lack of a guard on Tier 2 types is a **current** state, not a permanent
one. As Webkernel moves toward an automated or semi-automated distribution
pipeline (publishing to a package registry, agent-assisted module
generation, partner submissions), each type can gain its own required
`extra.webkernel` shape, checked before a package is accepted into the
distribution.

For example, a future rule could require:

```json
{
    "type": "webkernel-agent",
    "extra": {
        "webkernel": {
            "agent": {
                "dispatch": "queue",
                "autonomous": true
            }
        }
    }
}
```

And the distribution pipeline would reject any package declaring
`webkernel-agent` without that shape present — the same pattern already
used by `ModuleFeature` today with `extra.webkernel.module`.

**No such requirement exists yet for `Stdlib`, `Engine`, `Element`, `Agent`,
`Assets`, or `DevTool`.** This README exists so that whoever adds that
enforcement later does so deliberately, type by type, rather than assuming
it already exists somewhere.

## Summary

| Type                       | Install path                                                    | Enforced today                           |
| -------------------------- | --------------------------------------------------------------- | ---------------------------------------- |
| `webkernel-module`         | `modules/{vendor}/{name}/`                                      | path only                                |
| `webkernel-module-feature` | `modules/{parentVendor}/{parentName}/features/{vendor}-{name}/` | path + required `extra.webkernel.module` |
| `webkernel-ffi`            | `ffi/{vendor}/{name}/`                                          | path only                                |
| `webkernel-stdlib`         | `third_party/{vendor}/{name}/`                                  | nothing                                  |
| `webkernel-engine`         | `third_party/{vendor}/{name}/`                                  | nothing                                  |
| `webkernel-element`        | `third_party/{vendor}/{name}/`                                  | nothing                                  |
| `webkernel-agent`          | `third_party/{vendor}/{name}/`                                  | nothing                                  |
| `webkernel-assets`         | `third_party/{vendor}/{name}/`                                  | nothing                                  |
| `webkernel-devtool`        | `third_party/{vendor}/{name}/`                                  | nothing                                  |

Tier 2 enforcement is a roadmap item, not a missing feature being silently
worked around. Treat any `extra.webkernel.*` shape for these six types as
optional and unverified until this README says otherwise.
