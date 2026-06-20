# 01 - Vision & Intentions

> This document tries to capture what is in our head — the real reasons Webkernel exists.

## Webkernel, because building a web app is still too hard

Writing the code got easier. What's still hard is everything around it — setting up the panels, the auth, the file management, the roles, the installer, the hosting, the tenant isolation, the multi-panel backoffice, the module system...

Every project starts the same way and most of the time gets abandoned before it ships.

That cost has always fallen hardest on the ones who need software the most and have the least access to build it — companies and institutions across North Africa and around the world, governments included.

Webkernel aims to fix that.

## Who it's for (the honest version)

Webkernel is built for **enterprises and institutions first**.

- A company shouldn't need a full dev team to get an ERP, a CRM, a training portal, or a compliance system that's actually theirs.
- A ministry shouldn't have to depend on a foreign vendor to run its own infrastructure.

That's the gap.

**No-code where possible, low-code where needed, full code when you want to go deeper.**

We see the future like this:

> Developers and agencies build **on** Webkernel (by creating modules).  
> Everyone else just **uses** it.

## What it is today

You install Webkernel once. You get:

- A working multi-panel backoffice out of the box
- A real module system (features plug in like apps)
- A single-file installer (HTTP or CLI)
- Foundational capabilities (auth, access control, business scoping, dynamic configuration...)

## The deeper intention

Most platforms make things possible by building everything for you, then charging you for every feature you'll never use.

Webkernel makes things possible by giving you a **surface** that's complete enough that you never have to start from scratch — but open enough that nothing is hidden from you, and simple enough that you don't need to be a developer to own it.

**The depth is yours. The breadth is ours.**

## Webkernel is Numerimondes' product

Let's be clear (even if we don't always say it loudly):

Webkernel is the core product of Numerimondes.

It is not just an open source framework that happens to exist.

It is the vehicle through which we deliver sovereign, durable systems to organizations that want to own their technology.

- We build the platform.
- We (and trusted partners) deploy and operate instances.
- We incubate the ecosystem of modules.
- We transfer real capability instead of creating dependency.

It is a deliberate bet on independence.

## How developers actually fit in

Webkernel is a tool for developers, yes.

But developers do not create "Webkernel applications" from scratch.

They create **packages** of these types only:

- `webkernel-business-module`
- `webkernel-business-module-feature`
- `webkernel-platform-module`
- `webkernel-platform-module-feature`

Everything else (core engine, components, installer, access control, dynamic DB config, etc.) is provided by the platform.

This constraint is intentional. It keeps the system coherent and makes it possible for agents (AI or human) to generate valid modules reliably.

## Principles we care about

- Sovereign first (you host it, you own it)
- Modules as the only extension unit
- Runtime configuration injection instead of editing core config files
- No generated policy files, no "php artisan make:policy" after adding a module
- One set of migrations that work on SQLite (dev), MySQL, PostgreSQL (prod)
- Simple enough for non-technical teams to stand up real functionality
- Explicit contracts so that an agent can build a module from a description

---

This is the spirit. Everything else (code structure, package taxonomy, documentation) should flow from these intentions.

**Webkernel is becoming real.** We are not pretending it is finished.
