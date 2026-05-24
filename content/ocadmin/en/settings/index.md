---
title: "Settings Mechanism: What You Need to Solve Once Settings Live in the DB"
date: 2026-05-25T22:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "Settings", "Database Design"]
categories: ["OCAdmin"]
weight: 6
summary: "OCAdmin stores system parameters (per-page count, popup image width, IP whitelist, SMTP host…) in the DB rather than in .env or config files. But putting settings in the DB isn't free — you have to solve four problems: type, performance, soft retirement, and multilingual labels. This post explains why we replaced OpenCart's serialize flag with a type enum, why we designed the two-tier is_autoload read mechanism, and why we use is_active soft retirement instead of DELETE."
build:
  list: local
---

> [→ 繁體中文版](/ocadmin/settings/)

Every system has settings like "how many rows per page", "how wide should the popup image be", "which IPs can access the admin", "what's the SMTP host". This post is about how OCAdmin stores them, reads them, and retires them.

## 1. Why put settings in the DB?

Every system has plenty of parameter values to store. Laravel gives you three places to put them:

| Where | Cost of changing a value | Best suited for |
|---|---|---|
| `.env` | SSH into the server, edit the file, restart the process | Secrets, environment differences (DB credentials, API keys) |
| `config/*.php` | Edit file → commit → deploy | Programmatic constants (things that never change at runtime) |
| **DB** | Click "save" in the admin UI | **Business parameters** (things admins need to change, things that vary) |

"How many rows per the backend list" is too heavy for `.env` — non-engineers can't change it. Putting it in `config/*.php` is also wrong — every tweak requires a deploy. **Only the DB lets admins change it themselves, see it take effect immediately, and leave an audit trail.**

OCAdmin follows OpenCart's playbook here: all business parameters live in the DB. But OpenCart's `oc_setting` table has a few gaps, and OCAdmin adds four key columns to the prototype to solve four independent problems.

## 2. Table at a glance: `sys_settings`

Core columns (the full schema lives in internal docs; here are the highlights):

| Column | Purpose |
|---|---|
| `code` | Setting key, unique across the whole table, e.g. `admin_per_page` |
| `value` | The stored value (always a string; how to parse it depends on `type`) |
| `type` | Type enum: `text` / `int` / `bool` / `float` / `array` / `line` / `json` / `serialized` |
| `is_autoload` | true = preloaded into Config at boot, 0 DB query; false = on-demand fetch |
| `is_active` | false = soft-retired, `setting()` treats it as if it doesn't exist |
| `name` / `name_translations` | Display name (used by the admin UI), with multilingual support |
| `group` | Grouping label (e.g. `config` / `portal` / `mail`), purely for grouping rows in the admin UI |

Other columns (`note`, `created_by`, `updated_by`, `timestamps`) are audit columns — skipped here.

Examples:

| code | value | type | is_autoload |
|---|---|---|---|
| `admin_per_page` | `10` | int | false |
| `web_pop_img_width` | `800` | int | false |
| `admin_allowed_ips` | `127.0.0.1,::1` | array | **true** |
| `config_maintenance` | `0` | bool | false |
| `config_mail_smtp_host` | `smtp.gmail.com` | text | false |

The next four sections walk through the four key columns in turn.

## 3. `type`: an explicit enum replaces OpenCart's serialize flag

OpenCart's `oc_setting` has a single `serialized` column (0/1) telling the reader whether the value needs `json_decode`. Crude but workable. The problem: for non-serialized values, how does the caller know whether to cast to int, bool, or string?

OpenCart's answer: **the caller casts it themselves.**

```php
// OpenCart style: every caller has to do its own cast
$perPage = (int)$this->config->get('config_admin_limit');           // cast to int
$maintenance = (bool)$this->config->get('config_maintenance');      // cast to bool
```

This scattered cast logic has several problems:

