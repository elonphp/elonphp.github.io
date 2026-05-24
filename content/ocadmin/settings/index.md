---
title: "參數設定機制：把設定放進 DB 後，需要解決的幾件事"
date: 2026-05-25T22:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "Settings", "資料庫設計"]
categories: ["OCAdmin"]
weight: 6
summary: "OCAdmin 把系統參數（每頁筆數、彈窗圖片寬度、IP 白名單、SMTP…）放進 DB 而非 .env 或 config。但把設定放進 DB 不是免費的——要解決型別、效能、軟下線、多語名稱四個問題。這篇講為什麼用 type enum 取代 OpenCart 的 serialize 旗標、為什麼設計 is_autoload 兩層讀取、為什麼用 is_active 軟下線取代 DELETE。"
---

> [English version →](/ocadmin/en/settings/)

系統那些「每頁顯示幾筆」「彈窗圖片要多寬」「哪些 IP 可以進後台」的設定值，OCAdmin 怎麼存、怎麼讀、怎麼下線。

## 1. 為什麼把設定放進 DB？

任何系統都有一堆參數值要存。Laravel 給了三個選擇：

| 放哪 | 改一個值要付什麼成本 | 適合什麼 |
|---|---|---|
| `.env` | SSH 進伺服器改檔、重啟 process | 機密、環境差異（DB 連線、API key） |
| `config/*.php` | 改檔 → commit → deploy | 程式邏輯常數（永遠不會在 runtime 變的東西） |
| **DB** | 後台按一下儲存 | **業務參數**（管理員要改的、會變的） |

「後台每頁顯示幾筆」這種值放 `.env` 太重——非工程師要改就改不了；放 `config/*.php` 也卡——每次調都得 deploy。**只有放 DB 才能讓管理員自己改、立刻生效、留下變動紀錄**。

OCAdmin 走 OpenCart 的老路，所有業務參數都進 DB。但 OpenCart 的 `oc_setting` 表結構有幾個地方不夠用，OCAdmin 在原型上加了四個關鍵欄位來解決四個獨立問題。

## 2. 表結構快覽：`sys_settings`

核心欄位（完整 schema 在內部規範文件，這裡只列重點）：

| 欄位 | 作用 |
|---|---|
| `code` | 設定代碼，全表唯一，如 `admin_per_page` |
| `value` | 設定值（永遠是字串，怎麼解析看 `type`） |
| `type` | 型別 enum：`text` / `int` / `bool` / `float` / `array` / `line` / `json` / `serialized` |
| `is_autoload` | true=啟動時預載到 Config，0 DB query；false=按需查 |
| `is_active` | false=軟下線，`setting()` 視同不存在 |
| `name` / `name_translations` | 顯示名稱（後台 UI 用），支援多語 |
| `group` | 分組（如 `config` / `portal` / `mail`），純後台 UI 分群用 |

其他像 `note`、`created_by`、`updated_by`、`timestamps` 是審計欄位，這裡略。

例子：

| code | value | type | is_autoload |
|---|---|---|---|
| `admin_per_page` | `10` | int | false |
| `web_pop_img_width` | `800` | int | false |
| `admin_allowed_ips` | `127.0.0.1,::1` | array | **true** |
| `config_maintenance` | `0` | bool | false |
| `config_mail_smtp_host` | `smtp.gmail.com` | text | false |

接下來四節依序講四個關鍵欄位。

## 3. `type`：明確型別取代 OpenCart 的 serialize 旗標

OpenCart 的 `oc_setting` 只有一個 `serialized` 欄位（0/1），告訴讀取端「這個值要不要 `json_decode`」。粗略但夠用。問題是——非 serialized 的值，呼叫端怎麼知道要當 int、bool、還是 string？

OpenCart 的答案：**呼叫端自己 cast**。

```php
// OpenCart 風格：每個地方都得自己處理
$perPage = (int)$this->config->get('config_admin_limit');  // 轉 int
$maintenance = (bool)$this->config->get('config_maintenance');  // 轉 bool
```

