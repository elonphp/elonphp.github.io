---
title: "Permission Mechanism: Four-Segment Naming, Role Design, and Prefix Decoupling"
date: 2026-05-25T16:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "Spatie", "Permissions", "ACL"]
categories: ["OCAdmin"]
weight: 5
summary: "OCAdmin's permission mechanism is built on top of Spatie Permission, with a few key design choices layered on top: four-segment naming ({portal}.{module}.{resource}.{action}), using Gate::before to grant super_admin system-wide highest privilege, and a prefix injection mechanism that keeps seeders free of hardcoded prefixes. This post explains the rationale behind each decision, why resources are singular, and why enforcement lives at the route middleware layer."
build:
  list: local
---

> [→ 繁體中文版](/ocadmin/permissions/)

The previous posts covered [overall architecture](/ocadmin/en/architecture/) and [the multilingual mechanism](/ocadmin/en/multilingual/). This one is about **permissions** — how OCAdmin names roles and permissions, how it checks them, and why it's designed this way.

## 1. Don't reinvent the wheel: build on Spatie Permission

[`spatie/laravel-permission`](https://spatie.be/docs/laravel-permission) is one of the most mature permission packages in the Laravel ecosystem, covering roles, permissions, wildcards, cache, and policies. **OCAdmin didn't rewrite any of it** — it just uses Spatie directly, and spends its effort on the "how to use it" conventions.

| What Spatie provides | What OCAdmin layers on top |
|---|---|
| DB schema and ORM for roles / permissions | Four-segment naming convention |
| `$user->can()` / `Gate` integration | Portal prefix isolation |
| `*` wildcard support | Two global roles: `super_admin` (granted via Gate::before) + `system` (audit marker) |
| Built-in permission-definition cache | Custom role-combination cache (for 0 DB queries per request) |
| `HasRoles` trait | Helpers like `hasPortalRole()`, `permName()`, `roleName()` |

Derivative projects don't need to learn a new permission system — they can keep using Spatie knowledge wherever it applies, and only learn the few conventions on top.

## 2. Four-segment naming: `{portal}.{module}.{resource}.{action}`

Permission names always have four segments, all lowercase, snake_case:

```
admin.catalog.product.access     ← in the admin Portal, the catalog module's "access" action on the product resource
admin.catalog.product.modify     ← "modify"
admin.catalog.product.delete     ← "delete"
hrm.mss.employee.access          ← in the HR Portal, the MSS module's "access" on employee
hrm.ess.profile.modify           ← in the HR Portal, the ESS module's "modify" on profile
```

### What each segment represents

| Segment | Maps to | Examples |
|---|---|---|
| `portal` | The application entry point (the [Portal concept](/ocadmin/en/architecture/) covered in the previous post) | `admin`, `hrm`, `pos` |
| `module` | The functional module (a business grouping inside a Portal) | `catalog`, `sale`, `system_acl`, `mss`, `ess` |
| `resource` | The resource type | `product`, `order`, `user`, `role`, `employee` |
| `action` | The operation verb | `access`, `modify`, `delete`, `approve`, `export` |

### Why exactly 4 segments?

**3 is too few**: without `portal`, permissions collide across Portals (admin's `user` vs the HR Portal's `user`). Cross-Portal coexistence is the norm in practice.

**5 is too many**: slicing finer makes names harder to remember and harder to align with the namespace. Slicing down to "product / order / employee" is already the right granularity; going further into "product.option / product.image" sub-resources typically just splits within the same controller's scope.

**4 segments line up cleanly with the system's hierarchy**:

```
URL:         /admin/catalog/products/{id}
             │      │       │
             ▼      ▼       ▼
Permission:  admin.catalog.product.access
             │      │       │       │
Module dir:  Modules/Catalog/Product/
                     │       │
             (portal = the whole Portal)
```

Four segments map onto the system's layers — the **permission name is itself a map**. Looking at `admin.catalog.product.access`, you immediately know: which Portal, which folder, which Model, which action.

### Why singular instead of plural for resource?

This is the choice that gets questioned the most — RESTful URLs are plural (`/products`, `/users`), and industry IAM systems often use plural too (GCP IAM's `compute.instances.get`, Spatie's own docs using `edit articles`). OCAdmin deliberately chose **singular**, for three main reasons:

**1. Alignment with Eloquent Model naming**

Laravel Models are singular (`Product`, not `Products`). Permissions following the same convention keeps naming consistent across layers:

```
Model:       App\Models\Product
Policy:      ProductPolicy
Permission:  admin.catalog.product.access     ← read the permission, immediately map it to the Model
Route URL:   /admin/catalog/products          ← URLs stay RESTful plural — no conflict
```

URL uses RESTful plural; permissions use singular. The two conventions live side by side without interfering with each other.

**2. English irregular plurals are annoying**

`category → categories`, `taxonomy → taxonomies`, `child → children`, `analysis → analyses`, `status → statuses`… Sticking with singular means you never have to think about plural rules.

**3. Cleaner wildcard semantics**

```
admin.catalog.product.*    reads as "all actions on the product resource"     ← intuitive
admin.catalog.products.*   reads as "all actions on the products collection"? ← ambiguous
```

The singular form's `*` naturally points to "all actions on this resource type," without the "set vs individual" ambiguity.

## 3. Wildcard permissions: use Spatie's native `*`

With `config/permission.php`'s `enable_wildcard_permission => true`, you can assign permissions like this:

```php
// HR manager: full access to all three HR modules — MSS / Team / ESS
$hrManager->givePermissionTo([
    'hrm.mss.*.*',     // every resource and action in the MSS module
    'hrm.team.*.*',
    'hrm.ess.*.*',
]);

// Regular employee: limited to ESS personal operations
$employee->givePermissionTo([
    'hrm.ess.profile.access',
    'hrm.ess.profile.modify',
    'hrm.ess.attendance.modify',
    'hrm.ess.leave.modify',
]);
```

`*` can appear in any segment:

```
admin.catalog.product.*    # all actions on product
admin.catalog.*.*          # everything in the catalog module
admin.*.*.*                # the entire admin Portal
```

> Don't invent custom permissions like `admin.all`. Spatie's native `*` already covers that case — adding a custom layer just adds maintenance cost.

## 4. Two global roles: `super_admin` and `system`

Business roles (like `admin.operator` or `hrm.hr_manager`) all carry a Portal prefix. There are also two **global roles** that are exceptions — no prefix — with purposes different from regular business roles:

| Role | Purpose | Visible in admin UI | How it's assigned |
|---|---|---|---|
| `super_admin` | System-wide highest privilege | Yes | Assigned via admin UI |
| `system` | Marker for automated system actions (cron / queue / service account) | No | DB / Seeder |

### `super_admin`: granting system-wide highest privilege via `Gate::before`

OCAdmin uses Laravel's built-in `Gate::before` mechanism to handle highest privilege:

```php
// AppServiceProvider::boot()
Gate::before(fn ($user, $ability) =>
    $user->hasRole('super_admin') ? true : null
);
```

`Gate::before` is Laravel's permission "pre-interceptor" — before any `$user->can(...)`, `@can(...)`, or `$this->authorize(...)` reaches the normal check, this callback runs first. **Returning `true` lets the request through immediately; returning `null` falls through to the normal permission check**.

#### Why use `Gate::before` instead of "assign every permission to super_admin"?

The obvious approach is to create a `super_admin` role and grant it every permission. Workable, but it has drawbacks:

- **You have to remember to add new permissions**: every time a new permission is added, `super_admin` has to be re-synced. Miss it once and you get "the highest-privileged role still returns 403"
- **Permissions can be accidentally edited**: every permission shows up on the admin's role management page and can be accidentally unchecked
- **Initialization cost**: when there are lots of permissions, you have to loop through and attach all of them — extra IO for no reason

`Gate::before` solves it once and for all: **the `super_admin` role doesn't need any permissions attached**, and the Gate layer lets everything through. New permissions don't need to be synced, the role can't be accidentally edited in the UI, and initialization needs no loop.

> The initial `super_admin` account is created by a seeder; from there, the `super_admin` role can be assigned to other accounts via the admin UI.

### `system`: for accounts that aren't human

`system` isn't a role for "humans" — it's for accounts that aren't human: system services, cron jobs, queue workers. When these automated programs leave entries in the audit log, they carry the `system` role, so when reviewing the log later you can immediately tell "this was the system, not some admin."

```php
// e.g. a service account running a batch update
$serviceUser = User::where('username', 'service')->first();
// it has the `system` role, so its operations get the system marker in the log
```

This is cleaner than "add an `is_system` column to the log" — the permission system is already tracking "who did what," so it can serve as the role marker too.

## 5. Prefix decoupling: `permName()` / `roleName()` helpers

The `admin` portal segment in permission names is **never written into code directly**. All composition goes through two helpers:

```php
function permName(string $suffix): string
{
    $prefix = config('portals.ocadmin.permission_prefix', 'admin');
    return "{$prefix}.{$suffix}";
}

function roleName(string $name, bool $isException = false): string
{
    if ($isException) return $name;   // super_admin / system: no prefix
    $prefix = config('portals.ocadmin.role_prefix', 'admin');
    return "{$prefix}.{$name}";
}
```

Usage:

```php
permName('catalog.product.access')     // → 'admin.catalog.product.access'
roleName('order_operator')             // → 'admin.order_operator'
roleName('super_admin', true)          // → 'super_admin' (exception, no prefix)
```

### Inside seeders: pure business tokens

The key benefit: **seeders' array keys hold only the trailing three "business token" segments**, with no prefix:

```php
$permissions = [
    'catalog.product.access' => 'Product access',    // ← pure business token
    'catalog.product.modify' => 'Product modify',
    'catalog.option.access'  => 'Option access',
];

foreach ($permissions as $suffix => $name) {
    Permission::updateOrCreate(['name' => permName($suffix)]);
    //                                    ↑ prefix composed at runtime
}
```

### Why is this decoupling so important?

**Derivative projects can swap the entire permission namespace by changing `.env`**:

```env
# Project A's .env
OCADMIN_PERMISSION_PREFIX=backend
# → permName('catalog.product.access') becomes backend.catalog.product.access

# Project B's .env
OCADMIN_PERMISSION_PREFIX=manage
# → permName('catalog.product.access') becomes manage.catalog.product.access
```

The seeder's business-token list stays **completely identical** across projects — no copy-and-rename needed. Every place that reads the prefix (middleware, Blade `@can`, Policy) goes through the helper; nothing hardcodes `'admin.'` anywhere.

If a seeder wrote out `'admin.catalog.product.access'` directly, reusing it across projects would mean a sed-replace across all files — painful and error-prone. **Pulling the prefix out and composing it at runtime is a prerequisite for portable seeders.**

This echoes the previous [overall architecture](/ocadmin/en/architecture/) post's point about "`url_slug`, `role_prefix`, `permission_prefix`, and `dir` being deliberately decoupled" — they're all the same idea: **let configuration be changed independently, without cascading consequences**.

## 6. Centralizing permission checks at the route middleware

Permission enforcement is deliberately **concentrated at the route middleware**, not at the controller layer.

```php
// ✓ Recommended: declared once at the route layer
Route::middleware(['permission:' . permName('catalog.product.access')])
    ->group(function () {
        Route::get('/products', [ProductController::class, 'index']);
        Route::get('/products/{product}/edit', [ProductController::class, 'edit']);
    });

// ✗ Not recommended: scattered across controllers
public function index()
{
    if (!auth()->user()->can('admin.catalog.product.access')) {
        abort(403);
    }
    // ...
}
```

### Why route, not controller

| Benefit | Centralized declaration at the route layer |
|---|---|
| **Auditability** | `php artisan route:list` shows at a glance what permission protects each route |
| **Detecting missed locks** | A missing lock stands out in code review |
| **Doesn't pollute controllers** | Controllers focus on business logic, free of permission checks |
| **Changing the permission doesn't change the controller** | One edit at the route layer is enough |

The controller's `$this->authorize()` is reserved for **field-level** permissions — for example, an employee profile page displays salary, but only some roles get to see the `salary` field:

```php
public function show(Employee $employee)
{
    $data = $employee->toArray();

    if (!auth()->user()->can(permName('mss.payroll.access:salary'))) {
        unset($data['salary']);
    }

    return view(...);
}
```

Appending `:column` to the action segment marks a field-level permission. Route middleware locks the coarse-grained "can you enter this page" at the door; controllers handle the finer field-level details inside.

## What's next

Upcoming posts:

- **List page conventions**: consistent design for filtering, sorting, pagination, and URL parameter preservation
- **Dual-mode architecture**: trade-offs between Ocadmin doubling as the public site vs. running an independent Web Portal

The advanced permission topics (Spatie's built-in cache limitations, the custom role-combination cache design, integrating Policies with data scoping) carry enough detail to deserve their own post — whether to split them out depends on reader feedback. If you want to read about any of those, let me know.
