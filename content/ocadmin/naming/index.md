---
title: "命名規範：橫向單複數 + 縱向階層對齊"
date: 2026-05-25T20:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "命名規範", "RESTful"]
categories: ["OCAdmin"]
weight: 4
summary: "OCAdmin 在很多地方都需要名字——URL、route name、permission、Model、Controller、Blade folder、DB table、變數。這篇集中規範兩件事：橫向（什麼時候用單數、什麼時候用複數）與縱向（同一個 resource 在各層的階層座標要對齊）。重點在元原則——學會了就能自己推，不用查表。"
---

> [English version →](/ocadmin/en/naming/)

OCAdmin 在很多地方都需要名字——URL、route name、permission name、Eloquent Model、Controller class、Blade folder、DB table、變數。**這些名字什麼時候該用單數、什麼時候該用複數，跨層之間要不要對齊**——這篇集中規範。

這個議題看起來瑣碎，但**早期確立、嚴格執行**省下的是「每次新增模組都要思考該怎麼命名」的反覆認知負擔，長期回報極大。

要回答的兩個問題：

1. **橫向**：同一個概念在 URL、程式、DB、view 各層該用單數還是複數？
2. **縱向**：同一個 resource 在各層的「階層座標」該怎麼對齊？

把這兩個問題講清楚，遇到新情境就能按規則推，不再每次思考。

## 1. 元原則：先想清楚「在描述什麼」

> **單數 = 描述「概念 / 類型 / 單一實例」**
> **複數 = 描述「集合 / 容器 / 多個實例」**

判斷一個名字該單數還是複數，問自己：**「這個名字描述的是什麼？」** 從**對象本質**回推，不要從「它擺在哪一層」反推。

### 三個語境

不同語境下，名字描述的對象不同：

| 語境 | 描述對象本質 | 規則 | 出現位置 |
|---|---|---|---|
| **OO / 程式內** | 概念、類型、namespace、單一物件 | **單數** | Model / Controller / Service / Policy / Module folder / **View folder** / Permission name resource 段 |
| **HTTP / RESTful** | 資源集合的對外位址 | **複數** | URL path / Route name resource 段 |
| **DB / 儲存層** | 存放多筆記錄的容器 | **複數** | Table 名 / Migration filename |

集合變數（如 `$users = User::get()`）走複數，是因為**它持有多個實例**——「對象本質」決定，不是「在哪個語境」決定。

## 2. 快速對照表

實際開發 90% 的情境都對得到這張表：

| 層 | 對象 | 規則 | 範例 |
|---|---|---|---|
| URL | URL path | **複數** | `/admin/system/users` |
| URL | Route name resource 段 | **複數** | `system.users.index` |
| 程式 | Permission name resource 段 | **單數** | `admin.system.setting.access` |
| 程式 | Eloquent Model | **單數** | `User`、`Product`、`Order` |
| 程式 | Controller class | **單數** | `UserController` |
| 程式 | Policy / Service | **單數** | `UserPolicy`、`OrderService` |
| 程式 | 單一物件變數 | **單數** | `$user`、`$product` |
| 程式 | 集合變數 | **複數** | `$users`、`$products` |
| DB | 資料表 | **複數** | `users`、`acl_roles` |
| DB | Foreign key | **單數 `_id`** | `user_id`、`product_id` |
| DB | Pivot table | **兩單數字母序** | `role_user`、`product_tag` |
| View | Blade 子目錄 | **單數**（跟 Controller 對齊） | `catalog/product/index.blade.php` |

> 邊界例外：**boolean 變數**（`$is_active`、`$has_permission`）描述狀態，無單複數問題；**不可數名詞**（`cache`、`data`）照英文本身慣例；**View namespace**（`ocadmin::`、`pos::`）採單數對應 Portal 名。

## 3. 最反直覺的爭議：URL 複數 vs Permission 單數

這是這套規範最反直覺的點——同一個 `User` 概念，URL 寫 `/users`、但 permission 寫 `admin.system.user.access`。看似不一致，但**兩套慣例服務不同目的**：

