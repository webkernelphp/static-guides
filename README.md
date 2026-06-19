# Webkernel — Packages Documentation

This directory documents the internal package architecture of the
Webkernel platform monorepo.

## Index

- [Architecture](architecture.md) — how packages relate to each other
- [Versioning](versioning.md) — why every `webkernel/*` package shares one version
- [std-functions](std-functions.md) — standalone helper functions
- [std-svg-collection](std-svg-collection.md) — standalone icon collection

## Quick map

| Package                        | Standalone? | Depends on                  |
| ------------------------------ | ----------- | --------------------------- |
| `webkernel/std-functions`      | Yes         | —                           |
| `webkernel/std-svg-collection` | Yes         | `std-functions`             |
| `webkernel/framework`          | No          | Laravel, Filament, std-\*   |
| `webkernel/power-boards`       | No          | `framework`                 |
| `webkernel/runtime`            | —           | composer plugin, merges all |
