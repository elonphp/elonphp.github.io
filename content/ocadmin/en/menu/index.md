---
title: "Menu Mechanism: Permission Filtering, Structural Independence, and Two Approaches (Code / DB)"
date: 2026-05-26T10:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "Menu", "Sidebar", "ACL"]
categories: ["OCAdmin"]
weight: 7
summary: "The backend sidebar needs to filter dynamically by user permission. Sounds simple, but it unfolds into two design decisions: should menu permissions share with data permissions, and should the menu tree align with code structure. OCAdmin decouples the first with a separate *.menu action and decouples the second by deliberately excluding the menu tree from five-layer coordinate alignment. It ships two coexisting implementations (Code Driven / DB Driven) switchable via MENU_DRIVER, with the single-file Code Driven (A1) as the sample-project default."
build:
  list: local
---

> [→ 繁體中文版](/ocadmin/menu/)

The backend sidebar can't be hardcoded — users should only see menu items they have permission for. Sounds simple, but it unfolds into two design decisions: **should menu permissions share with data permissions**, and **should the menu tree align with code structure**.

## 1. Design decision 1: the `*.menu` action — menu visibility decoupled from data permission

The intuitive approach is "the user has `product.access`, so show the product menu". This conflation sounds reasonable but gets stuck in practice.

Consider these scenarios:

- An admin can **edit a product directly via URL** (has access), but you **don't want an extra sidebar item adding visual noise**
- A business role has `product.access` (can view), but the menu is deliberately hidden to force entry through a specific workflow
- A new feature has just launched; the permissions are already granted, but you don't want the sidebar to expose it yet

Tying menu visibility to data permission forces you to detour around all of these — e.g. "pretend to revoke access and hack visibility through some other mechanism".

OCAdmin splits menu visibility into a separate `{resource}.menu` action:

| Action | Purpose | Who checks it |
|---|---|---|
| `{resource}.menu` | Whether the menu shows in the sidebar | MenuComposer |
| `{resource}.access` | List + single-item view (basic data permission) | Controller `authorize()` |
| `{resource}.access_all` | View everything (not just `created_by`) | Controller scope |
| `{resource}.modify` / `.delete` | Each operation | Controller `authorize()` |

This follows the four-segment naming from the [permission mechanism](/ocadmin/en/permissions/) — `admin.catalog.product.menu` and `admin.catalog.product.access` are two independent permissions, each assignable separately.

### Why this decoupling

- **Semantic separation**: UI visibility ≠ data permission; two concerns, two flags
- **No implicit logic**: no code that says "access_all implies access" or "access implies menu"; all implication is made explicit by checking the right checkboxes when assigning to a role
- **One checkbox = one concern**: the admin UI stays unambiguous

Sure, 95% of role designs are "has access ⇒ has menu". But **that 5% of exceptions justifies having two columns** — without them, you'd be forced to hack.

## 2. Design decision 2: the menu tree is independent of five-layer coordinate alignment

[Naming conventions](/ocadmin/en/naming/) covered OCAdmin's "five-layer coordinate alignment" — URL prefix, route name, permission, module folder, and view folder should all align for the same resource.

**The menu tree is deliberately excluded.**

### Why

The menu tree is a **UX-layer** concern — its grouping reflects "the user's mental model", which doesn't necessarily match the program's "natural classification".

Concrete example: `Taxonomy` / `Term` are two resources with all five layers aligned (`Core/Controllers/System/TaxonomyController.php` and `TermController.php` are flat siblings under `System/` — no intermediate layer). But the sidebar groups these two under an intermediate "Vocabulary" group:

```
System
├── Access Control     (5 ACL links)          ← Controller also has an Acl/ intermediate layer
├── Vocabulary         (2 links: Taxonomy/Term) ← Controller has no intermediate layer
├── Settings
├── Schema
└── Menu
```

The "Access Control" sidebar group happens to align with the Controller `Acl/` intermediate layer — that's **coincidence**. Both sides independently met their own criteria for adding a layer (Controllers because of "five-layer alignment + multiple siblings"; the menu because of UX grouping needs). The "Vocabulary" group exists only in the sidebar — Controller and URL don't have it; the sidebar just collapses these two into a visual group for tidiness.

Deciding when the sidebar should add an intermediate group is a **pure UX call** (similar links ≥ 2, avoid too many flat items under a parent that hurt scanning). **It does not, and should not, drive Controller / view / URL / permission to add the same intermediate layer.**

### Industry parallels

Decoupling menu tree from underlying structure is the industry norm:

- **GitHub Settings**: sidebar has ~15 groups, but URLs are all flat `/settings/{section}` with no group segment
- **VS Code's left activity bar**: Explorer / Search / Source Control are UI groups that don't map to any internal API namespace
- **Magento Admin**: Marketing / Content / Reports are large groups, but the controllers are spread across various modules

Menu grouping is a UX decision, not a code-structure decision — forcing alignment makes you create unnecessary folder layers just because the sidebar wanted a group.

## 3. Parent items auto-hide: driven by children, no parent permission needed

The menu tree often has pure grouping nodes (like "System", "Catalog") — no corresponding URL, not tied to any single controller. **These parent nodes don't need a permission.**