- Missing a cast somewhere creates a bug (`'0'` is truthy — "maintenance off" gets read as "on")
- The same key is read in N places, and each place repeats the same type assumption
- There's no single source of truth — the "real" type is just whatever the most common cast happens to be in the codebase

OCAdmin moves this responsibility to **the setting itself**, using a `type` enum with eight explicit types:

```php
// app/Enums/System/SettingType.php
enum SettingType: string
{
    case Text       = 'text';        // plain text → string
    case Line       = 'line';        // multi-line → string[] (split by newline)
    case Json       = 'json';        // JSON → assoc array
    case Serialized = 'serialized';  // PHP serialize → mixed
    case Bool       = 'bool';        // boolean → bool
    case Int        = 'int';         // integer → int
    case Float      = 'float';       // decimal → float
    case Array      = 'array';       // comma-separated → string[]
}
```

A single Model accessor handles all types:

```php
public function getParsedValueAttribute(): mixed
{
    return match ($this->type) {
        SettingType::Bool       => filter_var($this->value, FILTER_VALIDATE_BOOLEAN),
        SettingType::Int        => (int) $this->value,
        SettingType::Float      => (float) $this->value,
        SettingType::Json       => json_decode($this->value, true),
        SettingType::Serialized => unserialize($this->value),
        SettingType::Array      => array_map('trim', explode(',', $this->value ?? '')),
        SettingType::Line       => array_filter(array_map('trim', explode("\n", $this->value ?? ''))),
        default                 => $this->value,
    };
}
```

Callers get the correct type back — **no manual cast needed**:

```php
$perPage = setting('admin_per_page');         // int 10
$popWidth = setting('web_pop_img_width');     // int 800
$ips = setting('admin_allowed_ips');          // ['127.0.0.1', '::1']
$maintenance = setting('config_maintenance'); // false (parsed from '0')
```

A nice side benefit: **the admin UI can adapt to type** — bool shows a checkbox, array shows a "comma-separated" hint, json shows a textarea with syntax validation, all driven by `type`. OpenCart's admin UI hardcodes if-else branches per business key in the controller; OCAdmin lets the enum drive it uniformly, so adding a new setting doesn't require touching UI code.

## 4. `is_autoload`: a two-tier read mechanism

Putting settings in the DB buys "admin can change them" at the cost of "every read is a DB hit". For low-frequency settings (read SMTP host once per request) that's fine, but some settings **are read on every single request** — for example:

- `admin_allowed_ips` (IP whitelist; checked by middleware on every request)
- `config_maintenance` (maintenance mode; checked at the start of every request)
- Other flags used by middleware or global scopes

Hitting the DB on every request to read these is obvious waste.

### Design: selective preload into Config

OCAdmin adds an `is_autoload` flag. Settings with `is_autoload=true` are bulk-loaded into Laravel's Config at boot:

```php
// app/Providers/SettingServiceProvider.php
public function boot(): void
{
    $settings = Setting::active()->where('is_autoload', true)->get();

    foreach ($settings as $setting) {
        Config::set("settings.{$setting->code}", $setting->parsed_value);
    }
}
```

Then the `setting()` helper does a two-tier lookup:

```php
function setting(string $code, mixed $default = null): mixed
{
    // Tier 1: Config (autoload settings, 0 DB query)
    if (config()->has("settings.{$code}")) {
        return config("settings.{$code}") ?? $default;
    }

    // Tier 2: DB (per-request static cache; 0 queries on subsequent reads in the same request)
    static $cache = [];
    if (array_key_exists($code, $cache)) {
        return $cache[$code] ?? $default;
    }

    $row = Setting::active()->where('code', $code)->first();
    $cache[$code] = $row?->parsed_value;

    return $cache[$code] ?? $default;
}
```

Net effect:

| Setting kind | DB cost per request |
|---|---|
| `is_autoload=true` (high frequency) | **0 queries** |
| `is_autoload=false` (low frequency) | 1 query on first read, 0 afterwards |

### Why not "load everything" like OpenCart?

