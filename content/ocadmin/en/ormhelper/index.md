---
title: "OrmHelper: Replacing Endless `if`s with a Naming Convention"
date: 2026-05-28T10:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "Eloquent", "OrmHelper", "Query"]
categories: ["OCAdmin"]
weight: 9
summary: "OCAdmin has a carefully designed, arguably one-of-a-kind OrmHelper. Its purpose: stop you from writing if(request()->has('xxx')) where(...) all over the place — give it the query string and it auto-scans every filter_xxx / equal_xxx param and applies them to the Eloquent Builder. Paired with the filterData whitelist for field-level access control, list-page queries shrink from 80+ lines to ~15. This post covers OrmHelper's naming convention, the three-step standard flow, ProductController / PermissionController examples, and how it splits labor with Model Scope."
build:
  list: local
---

> [→ 繁體中文版](/ocadmin/ormhelper/)

A Laravel list page often ends up like this:

```php
// The old way: endless ifs
public function index(Request $request)
{
    $query = Order::query();

    if ($request->filled('filter_name')) {
        $query->where('name', 'like', '%' . $request->filter_name . '%');
    }
    if ($request->filled('filter_phone')) {
        $query->where('phone', 'like', '%' . $request->filter_phone . '%');
    }
    if ($request->filled('equal_status')) {
        $query->where('status', $request->equal_status);
    }
    if ($request->filled('filter_email')) {
        $query->where('email', 'like', '%' . $request->filter_email . '%');
    }
    // ... 8 more fields ...

    $sort = $request->get('sort', 'id');
    $order = $request->get('order', 'desc');
    $query->orderBy($sort, $order);

    $perPage = $request->get('per_page', 20);
    $orders = $query->paginate($perPage);
    $orders->withPath(route('orders.index'))->withQueryString();
    // ...
}
```

Every field gets its own `if (request()->filled(...)) where(...)` — a list page with 10 fields easily becomes 100+ lines of boilerplate, repeated in every Controller.

OCAdmin took this boilerplate away with a **carefully designed, arguably one-of-a-kind** OrmHelper. The same list page becomes:

```php
// The OrmHelper way
public function index(Request $request)
{
    $query = Order::query();

    $filter_data = $this->filterData($request, [
        'filter_name', 'filter_phone', 'filter_email', 'equal_status'
    ]);

    OrmHelper::prepare($query, $filter_data);

    $orders = OrmHelper::getResult($query, $filter_data);
    $orders->withPath(route('orders.index'))->withQueryString();
}
```

From ~80 lines down to ~10. **OrmHelper auto-loops over** every `filter_xxx` / `equal_xxx`, auto-handles sort / order, auto-paginates. `filterData`'s second argument is a whitelist of queryable fields (security).

## 1. Naming convention: filter_xxx / equal_xxx

OrmHelper's core is a URL query string naming convention. When it sees these prefixes, it handles them automatically:

| Param prefix | Behavior | Example query string |
|---|---|---|
| `filter_xxx` | Fuzzy match (REGEXP / LIKE) | `?filter_name=apple` → `name LIKE '%apple%'` |
| `equal_xxx` | Exact match | `?equal_status=active` → `status = 'active'` |
| `sort` / `order` | Sort field / direction | `?sort=created_at&order=desc` |
| `page` / `per_page` | Pagination | `?page=2&per_page=50` |
| `search` | Shared search param (custom handling) | `?search=keyword` |

`filter_xxx` also supports operators for precise matching:

| Notation | Behavior |
|---|---|
| `filter_phone=0912` | REGEXP fuzzy match (spaces act as wildcards) |
| `filter_phone==0912345678` | Exact equality |
| `filter_phone=*0912` | Ends with |
| `filter_phone=0912*` | Starts with |
| `filter_phone=<>` | Non-empty (any non-null value) |
| `filter_phone=<>0912` | Not equal |
| `filter_amount=>100` | Greater than |
| `filter_amount=<500` | Less than |

**Learn the convention once and it applies to every list page** — this is the biggest payoff of "convention over configuration".

## 2. The three-step standard flow

OCAdmin's `getList()` always follows three steps:

```php
// 1. filterData whitelist — limit queryable fields
$filter_data = $this->filterData($request, ['filter_name', 'equal_status']);

// 2. Preprocessing: custom sort, special logic (optional)
$filter_data['sort']  = $request->query('sort', 'id');
$filter_data['order'] = $request->query('order', 'desc');

// 3. OrmHelper auto-loops + auto-paginates
OrmHelper::prepare($query, $filter_data);
$result = OrmHelper::getResult($query, $filter_data);
```

Each step does one thing — clean boundaries.

### filterData whitelist: the second argument restricts queryable fields

```php
protected function filterData(Request $request, array $allowedFilters = []): array
{
    return $request->only(array_merge(
        ['search', 'sort', 'order', 'page', 'limit', 'per_page'],  // shared params auto-allowed
        $allowedFilters                                              // extra filter_* / equal_* allowed
    ));
}
```

The second argument explicitly lists **which fields this list page allows filtering on**. Everything else gets dropped.

For example, an `Order` list that only allows `filter_name` / `equal_status`:

```php
$filter_data = $this->filterData($request, ['filter_name', 'equal_status']);
```

Even if a user manually appends `?equal_customer_id=5&filter_is_closed=0` to the URL, **it won't get through**.

**Why this is required**: `OrmHelper::applyFilters()` internally goes "**if the column exists in the table, apply it**". Without the whitelist, a user could query any field (including admin-only ones) — **this is an obvious security hole on external-facing Portals**. The whitelist serves as the security gate at the "external API query surface".

> **Important caveat**: passing an empty array `filterData($request, [])` does **not** mean "allow everything" — it's the **strictest** setting, leaving only the shared params. All `filter_*` / `equal_*` get blocked. Backend internal pages that need fully open access should use `$request->query()` directly with a comment explaining why.

### OrmHelper::prepare runs the loop

`prepare()` is the OrmHelper core:

```php
public static function prepare(EloquentBuilder $query, array &$params = []): void
{
    self::select($query, $params);          // select columns
    self::applyFilters($query, $params);    // apply filter_/equal_
    self::sortOrder($query, $params);       // sort
}
```

`applyFilters()` is just a loop:

```php
foreach ($params as $key => $value) {
    if (!str_starts_with($key, 'filter_') && !str_starts_with($key, 'equal_')) {
        continue;
    }

    $column = preg_replace('/^(filter_|equal_)/', '', $key);

    if (in_array($column, $tableColumns)) {
        // Main-table column: apply directly
        self::filterOrEqualColumn($query, $key, $value);
    } elseif (in_array($column, $translatedAttributes)) {
        // Translation column: collect for whereHas
        $translationFilters[$key] = $value;
    }
}
```

**No more writing an `if` per field** — whatever `filter_xxx` comes in, it scans all columns, finds the match, applies it to the query.

## 3. ProductController example: the basic three-step flow

`app\Portals\Ocadmin\Modules\Catalog\Product\ProductController.php`:

```php
protected function getList(Request $request): string
{
    $query = Product::with('translations');
    $filter_data = $this->filterData($request, [
        'filter_name', 'filter_model', 'equal_status', 'equal_is_active'
    ]);

    // Default sort
    $filter_data['sort']  = $request->query('sort', 'sort_order');
    $filter_data['order'] = $request->query('order', 'asc');

    // filter_name lives in the translation table (not main table)
    // → preprocess + unset to prevent OrmHelper from applying it again
    if ($request->filled('filter_name')) {
        $name = $request->filter_name;
        $locale = app()->getLocale();

        $query->whereHas('translations', function ($tq) use ($name, $locale) {
            $tq->where('locale', $locale);
            $tq->where(function ($sq) use ($name) {
                OrmHelper::filterOrEqualColumn($sq, 'filter_name', $name);
            });
        });

        unset($filter_data['filter_name']);
    }

    // OrmHelper auto-handles filter_model, equal_status, equal_is_active, and sort
    OrmHelper::prepare($query, $filter_data);

    $products = OrmHelper::getResult($query, $filter_data);
    $products->withPath(route('lang.ocadmin.catalog.products.list'))->withQueryString();
    // ...
}
```

