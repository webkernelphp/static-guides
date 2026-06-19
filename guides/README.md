# Webkernel, because building a web app is still too hard.

Writing the code got easier. What's still hard is everything around it — setting up the panels, the auth, the file management, the roles, the installer, the hosting, the tenant isolation. Every project starts the same way and most of the time gets abandoned before it ships. Webkernel aims to fix that.

---

## What it is today

Webkernel is a PHP application platform built on top of Laravel, Filament, and Livewire.

You install it once. You get:

- **ALREADY:**
    - A working multi-panel backoffice out of the box
    - A module system so you can add features without touching the core
    - A single-file installer that works over HTTP or CLI

- **SOON:**
    - Auth, RBAC, and multitenancy already wired in
    - A runtime collection builder so you can create new data structures without writing migrations
    - A server optimizer that configures OPcache, PHP-FPM, and Nginx the right way

It's a sovereign platform. You host it yourself, or through Numerimondes Cloud — with no vendor lock-in. No proprietary runtime. The stack underneath is all standard PHP — Laravel, Symfony, Composer. You own everything.

---

## What it's becoming

Right now, building a SaaS means picking a lane.

You either go deep in one area and miss everything else, or you go broad and nothing works well enough. You hire specialists for each piece, and then spend half your time gluing those pieces together.

That trade-off is going away.

Webkernel's goal is to cover the full range of what a web application needs to do — and let you, your team, or your agents go deep on the parts that are specific to your business.

The platform handles the surface so that you can handle the substance.

Concretely, this means:

- **Modules as the extension unit.** Any feature lives in a module. Modules follow a strict contract. They install cleanly, they don't break each other, and they can be distributed through Composer.
- **An agent-ready module surface.** The module contract is explicit enough that an AI agent can generate a valid module from a spec. Migrations, Filament resources, Livewire components — all of it. You describe what you need, the agent builds it, Webkernel runs it.
- **One-command deployment.** Through the partner portal and Numerimondes Cloud, spinning up a new Webkernel instance should take minutes, not a day of server configuration.
- **A distribution network.** Webkernel is both a tool for developers and a platform that coordinators and partners can deploy for their own clients — with pricing, commissions, and onboarding all handled.

---

## The principle

Most platforms make things possible by building everything for you.

Webkernel makes things possible by giving you a surface that's complete enough that you never have to start from scratch — but open enough that nothing is hidden from you.

The depth is yours. The breadth is ours.

---

## Status

Webkernel is in active development by [Numerimondes](https://numerimondes.com), an independent technology company based in Morocco.

The core architecture is stable. The module system and the installer are in production use. The managed cloud offer and partner network are being launched.

If you're a developer, an integrator, an agency, or anyone with a problem to solve and an idea to ship — reach out.

→ [numerimondes.com](https://numerimondes.com)  
→ [numerimondes.cloud](https://numerimondes.cloud)  
→ [numerimondes.partners](https://numerimondes.partners)