這個分散的 cast 邏輯有幾個問題：
- 漏 cast 一個地方就出 bug（`'0'` 是 truthy，會把「維護模式關閉」判成開啟）
- 同一個 key 在 N 個地方讀，每個地方要重複寫一次型別假設
- 沒有 single source of truth——哪個型別才對只能看程式碼最常見的 cast 是什麼

OCAdmin 把這個責任搬到**設定本身**，用 `type` enum 八種明確型別：

```php
// app/Enums/System/SettingType.php
enum SettingType: string
{
    case Text       = 'text';        // 純文字 → string
    case Line       = 'line';        // 多行 → string[]（換行切）
    case Json       = 'json';        // JSON → assoc array
    case Serialized = 'serialized';  // PHP serialize → mixed
    case Bool       = 'bool';        // 布林 → bool
    case Int        = 'int';         // 整數 → int
    case Float      = 'float';       // 小數 → float
    case Array      = 'array';       // 逗號分隔 → string[]
}
```

Model 一個 accessor 解所有型別：

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

呼叫端拿到的就是正確型別，**不用自己 cast**：

```php
$perPage = setting('admin_per_page');         // int 10
$popWidth = setting('web_pop_img_width');     // int 800
$ips = setting('admin_allowed_ips');          // ['127.0.0.1', '::1']
$maintenance = setting('config_maintenance'); // false（從 '0' 解出來）
```

也順便讓**後台 UI 可以針對型別變形**——bool 顯示 checkbox、array 顯示「逗號分隔」提示、json 顯示 textarea + 語法驗證，全憑 `type` 切換。OpenCart 的 admin UI 也是用業務 key 在 controller 端寫 if-else 處理，OCAdmin 改靠 enum 統一驅動，新增設定不用改 UI 程式碼。

## 4. `is_autoload`：兩層讀取機制

把設定放進 DB 換來「後台可改」，代價是「每讀一次要打 DB」。對低頻設定（一次 request 讀一次 SMTP host）這沒問題，但有些設定**每個 request 都要讀**——例如：

- `admin_allowed_ips`（IP 白名單，middleware 每個 request 都查）
- `config_maintenance`（維護模式，每個 request 開頭判斷）
- 其他用在 middleware / global scope 的旗標

每個 request 為了讀這幾個設定打一次 DB，明顯浪費。

### 設計：選擇性預載到 Config

OCAdmin 加 `is_autoload` 旗標。`is_autoload=true` 的設定，在 Laravel boot 時一次撈進來灌進 Config：

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

然後 `setting()` helper 做兩層查找：

```php
function setting(string $code, mixed $default = null): mixed
{
    // 第 1 層：Config（autoload 設定，0 DB query）
    if (config()->has("settings.{$code}")) {
        return config("settings.{$code}") ?? $default;
    }

    // 第 2 層：DB（per-request static cache，同一 request 後續 0 query）
    static $cache = [];
    if (array_key_exists($code, $cache)) {
        return $cache[$code] ?? $default;
    }

    $row = Setting::active()->where('code', $code)->first();
    $cache[$code] = $row?->parsed_value;

    return $cache[$code] ?? $default;
}
```

效果：

| 設定類型 | 一次 request 的 DB cost |
|---|---|
| `is_autoload=true`（高頻） | **0 query** |
| `is_autoload=false`（低頻） | 第一次讀 1 query，後續 0 query |

### 為什麼不像 OpenCart「全載」？

OpenCart 預設整個 `config` 群組（幾十筆設定）開機時全部撈進記憶體。對小系統可以，但有兩個問題：

1. **設定一多，啟動開銷線性增長**——尤其 PHP-FPM 每個 worker 都要載一次
2. **無法區分高低頻**——一個只在後台某個編輯頁讀一次的設定，每個 request 都跟著載

`is_autoload` 的精神是：**讓資料設計者明確標註哪些是高頻**，其他按需查。新增設定的預設值是 `is_autoload=false`，要明確判定「會在 middleware / global 用」才開 true。