OpenCart loads the entire `config` group (dozens of settings) into memory at boot by default. Fine for small systems, but two problems:

1. **Startup cost scales linearly with setting count** — especially since PHP-FPM has to load it once per worker
2. **No way to distinguish high and low frequency** — a setting only read once in a single backend edit page still gets loaded on every request

The spirit of `is_autoload` is: **let the data designer explicitly mark high-frequency settings**, and fetch the rest on demand. The default for new settings is `is_autoload=false`; you only flip it true when you've explicitly determined the setting will be used in middleware or global code.

### Known limitation

The Config preload only runs at boot, so **changing a setting in the admin within the same request won't show up in Config immediately** — but the next request will pick it up. If you really need same-request reflection (rare), manually call `Config::set()` after the update.

## 5. `is_active`: soft retirement is safer than DELETE

After a setting row has been used for a while, some settings become "used to be in use, not sure if anyone still reads it". The intuitive move is to DELETE them, but DELETE has several risks:

- **What if some obscure code path still reads it?** After DELETE, `setting()` returns `null` (or the default) — features might silently break, and the bug is hard to trace
- **Historical context disappears** — once deleted, you can't look back to find what value the key used to have
- **Rollback is painful** — restoring a deleted setting requires manual recovery

OCAdmin adds an `is_active` flag for **soft retirement**:

```php
// Setting model scope
public function scopeActive(Builder $query): Builder
{
    return $query->where('is_active', true);
}
```

Both `SettingServiceProvider` and the `setting()` helper apply `active()` automatically — **an inactive row is invisible to `setting()`**, which automatically falls back to the default.

### Three-stage retirement

```
Mark inactive  →  Observe for N weeks  →  Nobody complained  →  DELETE
     ↑                  ↑                       ↑                ↑
fallback catches    logs stay clean    confirms no one reads   safe cleanup
```

A side benefit of not deleting immediately: **`updated_at` becomes a liveness signal**. With the row preserved, you can come back months later and see "last edit was two years ago + no complaints since marking inactive", and confidently DELETE it.

### Admin UI behavior

- The list shows only active rows by default; a toggle reveals inactive (greyed out)
- The edit form can switch between inactive / active — no DELETE required
- A batch-delete button remains for the final "definitely not needed" cleanup

## 6. `name_translations`: values are locale-agnostic, but labels can be multilingual

[Multilingual mechanism](/ocadmin/en/multilingual/) covered OCAdmin's three independent layers (UI, URL, content). **Which layer does a setting value belong to?**

Answer: **most settings have nothing to do with language**. `admin_per_page = 10`, `web_pop_img_width = 800`, `config_mail_smtp_host = smtp.gmail.com` — these values don't have a "Chinese version" and an "English version".

But the **display name** (the label shown to admins in the backend) might need to be multilingual. The name "後台列表每頁筆數" should show as "Admin list per-page" for an English-speaking admin.

OCAdmin separates this into two columns:

| Column | Content | Example |
|---|---|---|
| `name` | Display name in the default locale (required) | `後台列表每頁筆數` |
| `name_translations` | Other-locale translations, JSON `{locale: name}` | `{"en": "Admin list per-page"}` |

A Model accessor does the fallback:

```php
public function getTranslatedNameAttribute(): ?string
{
    $locale = app()->getLocale();
    return $this->name_translations[$locale] ?? $this->name;
}
```

`$setting->translated_name` always returns something — if the current locale has no translation, it falls back to `name`. No missing values, ever.

### Why not "one row per locale"?

OpenCart's design carries a language dimension on the setting row itself (historically via `language_id`). OCAdmin considered `unique(locale, code)` early on but dropped it. The reasoning (also covered in the [multilingual mechanism](/ocadmin/en/multilingual/) "historical decision" section):

- The vast majority of settings have nothing to do with language (per_page, SMTP, on/off flags)
- `locale=''` becomes semantically ambiguous — is it "global" or "forgot to fill in"?
- Goes against OpenCart's own design (OpenCart's `oc_setting` doesn't have a locale dimension either; only a few module-specific settings handle multilingual through other means)

