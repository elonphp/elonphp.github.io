---
title: "多語機制：介面、網址、內容三層獨立設計"
date: 2026-05-25T10:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "多語系", "i18n"]
categories: ["OCAdmin"]
weight: 3
summary: "OCAdmin 的多語不是一件事，而是三件——介面文字、網址 locale、動態資料翻譯。三者各自獨立運作、共用同一份設定，靠 middleware 串接。這篇講三層的設計脈絡、為什麼這樣分、為什麼前後台語系檔不共用，以及內容多語為什麼用翻譯表而不是 JSON 欄位。"
---

> [English version →](/ocadmin/en/multilingual/)

介面多語只是冰山一角。實務上做一個多語系統，你會遇到**三個獨立的問題**——介面文字、URL 結構、動態資料——這篇展開三層各自的設計脈絡。

## 多語不是一件事，是三件

很多框架把「多語」當成單一概念處理，結果只解決一個面向，剩下兩個各自冒出來變成 ad hoc 解。OCAdmin 一開始就把它**拆三層**：

| 層面 | 處理什麼 | 機制 |
|---|---|---|
| **介面多語** | 按鈕、欄位標題、提示訊息（程式碼寫死的固定文字） | `lang/` 語系檔 + `__()` |
| **網址多語** | URL 前綴 `/zh-hant/`、`/en/` | URL `{locale}` 段 + `SetLocale` middleware |
| **內容多語** | 後台管理的動態資料（分類名稱、商品標題…） | `xxx_translations` 翻譯表 + `HasTranslation` trait |

三者**獨立但靠 middleware 串聯**：URL 中的 `{locale}` 觸發 `SetLocale` middleware → `App::setLocale()` → 介面 `__()` 跟內容 `HasTranslation` 都自動跟著切。

下面分別講三層各自的設計。

## 1. 介面多語：`lang/` + `__()`

**處理**：按鈕「儲存」、欄位「名稱」、訊息「新增成功」這類**程式碼裡固定的 UI 文字**。

### 架構決策：前後台語系檔完全分離

OCAdmin 參考 OpenCart 4.x 的做法——前台跟後台的語系檔**完全分開、互不依賴**：

```
lang/
├── zh_Hant/
│   ├── admin/         ← 後台專用（按鈕、欄位、訊息）
│   ├── front/         ← 前台專用
│   └── global/        ← 跨 Portal 共用（Enum 顯示名稱等）
└── en/
    └── (同 zh_Hant 結構)
```

### 為什麼不共用？這是踩過坑才確定的決定

直觀想法是「兩邊都會用到的字（如『儲存』）應該共用，避免重複」。實際嘗試多次後放棄，理由：

- **同一個 key 兩邊語意不同**：後台「新增」是管理員加資料、前台「加入」是消費者加入購物車——同樣的 button 字、不同語境
- **大量 key 只有一邊用**：後台需要 `button_export`、`text_confirm_batch_delete`；前台需要 `text_add_to_cart`、`text_checkout`，混在一起判斷「這個 key 前台要不要」浪費認知資源
- **演化壓力相反**：前台改字會考慮 conversion、品牌調性；後台改字偏內部習慣用語。共用就是把兩種不同的演化壓力擠在同一個檔

> **第一性原理**：問題的本質是「讓每個 Portal 獨立翻譯」，不是「如何讓前後台共用語系檔」。
> **最小阻力**：不統一就不用花力氣決定邊界，重複 `button_save` 不是問題——複製一次後各自演化。

語言檔放在**專案根目錄 `lang/` 而非 Portal 內**：後台未來如果換實作（Blade → Livewire），語言檔原封不動。

### 不用 Laravel 的 namespace 機制

語系檔不透過 `loadTranslationsFrom()` 註冊 namespace，直接放 `lang/` 根目錄：

```php
__('admin/catalog/product.column_name')      // ✓ 直接用路徑當 key 前綴
__('ocadmin::catalog.product.column_name')   // ✗ 不用 namespace 前綴
```

好處：**任何 Portal、任何層級都能直接存取**，不受 ServiceProvider 註冊順序影響——少一層 indirection、少一個失效點。