### 已知的限制

Config 預載只在 boot 時跑，**同 request 內後台改了設定不會立即反映在 Config**——但下一個 request 就會更新。如果要同 request 內反映（罕見），update 後手動 `Config::set()` 一下就好。

## 5. `is_active`：軟下線比 DELETE 安全

設定 row 用一段時間後，總有些「以前在用、現在不確定還在不在用」的。直覺做法是 DELETE 掉，但 DELETE 有幾個風險：

- **萬一某段冷僻 code 還在偷讀**：DELETE 後 `setting()` 回 `null`（或 default），可能讓功能默默壞掉，bug 很難追
- **歷史脈絡丟失**：刪了就沒了，事後要追「這個 key 以前是什麼值」就難
- **回滾困難**：刪了要復原得手動補

OCAdmin 加 `is_active` 旗標走**軟下線**：

```php
// Setting model scope
public function scopeActive(Builder $query): Builder
{
    return $query->where('is_active', true);
}
```

`SettingServiceProvider` 和 `setting()` helper 都自動套用 `active()`——**inactive 的 row `setting()` 視同不存在**，自動 fallback 到 default。

### 三段下線流程

```
標 inactive  →  觀察 N 週  →  真的沒人抱怨  →  DELETE
   ↑              ↑              ↑              ↑
fallback 接住  log 沒爆     確認真的沒人讀    安全清除
```

不直接 DELETE 還有一個附帶好處：**`updated_at` 可以推斷活躍度**。標 inactive 後 row 留著，幾個月後再回來看，`updated_at` 顯示「上次改是兩年前 + 標 inactive 後沒人抱怨」，就能放心 DELETE。

### 後台 UI 行為

- 列表預設只顯示 active；提供 toggle「顯示 inactive」（灰底列）
- 編輯表單可切 inactive / active，不用走 DELETE
- 批次刪除按鈕保留，給「真的確定不用」的最終清理

## 6. `name_translations`：值無關 locale，但顯示名稱可以多語

[多語機制](/ocadmin/multilingual/) 那篇講過 OCAdmin 的多語走「介面、網址、內容」三層獨立。**設定值本身屬於哪一層？**

答案是：**多數設定根本與語言無關**。`admin_per_page = 10`、`web_pop_img_width = 800`、`config_mail_smtp_host = smtp.gmail.com`——這些值沒有「中文版」「英文版」之分。

但**顯示名稱**（後台給管理員看的 label）可能要多語。「後台列表每頁筆數」這個名字，給英文管理員看時要顯示 "Admin list per-page"。

OCAdmin 的解法是分開兩個欄位：

| 欄位 | 內容 | 範例 |
|---|---|---|
| `name` | 預設語言的顯示名稱（必填） | `後台列表每頁筆數` |
| `name_translations` | 其他語言對應，JSON 格式 `{locale: name}` | `{"en": "Admin list per-page"}` |

Model accessor 自動 fallback：

```php
public function getTranslatedNameAttribute(): ?string
{
    $locale = app()->getLocale();
    return $this->name_translations[$locale] ?? $this->name;
}
```

呼叫 `$setting->translated_name` 永遠回得到字串——某 locale 沒翻就 fallback 到 `name`，不會缺值。

### 為什麼不是「整個 row 一個 locale」？

OpenCart 的設計是讓設定整 row 帶語言維度（早年靠 `language_id` 欄位）。OCAdmin 早期也考慮過 `unique(locale, code)`，後來廢棄，理由講在 [多語機制](/ocadmin/multilingual/) 的「歷史決議」段：

- 絕大多數設定與語言無關（per_page、SMTP、開關值）
- `locale=''` 是「通用」還是「忘了填」語意混淆
- 違反 OpenCart 設計（OpenCart 的 `oc_setting` 也沒有 locale 維度，只有少數 module 設定靠別的方法處理多語）