**Conclusion**: `code` is unique across the whole table; only display names are multilingual, and values don't vary by locale. For the rare case where you genuinely need "different values per language", use two codes (`foo_zh` / `foo_en`) and let the caller decide which to read.

### When you enable a second locale

Old rows have `NULL` in `name_translations`, and the accessor falls back to `name` — **no data migration needed**, old data is compatible as-is. New setting forms automatically show multilingual tabs in the admin UI; fill in translations if you want, leave blank if you don't care.

## 7. `setting()` helper: hide all the details

I just spent six sections on design choices, but **the caller doesn't need to know any of them** — all reads go through one helper:

```php
$perPage = setting('admin_per_page', 10);
$popWidth = setting('web_pop_img_width', 800);
$ips = setting('admin_allowed_ips', []);
$maintenance = setting('config_maintenance', false);
```

The caller doesn't know:

- Whether the value is in Config or in the DB
- What type to cast to (the helper returns the correct type)
- Whether the setting is inactive (inactive falls back to the default automatically)
- Whether there's a per-request cache

The helper is barely a dozen lines, but it encapsulates the entire mechanism from §3 §4 §5. The caller sees only a minimal API: "give a code, get a value".

### `Setting` model is still usable directly

For multi-row queries (admin UI, batch processing), use the model directly:

```php
$mailSettings = Setting::active()
    ->where('group', 'config')
    ->where('code', 'like', 'config_mail_%')
    ->get()
    ->keyBy('code');

$host = $mailSettings['config_mail_smtp_host']?->parsed_value;
$port = $mailSettings['config_mail_smtp_port']?->parsed_value;
```

The `active()` scope and `parsed_value` accessor are shared with the helper, so behavior is consistent.

## 8. Summary: differences from OpenCart `oc_setting`

| Topic | OpenCart | OCAdmin |
|---|---|---|
| **Type info** | `serialized` 0/1 (only flags JSON or not) | `type` enum, 8 explicit values |
| **Parse responsibility** | Caller casts | Model `parsed_value` accessor parses |
| **Multi-store** | Built-in `store_id` column | `sys_settings` doesn't have it; multi-layer override designed separately (see below) |
| **Preload strategy** | Loads the entire `code='config'` group | `is_autoload` selective preload |
| **Retirement** | None, only DELETE | `is_active` soft retirement + 3-stage flow |
| **Multilingual labels** | None (setting value itself carries `language_id`) | `name_translations` JSON; **values are locale-agnostic** |
| **Admin UI** | Hardcoded if-else in admin controllers | Driven uniformly by `type` enum |

All these differences pull in the same direction: **move the responsibility for "how to interpret this data" onto the data definition itself** (expressed via enums and flags), so callers and the admin UI can adapt automatically without touching code for every new setting.

## 9. Coming up: multi-layer overrides (brand layer / store layer)

`sys_settings` doesn't carry `store_id`, but OCAdmin does have the "brand-layer / store-layer override" requirement — e.g. global default `admin_per_page=10`, but brand A wants 20, store B wants 5. How do we design this without turning `sys_settings` into multi-store hell?

OCAdmin takes the **separate extension table + helper-driven automatic fallback** route, keeping `sys_settings` clean. That design is worth its own post — coming up next.

---

Writing this post made me realize: settings look mundane, but behind the scenes they have to solve type, performance, retirement, multilingual, and UI automation simultaneously. **The four columns `type` / `is_autoload` / `is_active` / `name_translations` each handle one concern**, and individually none are complex — but **together they form a complete settings system**. Drop any one and you're back to OpenCart's pain points (scattered casts, full-load overhead, DELETE risks, hardcoded labels).

When designing this kind of "looks simple but has to last" foundational module, the trick isn't clever tricks — **it's knowing in advance which pain points will surface later, and designing the columns to handle them up front**.