### Why

Take a "Catalog" group with three children: "Product", "Option", "Specification" — each carrying its own `catalog.product.menu` / `catalog.option.menu` / `catalog.specification.menu`:

- User has any one of the children's menu permissions → "Catalog" **should show**
- User has none → "Catalog" **should hide**

If you defined a separate `catalog.menu` parent permission, it would either need to stay in sync with the children automatically (extra synchronization logic), or be assigned manually by an admin (extra work, easy to forget). **Both are worse than not having a parent permission at all.**

OCAdmin's MenuComposer uses "child-driven" recursive filtering:

```php
protected function filterByPermission(array $item, $user): ?array
{
    // 1. Item itself has a permission, and user doesn't have it → remove
    if (!empty($item['permission']) && !$user->can($item['permission'])) {
        return null;
    }

    // 2. Recursively filter children
    if (!empty($item['children'])) {
        $item['children'] = collect($item['children'])
            ->map(fn ($child) => $this->filterByPermission($child, $user))
            ->filter()
            ->values()
            ->toArray();

        // 3. Pure group node (empty href) + all children filtered out → hide parent
        if (empty($item['children']) && empty($item['href'])) {
            return null;
        }
    }

    return $item;
}
```

The key is step 3: **empty parent `href` + all children filtered out → the whole parent disappears**. The parent doesn't need a permission field, doesn't need a group-level permission, doesn't need any synchronization logic — visibility is purely derived from children.

`super_admin` uses the `Gate::before` pattern from the [permission mechanism](/ocadmin/en/permissions/), so `can()` always returns true, and the whole menu tree shows — no special-casing needed to keep super_admin from being filtered out.

## 4. Two approaches: Code Driven vs DB Driven

There are two choices for "where the menu is defined". The OCAdmin sample project **defaults to the single-file Code Driven style (A1)**; DB Driven is kept as an advanced option you can switch on when you actually need runtime menu editing.

### Approach A — Code Driven (sample-project default)

The menu is defined in PHP files, with structure baked in at compile time. Two sub-variants: A1 (single file, default) and A2 (one file per module, the variant for when you have lots of modules) — A1 first.

#### A1: One file per portal (sample-project default, OpenCart style)

OpenCart 4 puts the entire backend sidebar in a single file: `admin/controller/common/column_left.php` — all menu items in one big array, filtered by permission and output. The OCAdmin sample project follows this style; the whole sidebar is written directly inside `MenuComposer::buildMenus()`:

```php
// app/Portals/Ocadmin/Core/ViewComposers/MenuComposer.php
class MenuComposer
{
    protected function buildMenus(): array
    {
        $tree = [
            [
                'id'       => 'menu-catalog',
                'icon'     => 'fa-solid fa-tags',
                'name'     => 'Catalog',
                'href'     => '',
                'children' => [
                    ['name' => 'Product', 'permission' => permName('catalog.product.menu'),
                     'href' => route('lang.ocadmin.catalog.product.index'), 'children' => []],
                    ['name' => 'Option', 'permission' => permName('catalog.option.menu'),
                     'href' => route('lang.ocadmin.catalog.option.index'), 'children' => []],
                ],
            ],
            // other modules follow the same pattern
        ];

        return collect($tree)
            ->map(fn ($item) => $this->filterByPermission($item, auth()->user()))
            ->filter()
            ->values()
            ->toArray();
    }
}
```

No Menu class, no registration array — the whole sidebar lives in one file. The sample project defaults to A1 for three reasons:

- **Clone-and-run**: no DB dependency; you don't need to migrate / seed before seeing the sidebar
- **Single file is easy to read**: changing the menu is a single PHP file edit; you don't need to learn the `sys_menus` schema
- **Consistent with the OpenCart prototype**: as a system "designed using OpenCart as the blueprint", defaulting to the prototype's style is the most natural choice

#### A2: One Menu class per module (variant for many modules)

Once you cross some threshold (roughly ≥ 10 modules), the single file gets bulky and module PRs start colliding in it. At that point you can split into one Menu class per module:

```php
// app/Portals/Ocadmin/Modules/Finance/FinanceMenu.php
class FinanceMenu
{
    public static function items(): array
    {
        return [
            'id'   => 'menu-finance',
            'icon' => 'fa-solid fa-coins',
            'name' => 'Finance',
            'href' => '',
            'children' => [
                [
                    'name'       => 'Payment',
                    'permission' => permName('finance.payment.menu'),
                    'href'       => route('lang.ocadmin.finance.payment.index'),
                    'children'   => [],
                ],
                // ...
            ],
        ];
    }
}
```

The MenuComposer registers an array of classes and collects them in order:

```php
protected array $menuClasses = [
    CatalogMenu::class,
    InventoryMenu::class,
    MasterMenu::class,
    // ...
];
```

#### A1 vs A2: which to choose

