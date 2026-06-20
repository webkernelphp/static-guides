# Work Progress - Documentation & Structure

**Author / Owner:** Yassine (as per request)

**Date:** 2026-06-20

## Goals for Documentation

- Create a **real, clean docs folder** under `packages/static-guides/guides/` that reflects the actual Webkernel reality and package taxonomy.
- Structure documentation for **different audiences** (things are not the same for everyone):
    - People who want to understand **what's in my head** (intentions, global vision, strategy, why Numerimondes built this).
    - **Clients & Enterprises** (institutions, governments, companies in North Africa and beyond) — focus on benefits, sovereignty, no-code/low-code/full-code, ownership.
    - **Developers** — but be very clear: developers do **not** build full apps from scratch. They create **packages** of these types only:
        - `webkernel-business-module`
        - `webkernel-business-module-feature`
        - `webkernel-platform-module`
        - `webkernel-platform-module-feature`
          (See `packages/standard-lifecycle/src/Installer/SLCPackageType.php`)
- All English documentation files must use the `.en.md` suffix (preparation for future multilingual doc site generation).
- Make everything **simpler**, more direct, and honest.
- Clearly present that **Webkernel is the product of Numerimondes** (even if we don't always say it loudly).
- Present the documentation as a living thing "en devenir" (work in progress, becoming real).

## Current Tasks

- [x] Write this work progress file (first priority)
- [x] Designed and created clean audience-oriented structure in `packages/static-guides/guides/`:
    - `README.en.md`
    - `vision/vision-intentions.en.md`
    - `for-enterprises/overview.en.md`
    - `for-developers/` (package taxonomy, dynamic config, etc.)
    - `reference/` (access-control, internals, lifecycle, std-\*, architecture...)
- [x] All files suffixed with `.en.md`
- [x] Content made simpler and separated by audience (vision vs enterprises vs devs)
- [x] Package taxonomy clearly documented (business-module, platform-module, components, features)
- [x] Updated top-level README.md
- [x] Deleted `packages/static-guides/___trash_guides`
- [ ] Continue content improvement and simplification
- [ ] Add more depth in vision and enterprises sections
- [ ] Prepare for `.fr.md` + multi-lang doc site generation

## Notes

- Webkernel is both:
    - A sovereign application engine for modules
    - Numerimondes' core product
- Devs build **modules** following strict contracts. The platform provides the surface (auth, RBAC, business scoping, installer, etc.).
- We must not over-document like a generic Laravel thing. Focus on what makes Webkernel different.

## Later

- Generate doc site from this folder with proper multi-language support.
- Possibly extract parts into the actual package READMEs (component-_, standard-_, etc.).

---

**Status:** Major documentation restructuring done. New audience-based structure with `.en.md` files is live. Trash cleaned. Now in iteration / content improvement phase.