### Controller 宣告、View 直接用（簡要回顧）

[首篇](/ocadmin/introduction/)講過 OCAdmin 借用 OpenCart 思路改造的 `TranslationLibrary` 機制：

```php
// Controller
protected function setLangFiles(): array
{
    return ['admin/catalog/option'];   // base controller 自動加 admin/default
}
```

```blade
{{-- View 直接用，不用寫路徑 --}}
{{ $lang->button_save }}        {{-- 儲存（來自 admin/default） --}}
{{ $lang->heading_title }}      {{-- 選項管理（來自 admin/catalog/option） --}}
```

疊載順序、覆寫機制等細節看首篇，這裡不重複。

## 2. 網址多語：URL Locale + `SetLocale` Middleware

**處理**：URL 中的語系資訊，例如 `/zh-hant/admin/...` 跟 `/en/admin/...` 對應同一頁面的不同語言版本。

### URL 結構

```
/zh-hant/admin/config/taxonomies     ← 繁體中文
/en/admin/config/taxonomies          ← English
/admin/config/taxonomies             ← 自動轉址到 /zh-hant/...
```

URL 第一段固定是 locale。**URL 是 locale 切換的唯一真實來源**——使用者點語系切換按鈕，本質上就是把網址第一段從 `zh-hant` 改成 `en`，剩下全部由 middleware 處理。

### 運作流程

```
Request: /en/admin/config/taxonomies/1/edit
    │
    ├─ Route 匹配 {locale}/admin/...
    │
    ├─ SetLocale Middleware
    │   ├─ 取 segment(1) = "en"
    │   ├─ LocaleHelper::toInternalFormat("en") → "en"  (URL 格式 → 內部格式)
    │   ├─ App::setLocale("en")                  ← 影響 __() 和 HasTranslation
    │   ├─ URL::defaults(['locale' => 'en'])      ← route() 自動帶 locale
    │   └─ forgetParameter('locale')              ← Controller 收不到 locale 參數
    │
    └─ Controller / Blade 正常運作，所有 route() 自動帶 locale
```

Middleware 做了 3 件關鍵的事：

1. **設 App locale** — 讓 `__()` 和 `HasTranslation` 全部跟上
2. **設 URL defaults** — 讓 `route('lang.ocadmin.taxonomies.edit', $taxonomy)` 自動產生 `/en/admin/...`，Controller 不必每次手動傳 locale
3. **遺忘 locale 參數** — Controller method signature 不會多一個 `$locale`，業務邏輯保持乾淨

### 為什麼用 Middleware 不直接在路由檔呼叫 helper

看似可以這樣寫：

```php
// ✗ 這樣寫看起來合理，實際會壞
Route::prefix(LocaleHelper::getCurrentLocale() . '/admin')->group(...);
```

**不行**。`php artisan route:cache` 會把這個 prefix **固定**成一個語系（cache 當下的 `currentLocale`），其他語系直接 404。

Middleware 搭配 `{locale}` 路由參數則沒這問題——`{locale}` 是動態段，每次 request 才解析，跟 cache 無關：

```php
// ✓ 對的寫法
Route::group([
    'prefix'     => '{locale}/admin',
    'middleware' => 'setLocale',
], function () { ... });
```

## 3. 內容多語：翻譯表 + `HasTranslation` Trait

前兩層處理「程式碼裡寫死的字」。但**使用者在後台輸入的資料**怎麼辦？分類名稱、商品標題、選項規格...這些必須讓使用者自己翻譯，存進資料庫。

### 翻譯表結構

每個需要多語的主表，配一張 `xxx_translations` 翻譯表：

```
taxonomies                    taxonomy_translations
┌────┬──────┬──────┐         ┌────┬─────────────┬────────┬────────┐
│ id │ code │ ...  │         │ id │ taxonomy_id │ locale │ name   │
├────┼──────┼──────┤         ├────┼─────────────┼────────┼────────┤
│  1 │ skill│      │────────→│  1 │           1 │zh_Hant │ 技能   │
│    │      │      │         │  2 │           1 │ en     │Skills  │
└────┴──────┴──────┘         └────┴─────────────┴────────┴────────┘
```