**結論**：`code` 全表唯一，只有顯示名稱多語，值不分 locale。需要「中英文不同值」的罕見場景，用兩個 code 處理（`foo_zh` / `foo_en`），靠呼叫端決定要讀哪個。

### 啟用第 2 種 locale 時

舊資料的 `name_translations` 都是 NULL，accessor 自動 fallback 到 `name`——**不用 data migration**，舊資料直接相容。新建設定時後台 UI 自動長出多語 tab，要翻譯就填、不填也沒事。

## 7. `setting()` helper：把所有細節藏起來

前面講了一堆設計，但**呼叫端不需要知道**——所有讀取都走一個 helper：

```php
$perPage = setting('admin_per_page', 10);
$popWidth = setting('web_pop_img_width', 800);
$ips = setting('admin_allowed_ips', []);
$maintenance = setting('config_maintenance', false);
```

呼叫端不知道：

- 這個值在 Config 還是 DB
- 要 cast 成什麼型別（helper 回的就是正確型別）
- 該設定是不是 inactive（inactive 直接 fallback 到 default）
- 有沒有 per-request cache

helper 本身只 10 幾行，但把 §3 §4 §5 三個機制全部封裝起來，呼叫端只看到「給 code、回值」這個極簡 API。

### `Setting` model 仍可直接用

需要跨多筆查（後台 UI、批次處理）時直接用 model：

```php
$mailSettings = Setting::active()
    ->where('group', 'config')
    ->where('code', 'like', 'config_mail_%')
    ->get()
    ->keyBy('code');

$host = $mailSettings['config_mail_smtp_host']?->parsed_value;
$port = $mailSettings['config_mail_smtp_port']?->parsed_value;
```

`active()` scope + `parsed_value` accessor 跟 helper 共用，行為一致。

## 8. 跟 OpenCart `oc_setting` 的差異總結

| 議題 | OpenCart | OCAdmin |
|---|---|---|
| **型別資訊** | `serialized` 0/1（只標 JSON 與否） | `type` enum 八種，明確表達 |
| **解析責任** | 呼叫端 cast | Model `parsed_value` accessor 解 |
| **多商店** | 內建 `store_id` 欄位 | `sys_settings` 不加；多層覆寫另設計（見下節） |
| **預載策略** | 整個 `code='config'` 全載 | `is_autoload` 選擇性預載 |
| **下線機制** | 無，只能 DELETE | `is_active` 軟下線 + 三段流程 |
| **顯示名稱多語** | 無（設定值本身帶 `language_id`） | `name_translations` JSON，**值不分 locale** |
| **後台 UI** | 寫死在 admin controller 的 if-else | 靠 `type` enum 統一驅動 |

幾個差異都是同一個方向的延伸：**把「資料怎麼解讀」的責任搬到資料定義本身**（用 enum / 旗標表達），呼叫端跟 admin UI 都靠資料屬性自動處理，不用為每個新設定改程式碼。

## 9. 接下來：多層擴充（品牌層 / 門市層）

`sys_settings` 不帶 `store_id`，但 OCAdmin 確實有「品牌層 / 門市層覆寫」的需求——例如全域預設 `admin_per_page=10`，但 A 品牌想要 20、B 門市想要 5。這要怎麼設計才不會把 `sys_settings` 變成多商店地獄？

OCAdmin 走的是**獨立擴充表 + helper 自動 fallback**的路線，不污染 `sys_settings`。這個設計值得獨立一篇，未來會寫。

---

寫完這篇才發現參數設定看似平常，背後其實同時要解決型別、效能、下線、多語、UI 自動化五件事。**`type` / `is_autoload` / `is_active` / `name_translations` 四個欄位各自負責一件**，每件單獨看都不複雜，但**這些欄位放在一起才構成完整的設定系統**——少任何一個就會回到 OpenCart 的痛點（cast 散落、全載開銷、DELETE 風險、顯示名硬編碼）。

設計這類「業務看起來簡單，但要長期維護」的基礎模組，重點不在炫技，**而是清楚知道哪些痛點以後會冒出來，提早把對應的欄位設計進去**。
