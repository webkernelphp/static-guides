# 01 - What is Webkernel?

Webkernel is a sovereign PHP application platform built on Laravel + Filament + Livewire.

Install it **once**. Then add features as independent, contract-based **modules**.

## The core idea

Instead of starting a new Laravel project every time (auth, panels, permissions, installer, tenancy, etc.), you get a complete foundation:

- Multi-panel backoffice
- Business scoping
- Powerful access control (platform privileges + module-level RBAC)
- Runtime configuration injection (database, auth, mail)
- Clean module lifecycle and installer
- Everything designed to be extended, not modified

## Who builds what

- **Numerimondes / core** builds the platform and foundational `webkernel-component` packages.
- **You (developers)** build **modules** of these types:

    - `webkernel-business-module`
    - `webkernel-business-module-feature`
    - `webkernel-platform-module`
    - `webkernel-platform-module-feature`

That's it. You don't rebuild the surface.

## Next steps

Follow the numbered sections in order:

1. [02 - Vision & Intentions](../02-vision/01-vision-intentions.en.md)
2. [03 - For Enterprises](../03-for-enterprises/01-overview.en.md)
3. [04 - For Developers](../04-for-developers/01-how-to-build-on-webkernel.en.md)
4. [05 - Reference](../05-reference/01-access-control.en.md) (start here, then follow the numbers)

---

This documentation is designed to be consumed linearly. The numbered prefixes remove any ambiguity about order.
