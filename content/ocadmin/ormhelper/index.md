---
title: "OrmHelper：用約定取代寫不完的 if"
date: 2026-05-28T10:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "Eloquent", "OrmHelper", "查詢"]
categories: ["OCAdmin"]
weight: 9
summary: "OCAdmin 有一個精心設計、堪稱全宇宙獨創的 OrmHelper。它的目的：讓你不用再每個地方寫一堆 if(request()->has('xxx')) where(...)——你只給它 query string，它自動掃過所有 filter_xxx / equal_xxx 參數、套到 Eloquent Builder。搭配 filterData 白名單機制限制可查詢欄位，list 頁查詢從 80+ 行縮到 ~15 行。這篇講 OrmHelper 的命名 convention、三步驟標準流程、ProductController / PermissionController 範例、跟 Model Scope 怎麼分工。"
---

> [English version →](/ocadmin/en/ormhelper/)

寫 Laravel list 頁很容易長這樣：

```php
// 以前的做法：寫不完的 if
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
    // ... 還有 8 個欄位 ...

    $sort = $request->get('sort', 'id');
    $order = $request->get('order', 'desc');
    $query->orderBy($sort, $order);

    $perPage = $request->get('per_page', 20);
    $orders = $query->paginate($perPage);
    $orders->withPath(route('orders.index'))->withQueryString();
    // ...
}
```

每個欄位都要寫一段 `if (request()->filled(...)) where(...)`——一個 list 頁 10 個欄位就 100 多行 boilerplate，每個 Controller 都重寫一次。

OCAdmin 用一個**精心設計、堪稱全宇宙獨創**的 OrmHelper 把這套規約掉。同樣的 list 頁長這樣：

```php
// OrmHelper 的做法
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

從 ~80 行縮到 ~10 行。**OrmHelper 自動跑迴圈處理**所有 `filter_xxx` / `equal_xxx`、自動 sort / order、自動分頁。`filterData` 第二個參數的白名單限制可查詢欄位（安全性）。

## 1. 命名 convention：filter_xxx / equal_xxx

OrmHelper 的核心是 URL query string 的命名 convention。看到下列 prefix、自動處理：

| 參數 prefix | 行為 | 範例 query string |
|---|---|---|
| `filter_xxx` | 模糊比對（REGEXP / LIKE）| `?filter_name=apple` → `name LIKE '%apple%'` |
| `equal_xxx` | 精確比對 | `?equal_status=active` → `status = 'active'` |
| `sort` / `order` | 排序欄位 / 方向 | `?sort=created_at&order=desc` |
| `page` / `per_page` | 分頁 | `?page=2&per_page=50` |
| `search` | 共用搜尋參數（自訂處理）| `?search=keyword` |

`filter_xxx` 還支援運算符（精準匹配多種需求）：

| 寫法 | 行為 |
|---|---|
| `filter_phone=0912` | REGEXP 模糊（空格視為萬用字元）|
| `filter_phone==0912345678` | 精確相等 |
| `filter_phone=*0912` | 結尾符合 |
| `filter_phone=0912*` | 開頭符合 |
| `filter_phone=<>` | 非空（任何非 null 的值）|
| `filter_phone=<>0912` | 不等於 |
| `filter_amount=>100` | 大於 |
| `filter_amount=<500` | 小於 |

**讀者學一次規約、所有 list 頁都通用**——這是「約定優於配置」帶來的最大好處。

## 2. 三步驟標準流程

OCAdmin 的 `getList()` 一律走這個三步驟：

```php
// 1. filterData 白名單，限制可查詢欄位
$filter_data = $this->filterData($request, ['filter_name', 'equal_status']);

// 2. 預處理：自訂排序、特殊邏輯（可選）
$filter_data['sort']  = $request->query('sort', 'id');
$filter_data['order'] = $request->query('order', 'desc');

