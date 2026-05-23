---
title: "Overall Architecture: How We Split Portal, Core, and Module — and Why"
date: 2026-05-24T22:30:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "Architecture", "Portal"]
categories: ["OCAdmin"]
weight: 2
summary: "OCAdmin isn't a new framework — it's Laravel + a set of architectural conventions. The key concept is the Portal: letting one project serve admin, HR, in-store, and public websites at once. This post covers three things: why Portals exist, what Portals share and isolate, and why Core and Modules are organized differently inside a Portal."
build:
  list: local
---

> [→ 繁體中文版](/ocadmin/architecture/)

[The first post](/ocadmin/en/introduction/) explained **why OCAdmin borrows OpenCart's admin UI design**. This one explains **how the underlying code is organized** — the design rationale behind the three-layer architecture: Portal, Core, and Modules.

## 1. To be clear: the foundation is Laravel

OCAdmin **is not another framework**. It's just a Laravel project — routing, authentication, ORM, queues, cache — all handled by Laravel.

So what's OCAdmin's own contribution? **A set of Laravel-based architectural conventions**, mainly solving one problem:

> How can a single Laravel project elegantly serve "admin staff," "HR / employees," "in-store cashiers," and "public website visitors" all at once?

The answer is called **Portal**.

## 2. Portal: the abstraction for application entry points

Imagine the same Laravel project as a single building, and **a Portal is one of its entrances**:

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Ocadmin   │  │     Hrm     │  │     Web     │  │     POS     │
│   Portal    │  │   Portal    │  │   Portal    │  │   Portal    │
│             │  │             │  │             │  │             │
│  Internal   │  │ HR / staff  │  │   Public    │  │  In-store   │
│ admin panel │  │self-service │  │  website    │  │   sales     │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
  role: admin.*    role: hrm.*      role: web.*      role: pos.*