| | A1 (single file, sample-project default) | A2 (one file per module, for many modules) |
|---|---|---|
| Seeing the full sidebar | One file, top to bottom | Flip through multiple classes |
| Adding a module | Insert a block into the big array | Add a class + register one line in Composer |
| Removing a module | Manually clean up the array section (easy to miss) | Remove the module folder; Composer registration goes with it |
| Module-PR changes | Multiple PRs editing the same big file — easy git conflicts | The module's menu change lives inside its own PR |
| Portability | Copy the whole file, easy to drag unwanted items | One module packs up and goes |
| Best for | Few / stable modules (sample-project style, OpenCart style) | Many modules with independent iteration rhythms |

The sample project defaults to A1 because it assumes "easy onboarding and manageable module count" — derived projects can switch to A2 once they grow beyond that threshold. Both variants output the same format and share the same `filterByPermission()`; switching is just rewriting the data source.

### Approach B — DB Driven (advanced: open it when you need runtime editing)

When you need backend UI for menu editing — drag-to-reorder, disabling items, or different menu sets for different deployment instances — store the menu in the DB. It lives in a separate `sys_menus` table, decoupled from the permissions table:

```
sys_menus                            sys_menu_translations
├── id                               ├── menu_id (FK)
├── portal                           ├── locale
├── parent_id                        └── display_name
├── permission_name ──→ links to acl_permissions
├── route_name (dynamic URL)
├── href (external link)
├── icon
├── sort_order
└── is_active
```

Key columns:

| Column | Purpose |
|---|---|
| `portal` | Distinguishes menus across portals (ocadmin / pos / www) on the same shared table |
| `parent_id` | Self-reference; null = L1 root node |
| `permission_name` | Points to `acl_permissions.name`; null = pure group node |
| `route_name` | A system-defined named route (with locale prefix) |
| `href` | External URL; mutually exclusive with `route_name` |
| `is_active` | Soft-retirement (echoes the `is_active` design from [Settings](/ocadmin/en/settings/)) |

**When to switch to B:**
- Backend UI needs menu editing (drag-to-reorder, change icons, disable items)
- Different deployment instances need different menu sets (multi-tenant / SaaS)
- Translation lives in the DB rather than lang files (via `sys_menu_translations`)

### Why default to A1 instead of B

OCAdmin is a **sample project** — meant to be forked as a starting point for new projects, so the default has to honor "clone and run":

- DB Driven requires migrate + seed before the sidebar appears — high onboarding friction
- For most derived projects, once the sidebar is set it stays unchanged for years; runtime editing is a minority need (SaaS, multi-tenant)
- The OpenCart prototype doesn't use DB Driven; defaulting to it best matches this system's positioning

Switching mechanism:

```env
MENU_DRIVER=code       # Default — reads from MenuComposer's hardcoded array
# or
MENU_DRIVER=database   # Switch to the sys_menus table
```

Both MenuComposers fetch from different sources but **share the same output format** and **same permission-filtering logic** (the same `filterByPermission()`) — switching cost is minimal, just an `.env` change. `MenuController` / `MenuTreeController` / `MenuSeeder` are all kept; switch them on when you need runtime menu editing.

This coexistence echoes [the overall architecture](/ocadmin/en/architecture/)'s "derived projects should be able to switch strategies easily" — default for the majority, keep an escape hatch for the minority.

### Why `sys_menus` and `acl_permissions` aren't merged

DB Driven has another key decision: **menu table and permission table stay separate**, joined by `permission_name` as a string-based soft reference.

The intuitive move is to put menu columns (icon, href, sort_order) directly into `acl_permissions`. Looks like it saves a table, but downsides:

- `acl_permissions` gets cluttered with "nothing to do with permissions" UI columns
- Spatie's `hasPermissionTo()` / `can()` would return objects loaded with junk fields
- Pure group nodes (no permission) shoved into the permission table — semantically awkward

Each table does one thing: `acl_permissions` decides "can do", `sys_menus` decides "how to present". The soft reference via `permission_name` is the minimal coupling.

## 5. Multi-portal sharing one menu table

The `sys_menus.portal` column lets multiple portals share one table:

| portal | Application | Purpose |
|---|---|---|
| `ocadmin` | Backend admin | Full sidebar |
| `pos` | POS system | POS function bar |
| `www` | Public site | Frontend nav |

Each portal's MenuComposer queries only its own portal's rows:

```php
Menu::where('portal', 'ocadmin')->whereNull('parent_id')->...
```

Portals are fully independent and don't interfere with each other. The same permission (e.g. `admin.catalog.product.menu`) can be referenced by menu nodes across portals — naturally supported by the "permission is the condition, menu node is the presentation" relationship from §1.

## 6. What's next

Two related topics worth expanding:

- **Role-combination caching**: `super_admin` is already fast thanks to `Gate::before`, but for other roles, how do we get the per-request cost of fetching user permissions down to 0 queries? MenuComposer runs `can()` for every request, so this cache directly affects sidebar render speed.
- **Multi-layer settings overrides**: the "brand-layer / store-layer" hinted at in [Settings](/ocadmin/en/settings/) — how to do this without polluting `sys_settings`.

The advanced topics within menu mechanism itself (frontend sidebar collapse state, active-link highlighting, breadcrumb generation) are mostly view-layer concerns and may not warrant a dedicated post — depends on feedback.