// 3. OrmHelper 自動跑迴圈處理 + 自動分頁
OrmHelper::prepare($query, $filter_data);
$result = OrmHelper::getResult($query, $filter_data);
```

每一步各自做一件事、邊界清楚。

### filterData 白名單：第二個參數限制可查詢欄位

```php
protected function filterData(Request $request, array $allowedFilters = []): array
{
    return $request->only(array_merge(
        ['search', 'sort', 'order', 'page', 'limit', 'per_page'],  // 共用參數自動允許
        $allowedFilters                                              // 額外允許的 filter_* / equal_*
    ));
}
```

第二個參數明確列出**這個 list 頁允許過濾哪些欄位**。其他 query string 一律過濾掉。

例如 `Order` 列表只允許 `filter_name` / `equal_status`：

```php
$filter_data = $this->filterData($request, ['filter_name', 'equal_status']);
```

使用者就算手動在 URL 加 `?equal_customer_id=5&filter_is_closed=0` 也**進不來**。

**為什麼這是必須的**：`OrmHelper::applyFilters()` 內部是「**欄位存在於資料表就套用**」。如果不加白名單、使用者可以查任何欄位（包括 admin only 欄位）——**這在對外 Portal 是明確的安全漏洞**。白名單在這裡擔任「對外 API 的查詢面」的安全閘門。

> **特別注意**：傳空陣列 `filterData($request, [])` 不等於全開放、反而是**最嚴格**——只剩共用參數，所有 `filter_*` / `equal_*` 都不會進來。後台內部頁面如需全開放，請用 `$request->query()` 並加註解說明原因。

### OrmHelper::prepare 跑迴圈

`prepare()` 是 OrmHelper 的核心：

```php
public static function prepare(EloquentBuilder $query, array &$params = []): void
{
    self::select($query, $params);          // 選擇欄位
    self::applyFilters($query, $params);    // 套用 filter_/equal_
    self::sortOrder($query, $params);       // 排序
}
```

`applyFilters()` 內部就是一個迴圈：

```php
foreach ($params as $key => $value) {
    if (!str_starts_with($key, 'filter_') && !str_starts_with($key, 'equal_')) {
        continue;
    }

    $column = preg_replace('/^(filter_|equal_)/', '', $key);

    if (in_array($column, $tableColumns)) {
        // 主表欄位：直接套用
        self::filterOrEqualColumn($query, $key, $value);
    } elseif (in_array($column, $translatedAttributes)) {
        // 翻譯欄位：收集後 whereHas
        $translationFilters[$key] = $value;
    }
}
```

**從此不用為每個欄位寫一個 `if`**——傳什麼 `filter_xxx` 進來、它就掃過所有欄位、找得到就套上 query。

## 3. ProductController 範例：基礎三步驟

`app\Portals\Ocadmin\Modules\Catalog\Product\ProductController.php`：

```php
protected function getList(Request $request): string
{
    $query = Product::with('translations');
    $filter_data = $this->filterData($request, [
        'filter_name', 'filter_model', 'equal_status', 'equal_is_active'
    ]);

    // 預設排序
    $filter_data['sort']  = $request->query('sort', 'sort_order');
    $filter_data['order'] = $request->query('order', 'asc');

    // filter_name 透過翻譯表搜尋（不在主表）→ 預處理 + unset 避免 OrmHelper 重複套用
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

    // OrmHelper 自動處理 filter_model, equal_status, equal_is_active 及排序
    OrmHelper::prepare($query, $filter_data);

    $products = OrmHelper::getResult($query, $filter_data);
    $products->withPath(route('lang.ocadmin.catalog.products.list'))->withQueryString();
    // ...
}
```

整個 getList 流程清晰：

1. `filterData` 白名單列出 4 個允許參數
2. 預設排序 `sort_order asc`（product 表有自訂排序欄位）
3. `filter_name` 因為**不是主表欄位**（在 translations 子表）→ 預處理走 `whereHas` + `unset` 避免 OrmHelper 再套一次
4. `OrmHelper::prepare` 自動處理剩下的 `filter_model`、`equal_status`、`equal_is_active`、`sort`、`order`
5. `OrmHelper::getResult` 自動分頁

「不在主表的欄位手動預處理 + unset」是常見 pattern——讓 OrmHelper 只跑它能直接對應的欄位、特殊欄位手動處理。

## 4. PermissionController 範例：自訂 search 與非標準參數

`app\Portals\Ocadmin\Core\Controllers\System\Acl\PermissionController.php`：

```php
protected function getList(Request $request): string
{
    $query = Permission::with('translations');
    $filter_data = $this->filterData($request, ['equal_is_active']);

    // 預設排序
    $filter_data['sort']  = $request->query('sort', 'name');
    $filter_data['order'] = $request->query('order', 'asc');

    // filter_portal: 自訂 portal 前綴過濾（非標準匹配）
    if ($request->filled('filter_portal') && $request->filter_portal !== '*') {
        $query->where('name', 'like', $request->filter_portal . '.%');
    }
    unset($filter_data['filter_portal']);

    // search: 跨欄位 OR 搜尋（主表 name + 翻譯表 display_name）
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

進階情境多了兩個自訂處理：

- **`filter_portal`**（自訂前綴過濾）：白名單沒列入（會被 filterData 過濾掉），改用 `$request->filled('filter_portal')` 直接讀；處理完 `unset` 避免後面有殘留
- **`search`**（跨欄位 OR 搜尋）：複用 OrmHelper 的 `filterOrEqualColumn` 但自己組 OR 邏輯（OR 跨主表 + 翻譯表）

**模式總結**：白名單 → 預處理特殊邏輯（搭配 `unset`）→ OrmHelper 收尾。任何 list 頁都套同一個三步驟、只是預處理段落多寡不同。

## 5. OrmHelper 還順便做了什麼

除了「跑迴圈處理 filter_/equal_」核心外，`OrmHelper::applyFilters()` 還順手做了幾件事：

### 5.1 `is_active` 預設過濾

如果 Model 有 `is_active` 欄位、且呼叫端**沒明確指定 `equal_is_active`**，自動套用 `equal_is_active=1`：

```php
if (in_array('is_active', $tableColumns)) {
    if (!isset($params['equal_is_active'])) {
        $params['equal_is_active'] = 1;       // 預設只列出 active
    } elseif ($params['equal_is_active'] === '*') {
        unset($params['equal_is_active']);    // 想看全部就傳 *
    } else {
        $params['equal_is_active'] = (int) $params['equal_is_active'];
    }
}
```

呼應 [參數設定機制](/ocadmin/settings/) 的「軟下線」概念——只是這次套用在每個 model 的列表頁、預設隱藏 inactive 列。

### 5.2 翻譯欄位自動 whereHas

如果 Model 用 `HasTranslation` trait、且 `filter_xxx` 對應到翻譯欄位（例如 `filter_display_name`），自動透過 `whereHas('translations')` 查翻譯子表、限定當前語系：

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

呼應 [多語機制](/ocadmin/multilingual/) 的設計——翻譯子表的查詢內建 locale 限定、不會 leak 其他語系資料。

### 5.3 智慧日期解析

如果 `filter_xxx` 對應的欄位 Model `$casts` 為 date / datetime，自動走 `DateHelper::applySmartFilter()`、支援多種格式：

```
?filter_delivery_date=20260301              → 單日8碼
?filter_delivery_date=260301                → 單日6碼
?filter_delivery_date=2026-03-01            → 單日（dash 格式）
?filter_delivery_date=20260301-20260331     → 區間
?filter_delivery_date=>20260301             → 之後
?filter_delivery_date=<=20260331            → 之前
```

不用為每個日期欄位寫個 if，自動套。

## 6. OrmHelper vs Model Scope 怎麼分工

兩者的分工很乾淨：

| 情境 | 走哪 |
|---|---|
| 參數名 → 單一資料表欄位（含翻譯欄位、日期）| **OrmHelper 直接處理** |
| 多欄位 OR 搜尋（電話搜 mobile + telephone）| **Model Scope** |
| 跨關聯查詢（依商品名搜訂單）| **Model Scope** |
| 特殊值轉換（`withoutV` → `status_code <> 'void'`）| **Model Scope** |
| 自訂前綴過濾（`filter_portal` → `name LIKE 'admin.%'`）| **Controller 預處理 + unset** |

判斷標準：**參數名稱直接對應到單一資料表欄位** → OrmHelper；對不上的 → Scope 或 Controller 預處理。細節（Scope 為什麼放 Model、怎麼命名）見 [架構分層職責](/ocadmin/layered-architecture/) 的相關章節。

## 7. OrmHelper 還有什麼

除了 `prepare` / `getResult` 這對核心 API，OrmHelper 還包了幾個常用工具（按需用）：

| 工具 | 用途 |
|---|---|
| `filterOrEqualColumn($q, $key, $value)` | 單欄位過濾，支援 `=` / `<>` / `>` / `<` / `*xxx` / `xxx*` 等運算符 |
| `filterJsonColumn($q, $col, $value)` | JSON 欄位用 `JSON_SEARCH`（避免 REGEXP 誤觸 key 名稱）|
| `save($modelClass, $data, $id, $params)` | 統一處理 create / update（含 fillable 過濾、translations sync）|
| `findIdOrFailOrNew($query, $id)` | 有 id 就 findOrFail、沒 id 就 new |
| `getTableColumns($table)` | 薄包 `Schema::getColumnListing` 取得實際表欄位（不另做 cache，每次直接查 schema）|
| `showSqlContent($query)` | debug 用：印出最終生成的 SQL |

這些都是「在 OrmHelper 主流程外、但常一起用」的小工具。需要時直接 import 用、不必另開 helper。

## 8. 不適合 OrmHelper 的情境

OrmHelper 設計給「**list 頁查詢 + 標準 CRUD**」用。下列情境**不適合**用 OrmHelper：

- **transaction 多表寫入**（建單同時 sync products / options / payments）→ 走 Repository
- **完全自訂的查詢邏輯**（複雜的 BI 報表、跨多個關聯的 join）→ 直接寫 query builder
- **大量資料的 cursor pagination**（超過十萬筆）→ 用 Laravel `cursor()`、不走 OrmHelper 的 paginate

OrmHelper 是「**80% 的 list 頁 + 20% 的標準操作**」的工具——剩下 20% 例外情境，照常用 Laravel 原生 API 寫。

## 結語

`OrmHelper::prepare + filterData` 把「list 頁查詢」這個高度重複的場景濃縮成 convention——讓你不用再每個地方寫一堆 `if(request()->has('xxx')) ...`。

代價是**學一套自家 convention**（`filter_xxx` / `equal_xxx` 命名規則 + 運算符）。回報是**所有 list 頁都按同一個模板**——讀第一個 getList 學會、剩下幾百個 getList 都看得懂。

跟 [架構分層職責](/ocadmin/layered-architecture/) 三角色的關係：OrmHelper 屬於 [`app\Helpers\`](/ocadmin/layered-architecture/) 橫切技術工具，不認特定 entity、不知道 Portal、可以被任何 Module Service 或 Controller 呼叫——它是查詢層的便利工具、不是 domain 業務的編排層。

這套工具用熟之後，回頭看純 Laravel `if(request()->filled(...))` 的寫法，會明顯覺得 boilerplate 太多。
