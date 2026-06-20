# webkernel/component-access-control

> Sovereign access control for Webkernel. Privilege hierarchy, business scoping, dynamic RBAC, and module permission discovery — wired into the platform without a single generated policy file.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Privilege Layer](#privilege-layer)
- [Business Layer](#business-layer)
- [RBAC Layer](#rbac-layer)
- [Module Contract](#module-contract)
- [Permission Discovery](#permission-discovery)
- [Dynamic Policy](#dynamic-policy)
- [Multi-Tenancy Inside Modules](#multi-tenancy-inside-modules)
- [Filament UI](#filament-ui)
- [Installation](#installation)
- [Configuration](#configuration)
- [API Reference](#api-reference)
- [Module Developer Guide](#module-developer-guide)

---

## Overview

Most access control systems are built for monolithic apps with a fixed list of resources known at bundle time. Webkernel is a modular platform where modules are installed at runtime — each one potentially scoped to a different business, with its own roles and permissions.

`component-access-control` is the Webkernel answer to that problem. It is a purpose-built access layer with three distinct concerns:

| Layer         | What it answers                                            |
| ------------- | ---------------------------------------------------------- |
| **Privilege** | Who can administer the platform itself?                    |
| **Business**  | What is this module scoped to?                             |
| **RBAC**      | What can this user do inside this module in this business? |

These three layers are orthogonal. A `SUPER_ADMIN` privilege does not automatically grant module-level permissions — it grants the right to _assign_ those permissions. A module developer does not need to know which business they are running in — the platform injects that context.

### What this component does NOT do

- It does not generate policy files. Ever.
- It does not require run any command after adding a module.
- It does not hardcode business or tenant IDs in module code.
- It does not manage authentication — that is Filament's responsibility.

This component does not YET implement or manage auth providers (local, ldap, saml, oauth2 ...etc)— it only manages what authenticated users are allowed to do.

---

## Architecture

```
component-access-control
│
├── Privilege Layer
│   ├── UserPrivilegeLevel (enum)     — APP_OWNER › SUPER_ADMIN › SYSADMIN › STAFF
│   ├── UserOrigin (enum)             — INTERNAL | EXTERNAL
│   ├── UserPrivilege (model)         — pivot: user ↔ privilege level + granted_by
│   └── HasPrivilegeLevel (trait)     — on User model
│
├── Business Layer
│   ├── Business (model)              — a real legal entity or project
│   │                                   separate from the Filament tenant model,
│   │                                   which points to it via a relationship
│   ├── BusinessModule (pivot)        — which modules are active for a business
│   └── HasBusinessAccess (trait)     — on User model
│
├── RBAC Layer
│   ├── AccessControl (facade)        — central API for all permission operations
│   ├── ModulePermissionRegistry      — collects declarations from all modules
│   ├── DynamicPolicy                 — single universal policy, no files generated
│   └── spatie/laravel-permission     — storage: roles, permissions, model_has_roles
│
└── Filament UI
    ├── System panel pages            — Privileges, Businesses, Role/Permission management
    └── Module-level resources        — each module can expose its own role/permission UI
                                        (visibility controlled by the global delegation config)
```

### Dependency graph

```
component-access-control
    ├── webkernel/component-plugin      (Filament plugin traits)
    ├── webkernel/standard-lifecycle    (installer contract)
    ├── spatie/laravel-permission       (RBAC storage)
    └── filament/filament               (UI + auth)
```

---

## Privilege Layer

Privileges govern **platform-level** access only. They answer: who can manage the Webkernel instance itself — its settings, its modules, its businesses, its billing.

They are **not** roles. They do not grant permissions inside modules. They grant the right to delegate, administer, and configure.

The key idea: `APP_OWNER` decides who can delegate roles. An external provider (a digital services company managing a business on behalf of its client, for example) can receive a privilege that lets them manage that business — without being an internal platform administrator.

### Hierarchy

```
APP_OWNER      (rank 60)  — one per instance, internal only
SUPER_ADMIN    (rank 50)  — full platform administration
SYSADMIN       (rank 40)  — technical/infrastructure access
STAFF          (rank 30)  — standard authenticated platform team member
```

Each level has an `EXTERNAL_*` variant (same rank, different origin):

```
EXTERNAL_SUPER_ADMIN  (rank 50)
EXTERNAL_SYSADMIN     (rank 40)
EXTERNAL_STAFF        (rank 30)
```

### What privileges control

| Privilege     | Can do                                                                               |
| ------------- | ------------------------------------------------------------------------------------ |
| `APP_OWNER`   | Everything. Assign privileges to anyone. Designate Business Managers.                |
| `SUPER_ADMIN` | Manage all businesses and all modules. Assign roles within any business.             |
| `SYSADMIN`    | Technical configuration, module activation. Cannot assign roles.                     |
| `STAFF`       | Access only what their RBAC roles explicitly grant inside their assigned businesses. |

### Usage

```php
// Check privilege level
$user->isAppOwner();
$user->isSuperAdmin();
$user->hasPrivilege(UserPrivilegeLevel::SYSADMIN);
$user->hasPrivilegeOrAbove(UserPrivilegeLevel::STAFF);

// Grant a privilege
$user->grantPrivilege(
    level:     UserPrivilegeLevel::SUPER_ADMIN,
    grantedBy: $appOwner,
    origin:    UserOrigin::INTERNAL,
);

// Bootstrap first APP_OWNER (installer only)
$user->bootstrapAsAppOwner();

// Revoke
$user->revokePrivilege();
```

### Delegation rules

Only `APP_OWNER` can grant `APP_OWNER`. A `SUPER_ADMIN` can grant up to `SUPER_ADMIN`. A user cannot grant a privilege above their own level. These rules are enforced at the service layer, not just in comments.

---

## Business Layer

A **Business** is a real-world entity — a legal company (SARL, SAS, Ltd), a government department, an NGO, a project. It is the primary scoping unit for modules and permissions.

**A Business is a separate model from the Filament tenant.** The Filament tenant model used in your panel points to a Business via a relationship. This means the platform can use Filament's native tenancy routing and switching UI, while the Business model remains a clean domain object that the rest of Webkernel (modules, RBAC, API) references directly.

```php
// Example: your Filament tenant model pointing to Business
class Team extends Model
{
    public function business(): BelongsTo
    {
        return $this->belongsTo(Business::class);
    }
}

// In your panel configuration
->tenant(Team::class)
```

Module developers never reference a business ID directly. They use the `AccessControl` API, and the platform resolves the current business from context (Filament tenant, route parameter, or session).

### Business model

```php
$business->id;           // UUID
$business->name;         // "Acme Morocco SARL"
$business->type;         // BusinessType enum: COMPANY | PROJECT | DEPARTMENT | ...
$business->slug;         // "acme-morocco"
$business->is_active;

// Relations
$business->modules();    // active modules for this business
$business->users();      // users who have access to this business
```

### Scoping modules to a business

When a module declares itself as a `business_module`, all its RBAC permissions are automatically namespaced to the business context:

```
{module}::{permission}@business:{business_id}
```

When a module is a `platform_module`, permissions are not business-scoped:

```
{module}::{permission}@platform
```

### Business Managers

The `APP_OWNER` designates **Business Managers** — users (including external ones) who can administer a specific business (activate modules, assign roles) without having platform-level `SUPER_ADMIN` privilege.

```php
AccessControl::designateBusinessManager($user, $business, grantedBy: $appOwner);
AccessControl::revokeBusinessManager($user, $business);
AccessControl::isBusinessManager($user, $business); // bool
```

---

## RBAC Layer

The RBAC layer handles **module-level permissions** — what a given user can do inside a specific module in a specific business.

It is built on `spatie/laravel-permission` for storage, but the entire resolution layer is dynamic. No policy files are ever generated.

### Key principle

```
Privilege layer  →  controls who can ADMINISTER and DELEGATE
RBAC layer       →  controls who can ACT
```

A `STAFF` user with an `admin` role inside the `payroll` module of `business:acme` can do everything that role grants — but cannot touch the platform settings. A `SUPER_ADMIN` can configure everything — but does not automatically get `payroll::Export` unless their role in that business grants it.

### Permission keys

```
{module_slug}::{PermissionName}@business:{business_uuid}
payroll::viewAny@business:01J...
payroll::Export@business:01J...
crm::ViewContactInfo@business:01J...

# Platform modules (not business-scoped)
platform-logs::view@platform
```

### Roles

Roles are always scoped to a `(module, business)` pair. A user can have different roles in different businesses:

```php
// User is "manager" in payroll for business A, "viewer" in payroll for business B
AccessControl::assignRole($user, 'manager', module: 'payroll', business: $businessA);
AccessControl::assignRole($user, 'viewer',  module: 'payroll', business: $businessB);
```

### Checking permissions

```php
// Standard Laravel gate (works everywhere)
$user->can('payroll::Export', $business);

// AccessControl facade
AccessControl::check($user, 'Export', module: 'payroll', business: $business);

// In a Filament resource (via trait)
static::can('Export');                 // resource context
$this->can('editGlobalSettings');      // page context
```

### Super-admin bypass

If the user has `APP_OWNER` or `SUPER_ADMIN` privilege, all permission checks return `true` automatically before reaching Spatie. Configurable per module if a module wants to restrict even super-admins (e.g. sensitive financial data modules).

```php
AccessControl::forModule('payroll')
    ->bypassPrivilegeCheck(false)  // even SUPER_ADMIN must have explicit role
    ->permissions([...])
    ->register();
```

---

## Module Contract

Every Webkernel module that participates in access control implements the `WebkernelModule` contract. This is the only thing a module developer needs to do.

```php
interface WebkernelModule
{
    /**
     * Unique module slug. Used as permission namespace.
     * Example: 'payroll', 'crm', 'platform-logs'
     */
    public static function moduleSlug(): string;

    /**
     * Is this module scoped to a Business?
     * false = platform module (permissions are @platform-scoped)
     * true  = business module (permissions are @business:{id}-scoped)
     */
    public static function isBusinessModule(): bool;

    /**
     * Permission names this module declares.
     * Standard CRUD names + any custom action names.
     *
     * @return array<string>
     */
    public static function permissions(): array;

    /**
     * Optional: does this module support internal multi-tenancy
     * (its own organisations, sub-customers, etc.)?
     */
    public static function supportsMultiTenancy(): bool;

    /**
     * Optional: default roles this module ships with.
     * These are created automatically when the module is activated for a business.
     *
     * @return array<string, array<string>>  role => [permissions]
     */
    public static function defaultRoles(): array;
}
```

### Example: Payroll module

```php
class PayrollModule implements WebkernelModule
{
    public static function moduleSlug(): string
    {
        return 'payroll';
    }

    public static function isBusinessModule(): bool
    {
        return true;
    }

    public static function permissions(): array
    {
        return [
            // Standard CRUD
            'viewAny', 'view', 'create', 'update', 'delete',
            // Custom actions
            'Export',
            'ExportFinancialData',
            'ApprovePayslip',
            'LockPayrollPeriod',
        ];
    }

    public static function supportsMultiTenancy(): bool
    {
        return false;
    }

    public static function defaultRoles(): array
    {
        return [
            'payroll-admin'   => ['viewAny', 'view', 'create', 'update', 'delete', 'Export', 'ExportFinancialData', 'ApprovePayslip', 'LockPayrollPeriod'],
            'payroll-manager' => ['viewAny', 'view', 'create', 'update', 'ApprovePayslip'],
            'payroll-viewer'  => ['viewAny', 'view'],
        ];
    }
}
```

### Registration in ServiceProvider

```php
class PayrollServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        AccessControl::register(PayrollModule::class);
    }
}
```

That's all. No policy files. The platform handles everything else.

---

## Permission Discovery

The `ModulePermissionRegistry` collects all module declarations as ServiceProviders boot. It runs during the Laravel boot cycle, not at compile time.

### How discovery works

```
1. Each module ServiceProvider calls AccessControl::register(ModuleClass::class)
2. ModulePermissionRegistry stores the declaration (slug, permissions, scope, roles)
3. On first access (lazy), the registry syncs declared permissions to the database
   via spatie/laravel-permission — only adding new ones, never removing existing ones
4. When a module is activated for a Business, its defaultRoles() are created
   scoped to that (module, business) pair
5. DynamicPolicy resolves all Gate calls at runtime from this registry
```

### Sync strategy

Permissions are **additive only** during normal operation. Removing a permission from the module declaration does not delete it from the database — a separate explicit command is required. This prevents accidental data loss when a module is updated.

```bash
php artisan access-control:sync          # sync all registered modules
php artisan access-control:sync payroll  # sync a specific module
php artisan access-control:prune payroll # remove permissions no longer declared (destructive, requires --force in production)
```

---

## Dynamic Policy

There is one policy class in the entire system. It handles all models for all modules. No file is ever generated.

```php
class DynamicPolicy
{
    public function __call(string $method, array $args): bool
    {
        /** @var \App\Models\User $user */
        $user = $args[0];
        $subject = $args[1] ?? null;

        // Super-admin bypass
        if ($user->isSuperAdmin()) {
            $module = ModulePermissionRegistry::moduleForModel($subject);
            if ($module === null || $module::bypassPrivilegeCheck()) {
                return true;
            }
        }

        // Resolve the business context from the subject model or current request
        $business = BusinessContextResolver::resolve($subject);

        // Build the permission key
        $permissionKey = ModulePermissionRegistry::buildKey(
            method:   $method,
            subject:  $subject,
            business: $business,
        );

        return $user->hasPermissionTo($permissionKey);
    }
}
```

Registered globally in `AccessControlServiceProvider`:

```php
Gate::guessPolicyNamesUsing(fn() => DynamicPolicy::class);
```

### Business context resolution

The `BusinessContextResolver` finds the current business from (in order):

1. The model's `business_id` attribute (if the model implements `BelongsToBusiness`)
2. The current Filament tenant's associated Business
3. The current request's `business` route parameter
4. The session-stored current business

Module developers never call this directly. It is transparent.

---

## Multi-Tenancy Inside Modules

A module that declares `supportsMultiTenancy(): true` can have its own internal tenant structure — organisations, sub-customers, franchises, etc. This is entirely managed at the module level. The platform provides the scoping context; the module owns the data model.

This applies to both module types:

- A **business module** with multi-tenancy: scoped to a business, and further scoped to an internal organisation within that business.
- A **platform module** with multi-tenancy: not business-scoped, but can still have its own internal tenant structure.

A business module with multi-tenancy enabled gets an additional scope dimension in its permission keys:

```
{module}::{permission}@business:{business_id}:tenant:{tenant_id}
```

The `AccessControl` facade exposes the tenant context:

```php
AccessControl::forModule('crm')
    ->inBusiness()
    ->withTenancy()
    ->permissions([...])
    ->register();
```

The internal tenant is resolved from the current request/session automatically, following the same pattern as the business resolver. Module developers do not call this directly.

---

## Filament UI

### System panel

The system panel is accessible to users with the appropriate privilege level. It is where the `APP_OWNER` and `SUPER_ADMIN` manage the platform globally.

| Page                    | Privilege required                   | Description                                                            |
| ----------------------- | ------------------------------------ | ---------------------------------------------------------------------- |
| `Businesses`            | `STAFF+` (read), `SYSADMIN+` (write) | List, create, and configure businesses                                 |
| `BusinessDetail`        | `SYSADMIN+`                          | Modules active for a business, Business Managers                       |
| `PlatformUsers`         | `SUPER_ADMIN+`                       | Manage platform users and their privilege levels                       |
| `RolePermissionManager` | `SUPER_ADMIN+`                       | Global view of all roles and permissions across modules and businesses |
| `AccessAuditLog`        | `SYSADMIN+`                          | Read-only log of privilege grants and role assignments                 |

### Module-level role/permission UI

Each module can expose its own Filament resource for managing its roles and permissions within a business. **Who can see and use that resource depends entirely on the global delegation configuration** — specifically, whether the current user has been granted the right to manage roles for that module in that business (either as a Business Manager or via an explicit privilege delegation by `APP_OWNER`).

This means:

- A Business Manager designated by `APP_OWNER` can access the role/permission UI of the modules active for their business.
- An external provider granted the right to manage a specific module can access only that module's UI.
- Nothing is hardcoded in the module — the visibility is driven by the privilege and delegation system.

```php
// In a module's Filament resource
use Webkernel\Component\AccessControl\Traits\HasResourcePermissions;

class PayrollResource extends Resource
{
    use HasResourcePermissions;

    public static function getEssentialsPlugin(): ?AccessControlPlugin
    {
        return AccessControlPlugin::get();
    }
}
```

### Plugin registration

```php
->plugins([
    \Webkernel\Component\AccessControl\AccessControlPlugin::make()
        ->navigationGroup('Access Control')
        ->showAuditLog(true)
        ->allowBusinessManagerSelfService(true),
])
```

---

## Installation

```bash
composer require webkernel/component-access-control
```

Publish and run migrations:

```bash
php artisan vendor:publish --tag="access-control-migrations"
php artisan migrate
```

Publish config (optional):

```bash
php artisan vendor:publish --tag="access-control-config"
```

Bootstrap the first `APP_OWNER` (run once, during install):

```bash
php artisan access-control:bootstrap-owner --email=owner@example.com
```

Or programmatically in the Webkernel installer:

```php
$user->bootstrapAsAppOwner();
```

---

## Configuration

```php
// config/access-control.php
return [

    /*
    |--------------------------------------------------------------------------
    | Super-admin bypass
    |--------------------------------------------------------------------------
    | When true, users with SUPER_ADMIN or above bypass all RBAC checks
    | globally. Individual modules can override this via bypassPrivilegeCheck().
    */
    'super_admin_bypass' => true,

    /*
    |--------------------------------------------------------------------------
    | Permission sync strategy
    |--------------------------------------------------------------------------
    | 'boot'    — sync declared permissions to DB on every boot (dev-friendly)
    | 'command' — sync only when access-control:sync is run (production-safe)
    */
    'sync_strategy' => env('ACCESS_CONTROL_SYNC', 'command'),

    /*
    |--------------------------------------------------------------------------
    | Business context resolver
    |--------------------------------------------------------------------------
    | Order in which the business context is resolved for a given request.
    */
    'business_context_resolvers' => [
        \Webkernel\Component\AccessControl\Resolvers\ModelBusinessResolver::class,
        \Webkernel\Component\AccessControl\Resolvers\FilamentTenantResolver::class,
        \Webkernel\Component\AccessControl\Resolvers\RouteParameterResolver::class,
        \Webkernel\Component\AccessControl\Resolvers\SessionBusinessResolver::class,
    ],

    /*
    |--------------------------------------------------------------------------
    | Audit log
    |--------------------------------------------------------------------------
    */
    'audit_log' => [
        'enabled' => true,
        'events'  => [
            'privilege_granted',
            'privilege_revoked',
            'role_assigned',
            'role_revoked',
            'business_manager_designated',
        ],
    ],

    /*
    |--------------------------------------------------------------------------
    | Tables
    |--------------------------------------------------------------------------
    */
    'tables' => [
        'businesses'           => 'wk_businesses',
        'business_modules'     => 'wk_business_modules',
        'user_privileges'      => 'wk_user_privileges',
        'user_business_access' => 'wk_user_business_access',
        'access_audit_log'     => 'wk_access_audit_log',
    ],

];
```

---

## API Reference

### `AccessControl` facade

```php
// Module registration (called in ServiceProvider::boot)
AccessControl::register(PayrollModule::class);

// Role management
AccessControl::assignRole($user, 'payroll-admin', module: 'payroll', business: $business);
AccessControl::revokeRole($user, 'payroll-admin', module: 'payroll', business: $business);
AccessControl::syncRoles($user, ['payroll-admin', 'payroll-viewer'], module: 'payroll', business: $business);
AccessControl::getUserRoles($user, module: 'payroll', business: $business);

// Permission check
AccessControl::check($user, 'Export', module: 'payroll', business: $business); // bool

// Business management
AccessControl::designateBusinessManager($user, $business, grantedBy: $admin);
AccessControl::revokeBusinessManager($user, $business);
AccessControl::isBusinessManager($user, $business); // bool
AccessControl::getBusinessesForUser($user); // Collection<Business>

// Module activation per business
AccessControl::activateModule('payroll', $business);    // creates defaultRoles
AccessControl::deactivateModule('payroll', $business);  // does not delete roles
AccessControl::isModuleActiveForBusiness('payroll', $business); // bool

// Registry introspection
AccessControl::getRegisteredModules();                     // all declared modules
AccessControl::getModulePermissions('payroll');            // declared permissions
AccessControl::getModulePermissions('payroll', $business); // resolved with business scope
```

### `HasPrivilegeLevel` trait (on User)

```php
$user->getPrivilegeLevel(): ?UserPrivilegeLevel
$user->getPrivilegeOrigin(): ?UserOrigin
$user->isAppOwner(): bool
$user->isSuperAdmin(): bool
$user->isExternal(): bool
$user->isInternal(): bool
$user->hasPrivilege(UserPrivilegeLevel $level): bool
$user->hasPrivilegeOrAbove(UserPrivilegeLevel $level): bool
$user->grantPrivilege(UserPrivilegeLevel, ?User $grantedBy, UserOrigin): UserPrivilege
$user->revokePrivilege(): bool
$user->bootstrapAsAppOwner(): UserPrivilege
```

### Module traits (in Filament Resources and Pages)

```php
// In a Filament Resource
use Webkernel\Component\AccessControl\Traits\HasResourcePermissions;

class PayrollResource extends Resource
{
    use HasResourcePermissions;

    public static function getEssentialsPlugin(): ?AccessControlPlugin
    {
        return AccessControlPlugin::get();
    }
}

// In a Filament Page
use Webkernel\Component\AccessControl\Traits\HasPagePermissions;

class PayrollSettingsPage extends Page
{
    use HasPagePermissions;

    public static function getPagePermissions(): array
    {
        return [
            'view'               => 'View payroll settings',
            'editGlobalSettings' => 'Edit global payroll configuration',
            'exportAuditLog'     => 'Export payroll audit log',
        ];
    }
}

// Checking in PHP
if ($this->can('editGlobalSettings')) { ... }            // page
if (PayrollResource::can('ExportFinancialData')) { ... } // resource (static)

// Injecting into child Livewire components
@livewire('payroll-sidebar', ['permissions' => $this->getPermissions()])
```

---

## Module Developer Guide

### Checklist

```
[ ] Implement WebkernelModule on a dedicated class
[ ] Call AccessControl::register(YourModule::class) in your ServiceProvider::boot()
[ ] Add HasResourcePermissions to your Filament Resources
[ ] Add HasPagePermissions to any Filament Page that needs fine-grained control
[ ] Declare defaultRoles() if your module ships with sensible presets
[ ] Do NOT create Policy files — the DynamicPolicy handles everything
[ ] Do NOT reference Business IDs directly — use the AccessControl API
[ ] Do NOT manage authentication — Filament handles that
```

### Permission naming conventions

```
Standard CRUD  : viewAny | view | create | update | delete | deleteAny
                 restore | restoreAny | forceDelete | forceDeleteAny | reorder | replicate
Custom actions : PascalCase noun-verb pairs — Export | ApprovePayslip | LockPeriod | ViewFinancialData
```

Keep custom permission names **module-namespaced in intent** even though the module slug is added automatically. `Export` is fine. `PayrollExport` is redundant.

### Testing module permissions

```php
it('enforces payroll permissions', function () {
    $business = Business::factory()->create();
    $user     = User::factory()->create();

    AccessControl::activateModule('payroll', $business);
    AccessControl::assignRole($user, 'payroll-viewer', module: 'payroll', business: $business);

    expect(AccessControl::check($user, 'view',   module: 'payroll', business: $business))->toBeTrue();
    expect(AccessControl::check($user, 'Export', module: 'payroll', business: $business))->toBeFalse();
});
```

---

## License

EPL-2.0 — see [LICENSE](LICENSE) file.

Part of the [Webkernel](https://webkernelphp.com) platform by [Numerimondes](https://numerimondes.com).