| | URL（複數） | Permission name（單數） |
|---|---|---|
| **設計來源** | RESTful（資源集合導向） | Eloquent / OO（class 命名對齊） |
| **意義** | 「這個路徑管理 user 這類資源**集合**」 | 「對 user 這個資源**類型**的權限」 |
| **業界共識** | Rails / Django / Laravel `Route::resource` 預設皆複數 | AWS IAM / GCP IAM 混用，無壓倒性偏好；Laravel 內部以 Model 單數命名為主 |
| **不規則複數風險** | URL 必須拼對英文複數（`taxonomies` 不是 `taxonomys`） | 永遠單數，完全規避 |

維護者習慣「**URL 複數 / 程式內單數**」這個分工後，切換時不需要思考。

### View folder 跟 URL 不一致——這也是刻意的

```
URL:         /admin/catalog/products              ← 複數
View folder: views/catalog/product/index.blade   ← 單數
```

View folder 屬於「OO / 程式內」語境——folder 描述「某個領域概念的視圖空間」，跟 `Modules/Catalog/Product/ProductController.php`、`app/Models/Catalog/Product.php` 的命名階層對齊；URL 是 HTTP 對外的資源集合（複數），是另一個維度。

「URL 複數 / 程式內單數」的分工**一致地**套用到 view folder。

## 4. 縱向對齊：同一個 resource，五層座標要一致

橫向（單複數）解決了，還有第二個問題：**同一個 resource 在不同層的「階層位置」也要對齊**。

每個 resource 在系統中有一個**身份座標** = `{module}/{resource}`。系統各層對這個 resource 的指代都用同一個座標——誰也不跳層、誰也不加層、誰也不換字。

### 五層座標對齊（以 `setting` 為例）

```
URL prefix:    /admin/system/settings
Route name:    lang.ocadmin.system.settings.index
Permission:    admin.system.setting.access
Module folder: Core/Controllers/System/SettingController.php
View folder:   views/system/setting/index.blade.php
```

主體 token 都是 `system / setting`，五層唯一的差異**只在 case 與單複數**。

### 為什麼一定要對齊？

**座標不對齊 = 維護者腦中要建多套對映表**：

- 看到 URL `/admin/orgs/dealers` 要查才知對應 route name
- 看到 route name 要查才知對應 view folder
- 看到 view folder 要查才知對應 module folder

座標對齊後，**知一處即知所有處**——看到 URL 直接反推 controller 位置、route name、permission、view 路徑，IDE 直接 navigate。

### 主體 token 不能變

case 可以不同（kebab-case / snake_case / PascalCase）、單複數可以不同（按 §2 表），但**主體 token 跨層拼字必須相同**。

複合詞跨層轉換對照：

| 概念 | URL / Route | Permission | Folder | 變數 |
|---|---|---|---|---|
| authorization plan | `authorization-plans` | `authorization_plan` | `AuthorizationPlan/` | `$authorization_plan` |
| image manager | `image-manager` | `image_manager` | `ImageManager/` | `$image_manager` |
| custom option name | `custom-option-names` | `custom_option_name` | `CustomOptionName/` | `$custom_option_name` |

✗ 反例：view folder 叫 `common/imgmanager/` 但 route 叫 `common.image-manager.*`——主體一個是 concat `imgmanager`，另一個是兩字 hyphenated `image-manager`，token 拼字不同。

✓ 修正：統一為 `image-manager` / `ImageManager/` / `image_manager`，五層 case 不同但 token 序列一致。

## 5. 四種典型違規

實際遇到的座標不對齊，幾乎都是這四種變形：

### 5.1 階層深度不對齊

```
Module folder:  Modules/System/Acl/UserController.php   (3 層)
Route:          system.users.*                          (2 層，跳過 acl)
View:           views/acl/user/                         (2 層，跳過 system)
```

三層各跳一層，認知成本最高。修正：要嘛全部 3 層（加 `acl`）、要嘛全部 2 層（移除 `acl`）。

### 5.2 階層位置不對齊（module 段有/無）

```
Route:          /admin/organizations               (root 之下，無 module)
Permission:     admin.organization.organization.* (多了一層 module)
Module folder:  Modules/Party/Organization/        (Party 是 module)
```

