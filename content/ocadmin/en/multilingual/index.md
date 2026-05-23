---
title: "Multilingual Mechanism: Three Independent Layers — UI, URL, and Content"
date: 2026-05-25T10:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "i18n", "internationalization"]
categories: ["OCAdmin"]
weight: 3
summary: "OCAdmin's multilingual support isn't one thing — it's three: UI text, URL locale, and dynamic content translation. The three layers operate independently, share a single configuration, and are connected via middleware. This post explains the design rationale for each layer, why front-office and back-office language files aren't shared, and why content translations use a separate table instead of a JSON column."
build:
  list: local
---

> [→ 繁體中文版](/ocadmin/multilingual/)

[The first post](/ocadmin/en/introduction/) touched on how MVCL's `L` works in OCAdmin. That was just the tip of the "UI multilingual" iceberg. In practice, building a multilingual system means tackling **three independent problems** — UI text, URL structure, and dynamic content. This post unpacks the design rationale for each of the three layers.

## Multilingual isn't one thing — it's three

Many frameworks treat "multilingual" as a single concept, end up solving only one aspect, and the other two get patched in ad hoc later. OCAdmin splits it into **three layers from the start**:

| Layer | Handles | Mechanism |
|---|---|---|
| **UI multilingual** | Buttons, field labels, prompts (text hardcoded in the source) | `lang/` files + `__()` |
| **URL multilingual** | URL prefixes like `/zh-hant/`, `/en/` | URL `{locale}` segment + `SetLocale` middleware |
| **Content multilingual** | Dynamic data managed by users (category names, product titles…) | `xxx_translations` translation tables + `HasTranslation` trait |

The three layers are **independent but tied together via middleware**: the `{locale}` in the URL triggers `SetLocale` middleware → calls `App::setLocale()` → both UI `__()` and content `HasTranslation` automatically follow.

Let's go through each layer's design.

## 1. UI Multilingual: `lang/` + `__()`

**Purpose**: text like "Save," "Name," or "Created successfully" — the **fixed UI text hardcoded in the source**.

### Architectural decision: complete separation of front-office and back-office language files

Following OpenCart 4.x's approach, the **front-office (public site) and back-office (admin) language files are kept completely separate, with no shared dependencies**:

```
lang/
├── zh_Hant/
│   ├── admin/         ← back-office only (buttons, fields, messages)
│   ├── front/         ← front-office only
│   └── global/        ← shared across Portals (e.g., Enum display names)
└── en/
    └── (same structure as zh_Hant)
```

### Why not share them? A decision earned through repeated mistakes

The intuitive approach is "words used on both sides (like 'Save') should be shared to avoid duplication." After multiple attempts I abandoned this, for reasons:

- **The same key can mean different things on each side**: in the back-office, "Add" means an admin creating a record; on the front-office, "Add" might mean a customer adding to cart — same button text, different context
- **Most keys are one-sided**: back-office needs `button_export`, `text_confirm_batch_delete`; front-office needs `text_add_to_cart`, `text_checkout`. Mixing them forces the constant judgment of "does the front-office need this key?" — wasted cognitive effort
- **The pressures of evolution run opposite**: front-office text evolves around conversion and brand voice; back-office text evolves around internal idioms. Sharing means cramming two different evolutionary pressures into one file

> **First-principles thinking**: the essence of the problem is "letting each Portal translate independently," not "how to share language files between front and back."
> **Path of least resistance**: if you don't unify, you don't have to spend effort deciding boundaries. Duplicating `button_save` isn't a problem — copy it once, and each side evolves separately.

Language files live in the **project's root `lang/` directory rather than inside individual Portal directories**: if the back-office is later re-implemented (Blade → Livewire), the language files stay intact.

### Not using Laravel's namespace mechanism

Language files are placed in the project's root `lang/` directory and aren't registered via `loadTranslationsFrom()`:

```php
__('admin/catalog/product.column_name')      // ✓ use the path directly as the key prefix
__('ocadmin::catalog.product.column_name')   // ✗ no namespace prefix
```

The benefit: **any Portal, at any layer, can access translations directly** without depending on ServiceProvider registration order — one fewer layer of indirection, one fewer failure point.

### Controller declares, View just uses (brief recap)

[The first post](/ocadmin/en/introduction/) covered the `TranslationLibrary` mechanism that OCAdmin borrowed from OpenCart and refined:

```php
// Controller
protected function setLangFiles(): array
{
    return ['admin/catalog/option'];   // base controller auto-merges admin/default
}
```

```blade
{{-- View just uses the key, no path required --}}
{{ $lang->button_save }}        {{-- Save (from admin/default) --}}
{{ $lang->heading_title }}      {{-- Option Management (from admin/catalog/option) --}}
```

Loading order, override mechanism, and other details are covered in the first post — not repeated here.

## 2. URL Multilingual: URL Locale + `SetLocale` Middleware

**Purpose**: the locale information in the URL — for example, `/zh-hant/admin/...` and `/en/admin/...` point to the same page in different languages.

### URL structure

```
/zh-hant/admin/config/taxonomies     ← Traditional Chinese
/en/admin/config/taxonomies          ← English
/admin/config/taxonomies             ← Auto-redirects to /zh-hant/...
```

The first URL segment is always the locale. **The URL is the single source of truth for locale switching** — clicking the language switcher essentially means rewriting the first URL segment from `zh-hant` to `en`; the rest is handled by middleware.

### How it works

```
Request: /en/admin/config/taxonomies/1/edit
    │
    ├─ Route matches {locale}/admin/...
    │
    ├─ SetLocale Middleware
    │   ├─ segment(1) = "en"
    │   ├─ LocaleHelper::toInternalFormat("en") → "en"  (URL format → internal format)
    │   ├─ App::setLocale("en")                  ← affects __() and HasTranslation
    │   ├─ URL::defaults(['locale' => 'en'])      ← route() automatically includes locale
    │   └─ forgetParameter('locale')              ← Controller doesn't receive locale param
    │
    └─ Controller / Blade runs normally; all route() calls auto-include locale
```

Middleware does three critical things:

1. **Set App locale** — so `__()` and `HasTranslation` both follow
2. **Set URL defaults** — so `route('lang.ocadmin.taxonomies.edit', $taxonomy)` automatically produces `/en/admin/...`; the Controller doesn't need to pass locale every time
3. **Forget the locale parameter** — Controller method signatures don't gain an extra `$locale`; business logic stays clean

### Why middleware, not directly calling a helper in the route file

This looks like it would work:

```php
// ✗ Looks reasonable. Actually broken.
Route::prefix(LocaleHelper::getCurrentLocale() . '/admin')->group(...);
```

**It doesn't**. `php artisan route:cache` would **freeze** the prefix to whatever locale was active at cache time — other locales would simply 404.

Middleware paired with the `{locale}` route parameter avoids this — `{locale}` is a dynamic segment, resolved per-request, unaffected by route caching:

```php
// ✓ The correct way
Route::group([
    'prefix'     => '{locale}/admin',
    'middleware' => 'setLocale',
], function () { ... });
```

## 3. Content Multilingual: Translation Tables + `HasTranslation` Trait

The first two layers handle "text hardcoded in source." But what about **data the user enters in the admin panel**? Category names, product titles, option specs — users need to translate these themselves, stored in the database.

### Translation table structure

Each main table that needs to be multilingual gets a companion `xxx_translations` table:

```
taxonomies                    taxonomy_translations
┌────┬──────┬──────┐         ┌────┬─────────────┬────────┬────────┐
│ id │ code │ ...  │         │ id │ taxonomy_id │ locale │ name   │
├────┼──────┼──────┤         ├────┼─────────────┼────────┼────────┤
│  1 │ skill│      │────────→│  1 │           1 │zh_Hant │ 技能   │
│    │      │      │         │  2 │           1 │ en     │Skills  │
└────┴──────┴──────┘         └────┴─────────────┴────────┴────────┘
```

