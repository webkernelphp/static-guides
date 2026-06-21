# Webkernel — Packages Documentation

This directory documents the internal package architecture of the Webkernel platform monorepo.

**The main human-readable documentation lives in `guides/`.**

See the numbered documentation (start here):

- [guides/00-index.en.md](guides/00-index.en.md)
- [guides/06-common-errors/00-index.en.md](guides/06-common-errors/00-index.en.md) — bootstrap failures, `view` errors, access-control wiring

Then follow the order:

- `01-introduction/`
- `02-vision/`
- `03-for-enterprises/`
- `04-for-developers/` (modules + package taxonomy)
- `05-reference/` (follow the numbers inside)

## Package Types (the real taxonomy)

Webkernel uses a strict classification (see `SLCPackageType`):

- `webkernel-component` — foundational pieces
- `webkernel-business-module` + `*-feature`
- `webkernel-platform-module` + `*-feature`

Developers mostly create modules of the last four types.

## Legacy / Internal notes

Some technical reference material has been moved into `guides/reference/`.

This top-level file is intentionally short. The real story is in `guides/`.
