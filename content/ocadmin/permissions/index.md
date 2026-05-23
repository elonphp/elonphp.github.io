---
title: "權限機制：四段式命名、角色設計、Prefix 解耦"
date: 2026-05-25T16:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "Spatie", "權限管理", "ACL"]
categories: ["OCAdmin"]
weight: 5
summary: "OCAdmin 的權限機制建在 Spatie Permission 之上，但加了幾個關鍵設計：四段式命名（{portal}.{module}.{resource}.{action}）、super_admin 用 Gate::before 處理全系統最高權限、prefix 注入機制讓 seeder 不寫死 prefix。這篇講為什麼這樣設計、resource 為什麼用單數、enforcement 為什麼放 route middleware。"
---

> [English version →](/ocadmin/en/permissions/)

前面幾篇講了 [整體架構](/ocadmin/architecture/) 與 [多語機制](/ocadmin/multilingual/)。這篇講**權限**——OCAdmin 的角色與權限怎麼命名、怎麼檢查、為什麼這樣設計。

## 1. 不重新發明：用 Spatie Permission 當底座

[`spatie/laravel-permission`](https://spatie.be/docs/laravel-permission) 是 Laravel 生態系最成熟的權限套件之一，覆蓋角色、權限、wildcard、cache、policy 整套機制。**OCAdmin 沒有重寫**——直接用，把心力花在「怎麼用」上。

| Spatie 提供 | OCAdmin 在上面加的約定 |
|---|---|
| 角色 / 權限的 DB 結構與 ORM | 四段式命名規範 |
| `$user->can()` / `Gate` 整合 | Portal 前綴隔離 |
| `*` wildcard 支援 | 兩個全域角色：`super_admin`（Gate::before 放行）+ `system`（系統操作標記） |
| 內建權限定義快取 | 自訂角色組合快取（為了 0 DB query per request） |
| `HasRoles` trait | `hasPortalRole()`、`permName()`、`roleName()` 等 helper |

衍生專案不需要學一套新的權限系統，能用 Spatie 知識的地方繼續用，只多學「這幾個約定」。

## 2. 四段式命名：`{portal}.{module}.{resource}.{action}`

權限名稱固定四段，全小寫、snake_case：

```
admin.catalog.product.access     ← 後台 catalog 模組對 product 資源的「檢視」權限
admin.catalog.product.modify     ← 「修改」
admin.catalog.product.delete     ← 「刪除」
hrm.mss.employee.access          ← HR Portal 的 MSS 模組對 employee 的「檢視」
hrm.ess.profile.modify           ← HR Portal 的 ESS 模組對 profile 的「修改」
```

### 四段各自的角色

| 段 | 對應 | 範例 |
|---|---|---|
| `portal` | 應用入口（[Portal 概念](/ocadmin/architecture/) 在前篇講過） | `admin`, `hrm`, `pos` |
| `module` | 功能模組（Portal 內的業務分群） | `catalog`, `sale`, `system_acl`, `mss`, `ess` |
| `resource` | 資源類型 | `product`, `order`, `user`, `role`, `employee` |
| `action` | 操作動作 | `access`, `modify`, `delete`, `approve`, `export` |

### 為什麼一定要 4 段？

**3 段太少**：少了 portal，跨 Portal 的權限會撞名（後台的 `user` vs HR Portal 的 `user`）。實務上跨 Portal 是常態。

**5 段太多**：分得太細反而難記、難對齊命名空間。資源切到「product / order / employee」這個粒度已經夠用；再切到「product.option / product.image」這種子資源，通常還是同一個 controller 在管。

**4 段剛好對應到系統的階層**：

```
URL:         /admin/catalog/products/{id}
             │      │       │
             ▼      ▼       ▼
Permission:  admin.catalog.product.access
             │      │       │       │
Module dir:  Modules/Catalog/Product/
                     │       │
             (portal = 整個 Portal 概念)
```

四段對應整個系統的層次，**權限名字本身就是地圖**——看到 `admin.catalog.product.access` 你立刻知道：哪個 Portal、哪個目錄、哪個 Model、哪個動作。

### 為什麼 resource 用單數而不是複數？

這個決定常被質疑——畢竟 RESTful URL 用複數（`/products`、`/users`），業界 IAM 也常見複數（GCP IAM `compute.instances.get`、Spatie 官方文件範例 `edit articles`）。本系統刻意選**單數**，主要三個理由：

**1. 跟 Eloquent Model 命名對齊**

Laravel Model 用單數（`Product` 而非 `Products`）。Permission 跟著單數，跨 layer 命名一致：

```
Model:       App\Models\Product
Policy:      ProductPolicy
Permission:  admin.catalog.product.access     ← 看到 permission 立刻對到 Model
Route URL:   /admin/catalog/products          ← URL 維持 RESTful 複數，不衝突
```

URL 用 RESTful 複數、Permission 用單數，兩套慣例各走各的、不互相牽動。

**2. 英文不規則複數很煩**

`category → categories`、`taxonomy → taxonomies`、`child → children`、`analysis → analyses`、`status → statuses`...選單數，永遠不用想複數規則。

**3. Wildcard 語意更乾淨**

```
admin.catalog.product.*    讀作「對 product 資源的所有動作」     ← 直覺
admin.catalog.products.*   讀作「對 products 集合的所有動作」？  ← 含混
```

單數版用 `*` 自然指向「對該資源類型的所有動作」，不會有「集合 vs 個體」的語意歧義。

## 3. Wildcard 權限：用 Spatie 原生 `*`

`config/permission.php` 啟用 `enable_wildcard_permission => true` 後，可以這樣指派：

```php
// HR 主管：MSS / Team / ESS 三個模組全開
$hrManager->givePermissionTo([
    'hrm.mss.*.*',     // MSS 模組所有資源、所有動作
    'hrm.team.*.*',
    'hrm.ess.*.*',
]);

// 一般員工：限定 ESS 個人操作
$employee->givePermissionTo([
    'hrm.ess.profile.access',
    'hrm.ess.profile.modify',
    'hrm.ess.attendance.modify',
    'hrm.ess.leave.modify',
]);
```

`*` 可以放在任何一段：

```
admin.catalog.product.*    # product 的所有 action
admin.catalog.*.*          # catalog 模組全部
admin.*.*.*                # 整個 admin Portal
```

> 不要自己再做 `admin.all` 之類的「全開」權限。Spatie 原生 `*` 已經涵蓋這個情境，多一層自訂邏輯只會增加維護成本。

## 4. 兩個全域角色：`super_admin` 與 `system`

業務角色（如 `admin.operator`、`hrm.hr_manager`）都帶 Portal 前綴。另外有兩個**全域角色**例外、不帶前綴，用途也跟一般業務角色不同：

| 角色 | 用途 | 後台可見 | 怎麼指派 |
|---|---|---|---|
| `super_admin` | 全系統最高權限 | 是 | 後台 UI 可指派 |
| `system` | 系統自動操作標記（cron / queue / service account） | 否 | DB / Seeder |

### `super_admin`：用 `Gate::before` 處理全系統最高權限

OCAdmin 用 Laravel 內建的 `Gate::before` 機制處理最高權限：

```php
// AppServiceProvider::boot()
Gate::before(fn ($user, $ability) =>
    $user->hasRole('super_admin') ? true : null
);
```

`Gate::before` 是 Laravel 的權限「前置攔截」——任何 `$user->can(...)`、`@can(...)`、`$this->authorize(...)` 到達正常檢查之前，這個 callback 先跑。**回 `true` 直接放行、回 `null` 才繼續走正常權限檢查流程**。

#### 為什麼用 `Gate::before` 而不是「指派所有權限給 super_admin」？

直覺做法是建一個 super_admin 角色，逐一賦予所有 permission。可行，但有缺點：

- **新增 permission 要記得補**：每次系統加新權限，super_admin 角色要 sync 一次；漏一次就出現「明明是最高權限卻 403」
- **權限可能被誤改**：所有權限都顯示在後台 role 管理頁、可以被誤手動取消勾選
- **初始化成本**：權限多時，要寫迴圈把所有 permission 加進去——多餘的 IO

用 `Gate::before` 一勞永逸：**super_admin 角色本身不用配任何權限**，Gate 層放行所有檢查。新增 permission 不用同步、不會被後台誤勾、初始化也不用跑迴圈。

> 初始 `super_admin` 帳號由 seeder 建立，之後可以在後台把 super_admin 角色指派給其他帳號。

### `system`：給「不是人」的帳號用

`system` 不是給「人」用的角色，是給「不是人」的帳號用——系統服務、cron job、queue worker。當這些自動程序在 audit log 留下操作紀錄時，會帶 `system` 角色，事後追查時一眼看出「這是系統做的，不是哪個管理員」。

```php
// 例如服務帳號跑批次更新
$serviceUser = User::where('username', 'service')->first();
// 它有 `system` role，操作會在 log 留下系統標記
```

這比「在 log 加一個 `is_system` 欄位」乾淨——權限系統本來就在追蹤「誰做了什麼」，順便擔任這個角色標記。

## 5. Prefix 解耦：`permName()` / `roleName()` Helper

權限名稱裡的 `admin` 那個 portal 段，**不直接寫進程式碼**。所有拼接走兩個 helper：

```php
function permName(string $suffix): string
{
    $prefix = config('portals.ocadmin.permission_prefix', 'admin');
    return "{$prefix}.{$suffix}";
}

function roleName(string $name, bool $isException = false): string
{
    if ($isException) return $name;   // super_admin / system 不加 prefix
    $prefix = config('portals.ocadmin.role_prefix', 'admin');
    return "{$prefix}.{$name}";
}
```

用法：

```php
permName('catalog.product.access')     // → 'admin.catalog.product.access'
roleName('order_operator')             // → 'admin.order_operator'
roleName('super_admin', true)          // → 'super_admin'（例外，不加 prefix）
```

### Seeder 內部：純業務 token

最關鍵的好處：**Seeder 內 array key 只寫後三段業務 token**，不含 prefix：

```php
$permissions = [
    'catalog.product.access' => '商品檢視',     // ← 純業務 token
    'catalog.product.modify' => '商品修改',
    'catalog.option.access'  => '選項檢視',
];

foreach ($permissions as $suffix => $name) {
    Permission::updateOrCreate(['name' => permName($suffix)]);
    //                                    ↑ runtime 拼接 prefix
}
```

### 為什麼這個解耦這麼重要？

**衍生專案改 `.env` 就換掉整套權限名稱**：

```env
# 衍生專案 A 的 .env
OCADMIN_PERMISSION_PREFIX=backend
# → permName('catalog.product.access') 自動變成 backend.catalog.product.access

# 衍生專案 B 的 .env
OCADMIN_PERMISSION_PREFIX=manage
# → permName('catalog.product.access') 自動變成 manage.catalog.product.access
```

Seeder 的業務 token 清單**完全相同**，不用為每個專案複製改名。所有「讀 prefix」的地方（middleware、Blade `@can`、Policy）都走 helper，沒有任何一處寫死 `'admin.'`。

如果 seeder 直接寫成 `'admin.catalog.product.access'`，跨專案重用就要 sed 全文取代——痛苦且容易遺漏。**抽掉 prefix、變成 runtime composition，是 portable seeder 的必要前提**。

這也呼應前一篇 [整體架構](/ocadmin/architecture/) 講的「`url_slug`、`role_prefix`、`permission_prefix`、`dir` 四個值刻意解耦」——核心都是同一件事：**讓設定可以單獨改、不會牽一髮動全身**。

## 6. 集中於 Route Middleware 的權限檢查

權限檢查（enforcement）刻意**集中在 route middleware**，不放 controller。

```php
// ✓ 推薦：route 一處宣告
Route::middleware(['permission:' . permName('catalog.product.access')])
    ->group(function () {
        Route::get('/products', [ProductController::class, 'index']);
        Route::get('/products/{product}/edit', [ProductController::class, 'edit']);
    });

// ✗ 不推薦：散落在 controller
public function index()
{
    if (!auth()->user()->can('admin.catalog.product.access')) {
        abort(403);
    }
    // ...
}
```

### 為什麼是 route 不是 controller

| 優點 | route 集中宣告 |
|---|---|
| **稽核** | 跑 `php artisan route:list` 一眼看到每條 route 掛了什麼 permission |
| **漏鎖檢測** | 漏寫一條 review code 時容易看出來 |
| **不污染 controller** | controller 專心做業務邏輯、不夾雜權限判斷 |
| **改 permission 不用改 controller** | 改 route 一處就好 |

Controller 內的 `$this->authorize()` 留給「**欄位級**」權限——例如員工資料頁要顯示薪資金額，但只有部分角色看得到 `salary` 欄位：

```php
public function show(Employee $employee)
{
    $data = $employee->toArray();

    if (!auth()->user()->can(permName('mss.payroll.access:salary'))) {
        unset($data['salary']);
    }

    return view(...);
}
```

權限段尾加 `:column` 的 suffix 標記欄位級權限。Route middleware 把粗粒度的「能不能進這個頁面」鎖住，欄位級的細節留給 controller 處理。

## 接下來

預定再寫：

- **列表頁規範**：篩選、排序、分頁、URL 參數保留的一致設計
- **雙模架構**：Ocadmin 兼任前台 vs 獨立 Web Portal 的取捨

權限的進階主題（Spatie 內建快取的問題、自訂角色組合快取的設計、Policy 怎麼跟資料權限整合）資訊量太大，是否抽獨立一篇看回饋決定。如果你看完想了解這幾個方向，留言告訴我。
