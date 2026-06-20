# 03 - Architecture

```
                 ┌────────────────────┐
                 │  webkernel/runtime   │  (composer plugin, merges everything)
                 └─────────┬────────────┘
                            │
        ┌───────────────────┼──────────────────────┐
        │                                            │
┌───────▼─────────┐                       ┌─────────▼──────────┐
│ webkernel/        │                       │ webkernel/           │
│ power-boards       │ ───requires────────▶ │ framework            │
└────────────────────┘                       └─────────┬────────────┘
                                                          │ requires
                                          ┌───────────────┴────────────────┐
                                          │                                  │
                              ┌───────────▼────────────┐      ┌─────────────▼─────────────┐
                              │ webkernel/std-functions   │◀────│ webkernel/std-svg-collection│
                              └────────────────────────────┘      └──────────────────────────────┘
                                  (standalone, no Laravel)            (standalone, no Laravel)
```

## Layers

1. **`std-*` packages** — pure PHP 8.4, zero Laravel dependency.
   Can be used in any project, e.g.:

    ```php
    require 'vendor/autoload.php';
    echo webkernel_grab_icon('arrow-right');
    ```

2. **`framework`** — wires Laravel 13 + Filament 5 into a `WebApp`
   application instance, depends on the `std-*` layer.

3. **`power-boards`** — feature module built on top of `framework`.

4. **`runtime`** — the Wikimedia-based composer merge plugin. Not part of
   the Webkernel namespace, untouched, orchestrates installation of all the
   above at build time.