```

Each Portal has its own:

- **URL prefix** (`/admin/...`, `/hrm/...`, `/pos/...`)
- **Roles and permissions** (`admin.user.edit`, `hrm.employee.access`, `pos.order.create`…)
- **Tech stack** (admin uses Blade; in-store can use Vue; the public site can pick something else entirely)
- **Login page** (an in-store tablet can bookmark `/pos/login` directly)

Underneath, they all share the same database, the same Models, and the same Services.

### Why Portal, not "multiple Laravel projects"?

"Spinning up a separate Laravel project for each Portal" is technically possible, but in practice it's painful:

| Problem | How Portal architecture solves it | What multi-project would mean |
|---|---|---|
| Sharing Models across business domains | Shared `app/Models/` | Either duplicate, extract a package, or go microservice |
| One user operating across Portals | One `users` table + Portal role checks | Multiple account systems, SSO integration |
| Syncing a single change | One PR | Three repos, three PRs |
| Dev environment | One `php artisan serve` | N servers, port mapping |

Portal architecture lets you **keep the convenience of "one codebase"** while still having "N application entry points" worth of flexibility.

### Need an API? Just spin up another Portal

Portal architecture **doesn't force** every entry point to use the same frontend approach. Each Portal independently decides whether to go Blade SSR or SPA + API.

**Ocadmin Portal chose not to separate frontend from backend** (it uses Blade + jQuery + Bootstrap 5), for these reasons:

- The admin panel is internal — no SEO need
- No Mobile App that would share an API
- No separate frontend team to develop in parallel
- Form- and table-heavy, desktop-first — traditional SSR is the most efficient; the extra SPA abstraction would just add complexity

But **other Portals can take the exact opposite approach**. When you need a Mobile App, third-party integration, or versioned APIs, you just open a new Portal:

```
app/Portals/
├── Ocadmin/         ← Blade + jQuery + Bootstrap (admin, traditional SSR)
├── WebApi/          ← Pure RESTful API (Mobile / 3rd party)
├── WebApiV1/        ← Versioned API: v1 in maintenance
└── WebApiV2/        ← Versioned API: v2 — new features, parallel to v1
```

Every Portal shares `app/Models/` and `app/Services/`; the difference is only at the "routes / views / middleware / auth strategy" layer. **Opening a new API Portal doesn't require refactoring any existing Portal** — add a folder, register a ServiceProvider, done.

That's the real value of Portal architecture: **"this entry point needs SSR" and "that entry point needs an API" stop being a binary choice**. It also makes API version control (v1 / v2 living side by side) fall naturally into the folder structure, instead of being hacked into route prefixes.

## 3. What Portals share, and what they isolate

The design principle: **share the backend, isolate the frontend**.

| Resource | Shared | Why |
|---|---|---|
| `app/Models/` | ✓ | A business entity should have exactly one definition (User, Order, Product…) |
| `app/Services/` | ✓ | Cross-Portal business logic ("create order" might be triggered from admin / in-store / public site) |
| `app/Helpers/` | ✓ | Utility classes have no Portal affinity |
| `database/migrations/` | ✓ | There's only one database |
| `lang/` | ✓ | Translation files centrally managed |
| Blade Views | ✗ | Each Portal is fully independent |
| Frontend assets (JS / CSS) | ✗ | Each Portal has its own Vite bundle |
| Route files | ✗ | Each Portal has its own `routes/*.php` |

The direct benefit of this split: **the admin's Bootstrap 5 and the in-store Vue don't have CSS class collisions, but both sides can still call `User::find($id)`**.

## 4. config/portals.php: a decoupled configuration

Each Portal is registered in `config/portals.php`:

```php
return [
    'ocadmin' => [
        'url_slug'          => 'admin',          // → /{locale}/admin/...
        'role_prefix'       => 'admin',
        'permission_prefix' => 'admin',
        'dir'               => 'Ocadmin',
    ],
    'pos' => [
        'url_slug'          => 'pos',            // → /{locale}/pos/...
        'role_prefix'       => 'pos',
        'permission_prefix' => 'pos',
        'dir'               => 'Pos',
    ],
    'web' => [
        'url_slug'          => '',               // empty string → /{locale}/... (root, no Portal path segment)
        'role_prefix'       => 'web',
        'permission_prefix' => 'web',
        'dir'               => 'Web',
    ],
];
```

> `permission_prefix` is normally set to the same value as `role_prefix`. The separate field exists to preserve flexibility — in rare scenarios, you might want the "role naming for entering a Portal" and the "authorization string prefix" to differ. In 90% of cases, they're identical.

**`url_slug`, `role_prefix`, and `dir` are deliberately fully decoupled**. It looks unnecessary, but in practice it saves your life:

```php
// Want to change /admin to /backend?
// Only touch url_slug — leave role_prefix alone → existing authorizations are untouched
'ocadmin' => [
    'url_slug'    => 'backend',    // ← only change this
    'role_prefix' => 'admin',
    'dir'         => 'Ocadmin',
],
```

Or something more extreme — a major version migration with old and new admin panels running in parallel:

```php
'ocadmin'    => ['url_slug' => 'ocadmin', 'role_prefix' => 'admin', 'dir' => 'Ocadmin'],
'ocadmin-v2' => ['url_slug' => 'admin',   'role_prefix' => 'admin', 'dir' => 'OcadminV2'],
```

Two Portals **share the same set of roles** (`admin.*`) but map to different code directories and different URLs. If the three were rigidly tied (e.g., "`/admin/` necessarily corresponds to `admin.*` roles and the `Admin/` folder"), this kind of transition wouldn't be possible.

> Decoupling looks like "writing extra for something that might never happen," but in practice every major rename or refactor uses it. Design once, benefit long-term.

## 5. Inside a Portal: two philosophies for Core and Modules

Step inside a Portal (using `app/Portals/Ocadmin/` as an example) and you'll see two top-level folders:

```
app/Portals/Ocadmin/
├── Core/                    ← Standard supplies that ship with the template
│   ├── Controllers/
│   ├── Services/
│   ├── Providers/
│   └── ViewComposers/
│
└── Modules/                 ← Project-specific business modules
    ├── Catalog/
    │   └── Product/
    │       ├── ProductController.php
    │       └── ProductService.php
    ├── Member/
    └── Hrm/
```

**These two folders are organized in completely different ways** — `Core/` is "grouped by layer" (Controllers/, Services/, Providers/...), while `Modules/` is "grouped by feature" (one folder per business feature, containing files from every layer).

This is deliberate. Here's why:

| Aspect | Core/ | Modules/ |
|---|---|---|
| **Content** | Template-bundled features: auth, permissions, system management | Project-specific business (products, orders, HR…) |
| **Scale** | Small (limited feature count) | Large (grows indefinitely with business needs) |
| **Change frequency** | Low (stable foundation) | High (continuous feature development) |
| **Typical modification** | Occasionally tweak a Provider or Controller | Develop one feature, touching Controller + Service + Request together |
| **Portability to other projects** | Doesn't happen (Core is a shared base) | Possible (features can be reused) |

### Core/ uses layer-grouped (Laravel standard)

```
Core/
├── Controllers/
├── Services/
├── Providers/
└── ViewComposers/
```

Aligned with Laravel conventions, so newcomers can find files intuitively. The feature count is limited, so there's no "folder explosion" problem.

### Modules/ uses module-grouped (one feature = one folder)

```
Modules/Catalog/Product/
├── ProductController.php
├── ProductService.php
└── Requests/
```

Benefits:

- **One feature = one folder** — finding, deleting, or porting the whole folder is straightforward
- When developing a feature, related Controller / Service / Request are all in one place — no jumping between `Controllers/`, `Services/`, and `Requests/`
- Want to extract a module into a standalone package? Just move the whole folder

If `Modules/` also used layer-grouped, each layer would be cluttered with unrelated files (`Controllers/ProductController.php`, `Controllers/OrderController.php`, `Controllers/EmployeeController.php`…). The developer experience would be poor.

### Why not unify them into one style?

**Because the problems on each side are genuinely different**, there's no need to pay the cost of "looking consistent":

- Force layer-grouped everywhere → `Modules/` loses the benefit of modularization
- Force module-grouped everywhere → `Core/` ends up with 1-2 files per module, making organization more scattered, not less

Back when I worked with standard Laravel's layer-grouped style on business projects, the biggest pain point I personally felt was **having to jump back and forth between folders in different layers just to edit one feature**:

```
Editing Product's update logic touches:
app/Http/Controllers/Catalog/ProductController.php
app/Services/Catalog/ProductService.php
app/Http/Requests/Catalog/ProductUpdateRequest.php
```

Three files scattered across three layers — every switch means going back up to `app/` and digging down again, then back up, then down again. Module-grouped just puts them in one folder:

```
app/Portals/Ocadmin/Modules/Catalog/Product/
├── ProductController.php
├── ProductService.php
└── Requests/ProductUpdateRequest.php
```

The difference is **not just "a little more convenient"** — you no longer have to remember "which `Services/` subfolder this feature's service lives in," because it's right next to the Controller. The cognitive-load difference compounds through dozens of small daily operations.

**"Consistency" isn't the design goal — "fit for purpose" is.** The essence of architectural decisions is "designing for actual conditions," not "tidying things up for the sake of appearance."

## 6. What's next

Upcoming posts will gradually cover:

- **Multilingual design**: how MVCL's `L` works in OCAdmin, lang file organization strategy
- **Permission mechanism**: role / permission naming conventions, Spatie integration, Portal isolation
- **List page conventions**: consistent design for filtering, sorting, pagination, URL preservation
- **Dual-mode architecture**: when Ocadmin doubles as the public site vs. spinning up an independent Web Portal

If you're particularly interested in one of these topics, leave a comment and I'll adjust the writing order.