主表只放**不翻譯的欄位**（`code`、`sort_order`、`is_active`…），翻譯欄位（`name`、`description`）全部進子表。

### 為什麼不在主表加 `name_zh`、`name_en` 欄位？

直觀但行不通：

- **加一種語言要 ALTER TABLE**——加日文要加 `name_ja`、`description_ja`...，N 個欄位 × N 種語言爆炸
- **查詢困難**：要篩「有 N 語翻譯的 record」要 OR 串一堆欄位
- **不夠 normalized**：違反正規化原則，遲早會在某個情境出問題

### 為什麼也不用 JSON 欄位？

MySQL JSON 可以塞 `{"zh_Hant":"技能","en":"Skills"}`，看起來很省事，但：

- **索引/排序差**：無法直接 `ORDER BY name COLLATE` per locale
- **fallback 邏輯要自己寫**：「找不到當前語系，退到 default」變成 view 層每次都要處理
- **join 篩選很難**：「找所有有英文翻譯的 record」要寫複雜的 JSON path 查詢

子表雖然多一個 JOIN，但**結構乾淨、查詢直觀、可以 per-語系建 index**，長期維護成本低。

### Model 設定

主表 Model：

```php
class Taxonomy extends Model
{
    use HasTranslation;

    public array $translatedAttributes = ['name'];   // 宣告哪些欄位在翻譯表
}
```

存取：

```php
$taxonomy->name;                            // 自動回當前 locale 的翻譯
$taxonomy->translate('en')->name;           // 指定語系
$taxonomy->translateOrDefault('ja')->name;  // 帶 fallback（找不到 ja 回 default）
```

存：

```php
$taxonomy->saveTranslations([
    'zh_Hant' => ['name' => '技能'],
    'en'      => ['name' => 'Skills'],
]);
```

`$taxonomy->name` 之所以「自動」回當前 locale，是因為 `HasTranslation` trait **覆寫了 `getAttribute()`**，內部去 `App::getLocale()` 查當前語系，再到翻譯表撈資料。

**這就是為什麼第 2 層的 `SetLocale` middleware 設定 `App::setLocale()` 之後，第 3 層的內容多語會自動跟上**——三層的串聯機制就在這一個 `App::setLocale()` 呼叫上。

## 三層怎麼串接

整個流程的串聯點是 `SetLocale` middleware 設定 `App::setLocale()` 那一刻：

```
URL /en/...
   ↓
SetLocale middleware
   ↓
App::setLocale('en')
   ├──→ __() 自動讀 lang/en/...        (第 1 層被動配合)
   └──→ HasTranslation 自動撈 en 翻譯  (第 3 層被動配合)
```

第 2 層自己就是**觸發點**，所以三層的真實關係是：**第 2 層觸發 → 第 1、3 層被動配合**。

這也是為什麼設計上 `App::setLocale()` 必須在 middleware 階段就完成、不能延後到 Controller 內呼叫——否則 view 開始渲染前狀態不一致，會出現「介面是英文、資料還是中文」這種詭異組合。

## 設計總結

三層獨立、但共用 `config/localization.php` 的設定（支援哪些 locale、預設 locale、URL 格式與內部格式對照）。新增一個語系（例如日文）只要：

1. `config/localization.php` 加 `'ja'`
2. 建立 `lang/ja/` 目錄、複製語系檔翻譯
3. 後台用「內容翻譯」分頁讓使用者輸入動態資料的日文版本

三層各自的擴充點清楚分工，**新增語系不會出現「改 A 但忘了改 B」的散落式修改**。

## 接下來

下一篇預定講：

- **權限機制**：角色 / 權限的命名規範、Spatie 整合、`{role_prefix}.{module}.{resource}.{action}` 為什麼這樣切
- **列表頁規範**：篩選、排序、分頁、URL 參數保留的一致設計
- **雙模架構**：Ocadmin 兼任前台 vs 獨立 Web Portal 的取捨