座標一個 `/organization`、一個 `organization/organization`、一個 `Party/Organization`，三層完全錯開。

修正：取一個 module 段名，五層全對齊。例如全用 `org`：URL `org/organizations` / Permission `admin.org.organization.*` / Module folder `Modules/Org/Organization/`。

### 5.3 主體 token 不一致

見上一節「主體 token 不能變」的 imgmanager 反例。

### 5.4 業務名 vs 技術名混用

```
Module folder:  Modules/Party/Organization/   ← Party 是技術 namespace
Route:          dealer.members.*              ← dealer 是業務語意
```

同一個 controller 上下層被兩個字代表——`Party` 跟 `dealer` 是不同 mental model 的字。

修正：擇一統一。**全技術**：`Modules/Party/Member/` + route `party.members` + view `party/member/`；**全業務**：`Modules/Dealer/Member/` + route `dealer.members` + view `dealer/member/`。

## 6. 一個很容易搖擺的範例：`Sale` 還是 `Sales`？

`Sale` 是這套規範裡**最容易搖擺**的字，因為英文裡 `sales` 不只是 `sale` 的複數，**還是業務語境下的獨立名詞**：

| 寫法 | 英文語感 | 在程式裡指什麼 |
|---|---|---|
| `a sale` / `the sale` | 單一交易 / 一個促銷活動 | 一筆銷售紀錄、一個促銷 entity |
| `sales` | 業務語境的「銷售」——the Sales team、sales report、sales pipeline | 業務領域 / 部門名稱 |

直覺反應是「Sales 模組聽起來合理啊，就是業務嘛」。這個拉力是真的，不是錯覺。但本規範仍選 `Sale/`：

1. **「網域概念」貼近 module folder 的角色**：folder 裡放的是 `Order` / `OrderProduct` / `Invoice`，每個都是「銷售網域裡的 entity」。folder 名作為這些 entity 的 namespace 前綴（`Sale\Order`），單數讀起來像「銷售（概念）下的 Order」；複數讀起來像「多筆 sale 下的 Order」反而怪
2. **跨層一致勝過個別字的英文直覺**：若 `Sale` 破例用 `Sales`，那 `Inventory` 該不該 `Inventories`、`Catalog` 變 `Catalogs`、`Org` 變 `Orgs`？最終變成逐字判斷。**一律單數最省心**
3. **沒有壓倒性業界共識**：Magento 用 `Sales` 確實存在，但 Rails / Django / 多數 Laravel 應用採單數 module 命名

**規避招式**：若真覺得 `Sale` 太單薄，換個更具體的 module 名繞開——直接叫 `Order/`（模組重心是訂單）、`Pos/`、`Checkout/`。**不要為了表達「業務領域」而把 module folder 改複數**，這會破壞跨層一致性。

## 7. 遷移順序：程式碼先動，URL 後動

舊專案要對齊本規範時，順序很重要：

```
1. 改 Module folder（namespace 跟 use 跟著改，IDE 自動 refactor）
   ↓
2. 改 View folder（view path string 跟著改，grep 一次取代）
   ↓
3. 改 Permission 字串 + seeder
   ↓
4. 改 Route prefix + name
   ↓
5. 改 URL（可選；對外服務考慮 301 redirect 舊 URL）
```

URL 改動**最後做**，因為涉及 SEO、外部連結、行動裝置書籤；前面四步全在程式碼內部，影響範圍可控。

> 用 `routePrefix()` 動態推算 route 前綴的 Controller，route 改完不需要逐一修改 `route()` 呼叫，反推邏輯自動適應新名稱——這是**降低遷移成本的關鍵**。

## 8. 接下來

接下來會展開：

- **權限機制**：四段式命名、Gate::before、Prefix 解耦
- **列表頁規範**：篩選、排序、分頁、URL 參數保留的一致設計
- **雙模架構**：Ocadmin 兼任前台 vs 獨立 Web Portal 的取捨

命名規範看起來瑣碎，但前期講清楚，後面所有命名問題都按規則推，再也不用思考——這是少數「投資一次、長期受益」的設計決策。
