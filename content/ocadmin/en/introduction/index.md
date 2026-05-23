---
title: "OCAdmin Introduction: Why I Chose OpenCart Admin as the Design Blueprint"
date: 2026-05-24T14:00:00+08:00
draft: false
tags: ["OCAdmin", "OpenCart", "Laravel", "Admin Design"]
categories: ["OCAdmin"]
summary: "OCAdmin is a management system I rebuilt with Laravel, but its UI structure is borrowed directly from OpenCart's admin panel. Why not build it from scratch, use WordPress, or AdminLTE? This post explains the reasoning."
build:
  list: local
---

> [→ 繁體中文版](/ocadmin/introduction/)

This is the first article in the **OCAdmin series**. I'll start by explaining what OCAdmin is and why it looks the way it does. Later articles will dive into individual modules and design decisions.

## What is OCAdmin

In a nutshell: **a backend system written in Laravel, with its UI structure copied from OpenCart's admin panel**.

It's not a fork of OpenCart, nor an OpenCart plugin. It's a project built from scratch on Laravel that simply borrows OpenCart's admin design blueprint — what list pages look like, which tabs an edit page has, where buttons go — so I don't have to re-think these UI details for every new project.

## Why OpenCart, and not something else

In practice, there are roughly three options:

### WordPress?

Also free, also PHP, with a huge community. But its architecture is **"a backend that evolved out of a blogging platform"** — the hook system, template hierarchy, and "post" concept leak everywhere. Turning it into a business/enterprise admin system means fighting a lot of historical baggage. The cost of deeper customization is not trivial.

### AdminLTE / CoreUI / Tabler — pure admin templates?

They provide beautiful UI components (buttons, tables, form fields, cards), but **they give you materials, not a blueprint for "what a typical page should look like"**.

In other words: you still have to decide whether the list page needs a search bar, whether filters go on the side or the top, how to divide an edit page into sections, where multilingual fields belong, what batch operations' UX should be. Re-thinking this for every project is inefficient.

### OpenCart: open source, complete page elements, easy to grasp

OpenCart is a fully functional e-commerce system. Its admin panel has **already solved the problems above hundreds of times**. Products, categories, orders, customers, all kinds of settings pages — every type of data management page is already designed.

Borrowing these "battle-tested" patterns is much faster than reinventing them. For anyone familiar with e-commerce admin panels, opening OpenCart's backend instantly tells them what each button is for — that's the value of "easy to grasp."

## OpenCart's admin has only two kinds of pages

Break the entire admin down, and **nearly every functional page is a variant of either a "list page" or an "edit page."**

### List page

```
┌──────────────────────────────────────────────┐
│ Title                  [Add] [Batch Delete]  │ ← top-right buttons
├──────────────────────────────────────────────┤
│ ☐  Field 1   Field 2   Field 3       Action  │
│ ☐  ...       ...       ...           [Edit]  │ ← right of each row
│ ☐  ...       ...       ...           [Edit]  │
└──────────────────────────────────────────────┘
       [Pagination]                  [Total]
```

Always has: filtering, sorting, pagination, checkbox batch operations.

### Edit page (product is the most representative example)

The product edit page essentially demonstrates every scenario you'll encounter in admin development:

- **Language tabs**: product name, meta_title, meta_description (**multilingual fields**)
- **Basic data**: SKU, Model, Categories... (**single-language, structured fields**)
- **Options tab**: product specs (**one-to-many complex sub-data**)
- **Image tab**: main image + extra images (**file upload management**)
- **Reward points tab**: bonus points settings (**conditional configuration**)
- **Special price tab**: promotional pricing (**time range + conditions**)

**Internalize this page's pattern and you can build the edit page for almost any business entity from the same mold — no need to assemble from scratch every time.** When developing a new module, just ask "which of these tabs is my entity most like?" and you have an answer.

## Program structure: separated frontend/backend + MVCL

OpenCart's native directory structure:

```
upload/
├── catalog/          ← Frontend (the shopping site customers see)
│   ├── model/
│   ├── controller/
│   ├── view/
│   └── language/
│
└── admin/            ← Backend (the admin panel for managers)
    ├── model/
    ├── controller/
    ├── view/
    └── language/
```

Each side is independent, with its own complete **MVCL** — Model / View / Controller / **Language**.

> The extra `L` (Language) compared to standard MVC is OpenCart's key design choice: **multilingual text is treated as a first-class citizen in the architecture**, not scattered around views with `<?= $lang['xxx'] ?>`. For systems that need internationalization, this decision saves a huge amount of language file organization pain down the line. OCAdmin carries this concept directly into Laravel.

OCAdmin borrows the `admin/` conventions and doesn't concern itself with `catalog/`.

## Multilingual: how MVCL's L works in OCAdmin

Here's a concrete example of OCAdmin's improvement to developer experience.

Laravel's native translation call looks like this:

```blade
{{ __('admin/system/acl/permission.column_display_name') }}
```

You have to write the **full file path + key** every time. Use a field 5 times on a page and you write the same prefix 5 times; relocate the lang file and every reference has to change; the long path makes lines visually noisy.

OCAdmin borrows OpenCart's approach: **the Controller declares which lang files to load, and the View just uses them**.

```php
// Controller: declare which lang files to load
class PermissionController extends OcadminController
{
    protected function setLangFiles(): array
    {
        return ['system/acl/permission'];
    }
}
```

The Controller will **load them in sequence** (later files override earlier ones, any number of layers allowed):

```
/lang/{locale}/admin/default.php                     ← shared buttons, messages (automatic)
/lang/{locale}/admin/system/acl/permission.php       ← this module (declared via setLangFiles)
```

```blade
{{-- View: just use the key, no path --}}
<label>{{ $lang->column_display_name }}</label>
<button>{{ $lang->button_save }}</button>
```

If needed, you can also manually add/override individual keys before passing to the view:

```php
$this->lang->column_name = 'Custom display name';
$data['lang'] = $this->lang;
return view('ocadmin::system.acl.permission.form', $data);
```

**The view only cares about which text to display, not where the text comes from** — that's MVCL's `L` paying off in practice. Details (fallback mechanism, key naming conventions, `tab_basic` / `tab_trans` standardization, etc.) are left for the upcoming "Multilingual Design" post.

## What OCAdmin built on top of OpenCart

| Borrowed from OpenCart | Reimplemented in Laravel |
|---|---|
| Backend UI conventions (list + edit page patterns, tab structures) | Routing, ORM, Middleware, DI container |
| Multilingual as a first-class concept | Eloquent + translation sub-tables |
| Frontend/backend separation philosophy | Service Provider registration |
| MVCL's `L` role | Laravel's native `lang/` directory |

In short: **the surface looks like OpenCart admin, the bones are Laravel**. OpenCart's legacy MVC engine, template system, and ORM aren't inherited — better alternatives exist in modern PHP. Only the UI/UX decision layer ("what should this page look like") is carried over.

## What's next in this series

Upcoming posts will gradually unfold:

- **Overall architecture**: Core vs Module — how I split them and why
- **Why I dropped breadcrumbs**: a decision record for intentionally *removing* a feature
- **Multilingual design**: lang file organization, default groups, fallback mechanics
- **List page conventions**: how filtering / sorting / pagination / URL preservation are designed consistently
- **Form AJAX submission**: why not use redirects, how errors are returned

Each post will be self-contained as much as possible — feel free to read whichever interests you.
