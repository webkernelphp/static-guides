# Webkernel — Packages Documentation

This directory documents the internal package architecture of the Webkernel platform monorepo.

**The main human-readable documentation lives in `guides/`.**

See:

- [guides/README.en.md](guides/README.en.md)
- [guides/vision/vision-intentions.en.md](guides/vision/vision-intentions.en.md)
- [guides/for-developers/](guides/for-developers/)
- [guides/for-enterprises/](guides/for-enterprises/)
- [guides/reference/](guides/reference/)

## Package Types (the real taxonomy)

Webkernel uses a strict classification (see `SLCPackageType`):

- `webkernel-component` — foundational pieces
- `webkernel-business-module` + `*-feature`
- `webkernel-platform-module` + `*-feature`

Developers mostly create modules of the last four types.

## Legacy / Internal notes

Some technical reference material has been moved into `guides/reference/`.

This top-level file is intentionally short. The real story is in `guides/`.