The main table holds **only non-translatable columns** (`code`, `sort_order`, `is_active`…); translatable fields (`name`, `description`) live in the child table.

### Why not add `name_zh`, `name_en` columns to the main table?

Intuitive but a dead end:

- **Adding a language requires ALTER TABLE** — adding Japanese means adding `name_ja`, `description_ja`, etc. — N columns × N languages, explosion
- **Querying is awkward**: "find records that have translations in N languages" becomes an OR chain across many columns
- **Not normalized**: violates normalization principles, will bite eventually in some scenario

### Why not use a JSON column either?

MySQL JSON can hold `{"zh_Hant":"技能","en":"Skills"}`, seemingly easy, but:

- **Poor indexing/sorting**: can't directly `ORDER BY name COLLATE` per locale
- **Fallback logic is yours to write**: "fallback to default when current locale is missing" has to be handled at the view layer every time
- **Join-based filtering is hard**: "find all records that have an English translation" requires complex JSON path queries

The child table adds a JOIN, but **the structure is clean, queries are clear, and you can index per-locale** — far lower maintenance cost long-term.

### Model setup

Main table Model:

```php
class Taxonomy extends Model
{
    use HasTranslation;

    public array $translatedAttributes = ['name'];   // declare which fields live in the translation table
}
```

Reading:

```php
$taxonomy->name;                            // auto-returns current locale's translation
$taxonomy->translate('en')->name;           // specify locale
$taxonomy->translateOrDefault('ja')->name;  // with fallback (returns default if ja is missing)
```

Writing:

```php
$taxonomy->saveTranslations([
    'zh_Hant' => ['name' => '技能'],
    'en'      => ['name' => 'Skills'],
]);
```

The reason `$taxonomy->name` "automatically" returns the current locale is that the `HasTranslation` trait **overrides `getAttribute()`** — internally it calls `App::getLocale()` to check the current locale, then fetches the translation from the table.

**This is why, after layer 2's `SetLocale` middleware calls `App::setLocale()`, layer 3's content multilingual follows automatically** — the entire three-layer integration hinges on that single `App::setLocale()` call.

## How the three layers connect

The integration point is the moment `SetLocale` middleware calls `App::setLocale()`:

```
URL /en/...
   ↓
SetLocale middleware
   ↓
App::setLocale('en')
   ├──→ __() automatically reads lang/en/...        (Layer 1 follows passively)
   └──→ HasTranslation auto-fetches en translation  (Layer 3 follows passively)
```

Layer 2 is itself **the trigger**, so the real relationship between layers is: **Layer 2 triggers → Layers 1 and 3 follow passively**.

This is also why `App::setLocale()` must be called at the middleware stage and can't be deferred to the Controller — otherwise the state is inconsistent before views start rendering, producing the bizarre combination of "English UI but Chinese data."

## Design summary

The three layers are independent but share `config/localization.php` (which locales are supported, the default locale, URL format to internal format mapping). Adding a new locale (say Japanese) just means:

1. Add `'ja'` to `config/localization.php`
2. Create the `lang/ja/` directory and translate the language files
3. Use the admin panel's "translation" tabs to let users input Japanese versions of dynamic data

The three layers have clear, separate extension points — **adding a locale won't result in the scattered "modified A but forgot to modify B" situation**.

## What's next

Upcoming posts will cover:

- **Permission mechanism**: role / permission naming conventions, Spatie integration, why `{role_prefix}.{module}.{resource}.{action}` is split this way
- **List page conventions**: consistent design for filtering, sorting, pagination, and URL parameter preservation
- **Dual-mode architecture**: trade-offs between Ocadmin doubling as the public site vs. running an independent Web Portal
