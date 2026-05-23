---
title: "Naming Conventions: Singular/Plural (Horizontal) and Cross-Layer Alignment (Vertical)"
date: 2026-05-25T20:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "Naming", "RESTful"]
categories: ["OCAdmin"]
weight: 4
build:
  list: local
summary: "OCAdmin needs names everywhere — URL, route name, permission, Model, Controller, Blade folder, DB table, variables. This post pins down two questions: horizontal (when singular, when plural) and vertical (one resource's coordinate must align across all layers). The focus is on the meta-principle — once you internalize it, you derive the answer instead of looking up a table."
---

> [→ 繁體中文版](/ocadmin/naming/)

OCAdmin needs names everywhere — URL, route name, permission name, Eloquent Model, Controller class, Blade folder, DB table, variables. **When should each be singular, when plural, and do they have to line up across layers?** This post sets the rules in one place.

It looks like a small concern, but **getting it right early and applying it consistently** removes a recurring cognitive tax: every new module no longer needs a fresh "how do I name this?" debate. The long-term payoff is huge.

Two questions to answer:

1. **Horizontal**: across URL, code, DB, and view layers, when does the same concept go singular and when plural?
2. **Vertical**: how should the same resource's "hierarchical coordinate" line up across those layers?

Once these are clear, you derive the answer for new situations from rules instead of relitigating each time.

## 1. The meta-principle: first decide what you're describing

> **Singular = describing a "concept / type / single instance"**
> **Plural = describing a "collection / container / multiple instances"**

To decide whether a name should be singular or plural, ask: **"What is this name describing?"** Reason from the **nature of the object**, not from "which layer it sits in."

### Three contexts

Different contexts describe different kinds of objects:

| Context | Nature of what it describes | Rule | Where it appears |
|---|---|---|---|
| **OO / in code** | Concept, type, namespace, single object | **Singular** | Model / Controller / Service / Policy / Module folder / **View folder** / Permission name resource segment |
| **HTTP / RESTful** | Public address of a resource collection | **Plural** | URL path / Route name resource segment |
| **DB / storage layer** | Container holding multiple records | **Plural** | Table names / Migration filenames |

A collection variable (`$users = User::get()`) is plural because **it holds multiple instances** — the **nature of the object** decides, not "which context it lives in."

## 2. Quick reference table

In practice, 90% of cases match this table:

| Layer | Object | Rule | Example |
|---|---|---|---|
| URL | URL path | **Plural** | `/admin/system/users` |
| URL | Route name resource segment | **Plural** | `system.users.index` |
| Code | Permission name resource segment | **Singular** | `admin.system.setting.access` |
| Code | Eloquent Model | **Singular** | `User`, `Product`, `Order` |
| Code | Controller class | **Singular** | `UserController` |
| Code | Policy / Service | **Singular** | `UserPolicy`, `OrderService` |
| Code | Single-object variable | **Singular** | `$user`, `$product` |
| Code | Collection variable | **Plural** | `$users`, `$products` |
| DB | Table | **Plural** | `users`, `acl_roles` |
| DB | Foreign key | **Singular `_id`** | `user_id`, `product_id` |
| DB | Pivot table | **Two singulars, alphabetical** | `role_user`, `product_tag` |
| View | Blade subfolder | **Singular** (matches Controller) | `catalog/product/index.blade.php` |

> Edge cases: **booleans** (`$is_active`, `$has_permission`) describe state and have no singular/plural question; **uncountable nouns** (`cache`, `data`) follow English convention as-is; **View namespace** (`ocadmin::`, `pos::`) is singular to match the Portal name.

## 3. The most counterintuitive choice: plural URLs vs singular permissions

This is the most counterintuitive part of the convention — for the same `User` concept, the URL is `/users` but the permission is `admin.system.user.access`. It looks inconsistent, but the **two conventions serve different purposes**:

| | URL (plural) | Permission name (singular) |
|---|---|---|
| **Design origin** | RESTful (resource-collection-oriented) | Eloquent / OO (aligned with class naming) |
| **Meaning** | "this path manages the **collection** of `user` resources" | "permission for **the type** `user`" |
| **Industry consensus** | Rails / Django / Laravel's `Route::resource` all default to plural | AWS IAM / GCP IAM mixed, no overwhelming preference; Laravel internals use singular Model names |
| **Irregular-plural risk** | URL has to get English plural right (`taxonomies`, not `taxonomys`) | Always singular — completely sidesteps it |

Once you internalize the split "**plural in URLs / singular inside code**," switching between them no longer requires conscious thought.

### View folder doesn't match the URL — also deliberate

```
URL:         /admin/catalog/products              ← plural
View folder: views/catalog/product/index.blade   ← singular
```

The view folder belongs to the "OO / in-code" context — it describes "the view space for a domain concept," aligned with `Modules/Catalog/Product/ProductController.php` and `app/Models/Catalog/Product.php`. The URL is HTTP's public collection address (plural) — a different dimension.

The "plural in URLs / singular inside code" split applies **consistently** to view folders too.

## 4. Vertical alignment: one resource, five layers, one coordinate

Horizontal (singular/plural) is solved. There's a second question: **the same resource's "hierarchical position" should line up across layers**.

Every resource has a **coordinate** in the system: `{module}/{resource}`. Every layer that refers to this resource uses the same coordinate — no skipping levels, no extra levels, no different words.

### Five-layer coordinate alignment (using `setting` as an example)

```
URL prefix:    /admin/system/settings
Route name:    lang.ocadmin.system.settings.index
Permission:    admin.system.setting.access
Module folder: Core/Controllers/System/SettingController.php
View folder:   views/system/setting/index.blade.php
```

The main token is `system / setting` across all five — the **only differences are case and singular/plural**.

### Why this matters

**Misaligned coordinates = the maintainer has to keep mental lookup tables**:

- See a URL `/admin/orgs/dealers`, hunt to figure out the route name
- See a route name, hunt to figure out the view folder
- See a view folder, hunt to figure out the module folder

When coordinates align, **knowing one location means knowing all of them** — see a URL, immediately infer the controller location, route name, permission, view path. The IDE just navigates.

### The main token cannot change