The whole getList flow is clear:

1. `filterData` whitelist lists 4 allowed params
2. Default sort `sort_order asc` (product table has a custom sort column)
3. `filter_name` is **not a main-table column** (it's in translations) → preprocess via `whereHas` + `unset` to stop OrmHelper from applying it again
4. `OrmHelper::prepare` automatically handles the remaining `filter_model`, `equal_status`, `equal_is_active`, `sort`, `order`
5. `OrmHelper::getResult` auto-paginates

"Manually preprocess non-main-table columns + unset" is a common pattern — let OrmHelper run only on directly-mappable columns, handle special cases manually.

## 4. PermissionController example: custom search and non-standard params

`app\Portals\Ocadmin\Core\Controllers\System\Acl\PermissionController.php`:

```php
protected function getList(Request $request): string
{
    $query = Permission::with('translations');
    $filter_data = $this->filterData($request, ['equal_is_active']);

    // Default sort
    $filter_data['sort']  = $request->query('sort', 'name');
    $filter_data['order'] = $request->query('order', 'asc');

    // filter_portal: custom portal-prefix filter (non-standard match)
    if ($request->filled('filter_portal') && $request->filter_portal !== '*') {
        $query->where('name', 'like', $request->filter_portal . '.%');
    }
    unset($filter_data['filter_portal']);

    // search: cross-column OR (main-table name + translation-table display_name)
    if ($request->filled('search')) {
        $search = $request->search;
        $locale = app()->getLocale();

        $query->where(function ($q) use ($search, $locale) {
            OrmHelper::filterOrEqualColumn($q, 'filter_name', $search);

            $q->orWhereHas('translations', function ($tq) use ($search, $locale) {
                $tq->where('locale', $locale);
                OrmHelper::filterOrEqualColumn($tq, 'filter_display_name', $search);
            });
        });

        unset($filter_data['search'], $filter_data['filter_name']);
    }

    OrmHelper::prepare($query, $filter_data);

    $permissions = OrmHelper::getResult($query, $filter_data);
    // ...
}
```

The advanced case adds two custom handlers:

- **`filter_portal`** (custom prefix filter): not in the whitelist (filterData would drop it), so read via `$request->filled('filter_portal')` directly; `unset` afterward to avoid leftovers
- **`search`** (cross-column OR search): reuse OrmHelper's `filterOrEqualColumn` but compose the OR logic manually (OR across main table + translation table)

**Pattern summary**: whitelist → preprocess special logic (with `unset`) → OrmHelper finish. Every list page follows the same three steps — only the preprocessing portion varies.

## 5. What else OrmHelper does along the way

Beyond the core "loop over filter_/equal_", `OrmHelper::applyFilters()` also throws in a few conveniences:

### 5.1 Default `is_active` filtering

If the Model has an `is_active` column and the caller **doesn't explicitly specify `equal_is_active`**, it auto-applies `equal_is_active=1`:

```php
if (in_array('is_active', $tableColumns)) {
    if (!isset($params['equal_is_active'])) {
        $params['equal_is_active'] = 1;       // default: only show active
    } elseif ($params['equal_is_active'] === '*') {
        unset($params['equal_is_active']);    // pass * to see everything
    } else {
        $params['equal_is_active'] = (int) $params['equal_is_active'];
    }
}
```

This echoes the "soft retirement" idea from [Settings Mechanism](/ocadmin/en/settings/) — applied here per Model's list page, hiding inactive rows by default.

### 5.2 Auto whereHas for translation columns

If the Model uses the `HasTranslation` trait and `filter_xxx` matches a translation column (e.g. `filter_display_name`), it auto-routes through `whereHas('translations')` to the translation subtable, scoped to the current locale:

```php
protected static function applyTranslationFilters(EloquentBuilder $query, Model $model, array $filters): void
{
    $localeKey = method_exists($model, 'getLocaleKey') ? $model->getLocaleKey() : 'locale';
    $locale = app()->getLocale();

    $query->whereHas('translations', function ($q) use ($filters, $localeKey, $locale) {
        $q->where($localeKey, $locale);

        foreach ($filters as $key => $value) {
            self::filterOrEqualColumn($q, $key, $value);
        }
    });
}
```

This echoes the design in [Multilingual Mechanism](/ocadmin/en/multilingual/) — translation-subtable queries have built-in locale scoping, so other-locale data never leaks.

### 5.3 Smart date parsing

If the column matching `filter_xxx` has a `$casts` of date / datetime, it auto-routes through `DateHelper::applySmartFilter()`, supporting multiple formats:

```
?filter_delivery_date=20260301              → single day (8-digit)
?filter_delivery_date=260301                → single day (6-digit)
?filter_delivery_date=2026-03-01            → single day (dash format)
?filter_delivery_date=20260301-20260331     → range
?filter_delivery_date=>20260301             → after
?filter_delivery_date=<=20260331            → on or before
```

No need to write an `if` per date field — it's automatic.

## 6. OrmHelper vs Model Scope: division of labor

The split is clean:

| Scenario | Goes through |
|---|---|
| Param name → single table column (incl. translation columns, date) | **OrmHelper directly** |
| Multi-column OR search (phone matches mobile + telephone) | **Model Scope** |
| Cross-relation query (search orders by product name) | **Model Scope** |
| Special value transformation (`withoutV` → `status_code <> 'void'`) | **Model Scope** |
| Custom prefix filter (`filter_portal` → `name LIKE 'admin.%'`) | **Controller preprocess + unset** |

Rule of thumb: **param name maps directly to a single table column** → OrmHelper; doesn't map → Scope or Controller preprocessing. Details (why Scope lives on Model, naming) see the relevant section in [Layered Architecture](/ocadmin/en/layered-architecture/).

## 7. What else is in OrmHelper

Beyond the `prepare` / `getResult` core API pair, OrmHelper bundles a few commonly-used utilities (use on demand):

| Utility | Purpose |
|---|---|
| `filterOrEqualColumn($q, $key, $value)` | Single-column filter, supports `=` / `<>` / `>` / `<` / `*xxx` / `xxx*` operators |
| `filterJsonColumn($q, $col, $value)` | Uses `JSON_SEARCH` for JSON columns (avoids REGEXP accidentally matching key names) |
| `save($modelClass, $data, $id, $params)` | Unified create / update (with fillable filtering, translations sync) |
| `findIdOrFailOrNew($query, $id)` | findOrFail if id exists, new otherwise |
| `getTableColumns($table)` | Thin wrapper over `Schema::getColumnListing` (no extra cache; queries schema each call) |
| `showSqlContent($query)` | Debug helper: print the final generated SQL |

These are "outside the OrmHelper main flow but often used together" tools. Import as needed — no separate helper class required.

## 8. Where OrmHelper doesn't fit

OrmHelper is designed for "**list-page queries + standard CRUD**". The following do **not** fit:

- **Multi-table transactional writes** (creating an order while syncing products / options / payments) → use Repository
- **Fully custom query logic** (complex BI reports, joins across many relations) → write the query builder directly
- **Cursor pagination on large datasets** (over 100k rows) → use Laravel's `cursor()`, not OrmHelper's paginate

OrmHelper is the "**80% of list pages + 20% of standard operations**" tool — for the remaining 20% edge cases, fall back to plain Laravel APIs.

## Closing

`OrmHelper::prepare + filterData` condenses the "list-page query" — a highly repetitive scenario — into a convention, so you don't have to write `if(request()->has('xxx')) ...` everywhere.

The cost is **learning one in-house convention** (`filter_xxx` / `equal_xxx` naming + operators). The payoff is **every list page follows the same template** — learn the first getList once, you can read hundreds of getLists afterward.

Relationship to the three roles in [Layered Architecture](/ocadmin/en/layered-architecture/): OrmHelper lives in [`app\Helpers\`](/ocadmin/en/layered-architecture/) as a cross-cutting technical utility — doesn't know any specific entity, doesn't know which Portal, callable from any Module Service or Controller. It's a convenience tool for the query layer, not a domain business orchestration layer.

Once you're comfortable with this tool, looking back at plain Laravel `if(request()->filled(...))` style code makes the boilerplate stand out.