Case can differ (kebab-case / snake_case / PascalCase), singular/plural can differ (per §2's table), but the **main token's spelling must be identical across layers**.

Compound-word transformations across layers:

| Concept | URL / Route | Permission | Folder | Variable |
|---|---|---|---|---|
| authorization plan | `authorization-plans` | `authorization_plan` | `AuthorizationPlan/` | `$authorization_plan` |
| image manager | `image-manager` | `image_manager` | `ImageManager/` | `$image_manager` |
| custom option name | `custom-option-names` | `custom_option_name` | `CustomOptionName/` | `$custom_option_name` |

✗ Bad: view folder is `common/imgmanager/` but route is `common.image-manager.*` — the main token is concatenated `imgmanager` on one side and hyphenated `image-manager` on the other. The token spelling differs.

✓ Fix: unify to `image-manager` / `ImageManager/` / `image_manager`. Case differs across the five layers, but the token sequence is the same.

## 5. Four common violations

Misaligned coordinates show up in practice almost always as one of these four shapes:

### 5.1 Hierarchy depth mismatch

```
Module folder:  Modules/System/Acl/UserController.php   (3 layers)
Route:          system.users.*                          (2 layers, skips acl)
View:           views/acl/user/                         (2 layers, skips system)
```

Each layer drops a different level — maximum cognitive cost. Fix: either all 3 layers (add `acl`) or all 2 layers (remove `acl`).

### 5.2 Module-segment mismatch (some have it, some don't)

```
Route:          /admin/organizations                (root-level, no module)
Permission:     admin.organization.organization.*  (extra layer of `organization` module)
Module folder:  Modules/Party/Organization/         (Party is the module)
```

The coordinates are `/organization`, `organization/organization`, and `Party/Organization` — three different shapes.

Fix: pick one module name and apply across all five layers. For example, settle on `org` everywhere: URL `org/organizations` / Permission `admin.org.organization.*` / Module folder `Modules/Org/Organization/`.

### 5.3 Main token differs

See the `imgmanager` example above.

### 5.4 Business-name vs technical-name mixing

```
Module folder:  Modules/Party/Organization/   ← Party is a technical namespace
Route:          dealer.members.*              ← dealer is business terminology
```

The same controller is referred to by two different words in different layers — `Party` and `dealer` belong to different mental models.

Fix: pick one. **All-technical**: `Modules/Party/Member/` + route `party.members` + view `party/member/`. **All-business**: `Modules/Dealer/Member/` + route `dealer.members` + view `dealer/member/`.

## 6. A tempting edge case: `Sale` or `Sales`?

`Sale` is the **most tempting** word to waffle on, because in English `sales` isn't just the plural of `sale` — **it's also an independent noun in business contexts**:

| Form | English connotation | What it points to in code |
|---|---|---|
| `a sale` / `the sale` | A single transaction / a single promotion | A single sales record, a promotion entity |
| `sales` | The business sense — the Sales team, sales report, sales pipeline | A business domain / department name |

The intuitive reaction is "Sales module sounds right — it's the sales business, right?" That pull is real, not imagined. But this convention still picks `Sale/`:

1. **"Domain concept" fits the role of a module folder better than "business-area label"**: the folder contains `Order` / `OrderProduct` / `Invoice` — each one is "an entity within the sales domain." The folder name acts as a namespace prefix (`Sale\Order`), and singular reads as "Order under sales (the concept)"; plural reads as "Order under multiple sales" — that's odd
2. **Cross-layer consistency beats per-word English intuition**: if `Sale` makes an exception and becomes `Sales`, should `Inventory` become `Inventories`? Should `Catalog` become `Catalogs`? Should `Org` become `Orgs`? You end up litigating word by word. **Always-singular is the lowest-friction rule**
3. **No overwhelming industry consensus**: Magento uses `Sales`, sure, but Rails / Django / most Laravel applications use singular module names

**Escape hatch**: if `Sale` truly feels too thin, pick a more specific module name and sidestep the question — call it `Order/` (if the module's center of gravity is orders), `Pos/`, `Checkout/`. **Don't pluralize the module folder just to convey "this is a business area"** — that breaks the cross-layer consistency.

## 7. Migration order: code first, URLs last

When an existing project needs to be brought in line with this convention, order matters:

```
1. Rename Module folders (namespaces and use statements follow; IDE auto-refactor)
   ↓
2. Rename View folders (view path strings follow; one grep-replace)
   ↓
3. Rewrite Permission strings + seeders
   ↓
4. Rewrite Route prefix + name
   ↓
5. Rewrite URLs (optional; for public-facing services, plan a 301 redirect for old URLs)
```

URL changes go **last**, because they touch SEO, external links, mobile bookmarks. The first four steps stay inside the codebase, where the impact radius is controllable.

> Controllers that use `routePrefix()` to derive the route prefix dynamically don't need their `route()` calls touched when routes are renamed — the dynamic reverse-lookup just adapts. **Adopting `routePrefix()` is the single biggest cost-reducer for migrations.**

## 8. What's next

Coming next:

- **[Permission mechanism](/ocadmin/en/permissions/)**: four-segment naming, Gate::before, prefix decoupling
- **List page conventions**: consistent design for filtering, sorting, pagination, and URL parameter preservation
- **Dual-mode architecture**: trade-offs between Ocadmin doubling as the public site vs. running an independent Web Portal

Naming conventions look like minutiae, but **getting them right up front** means every future naming question gets derived from rules instead of debated. This is one of the rare "invest once, benefit forever" design decisions.
