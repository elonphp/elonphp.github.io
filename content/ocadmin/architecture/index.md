---
title: "整體架構：Portal、Core、Module 怎麼切，為什麼這樣切"
date: 2026-05-24T22:30:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "架構設計", "Portal"]
categories: ["OCAdmin"]
weight: 2
summary: "OCAdmin 不是新框架，而是 Laravel + 一套架構約定。最重要的概念是 Portal——讓同一個專案同時服務後台、HR、門市、官網等不同對象。這篇講三件事：為什麼需要 Portal、Portal 之間共用什麼隔離什麼、Portal 內為什麼又分 Core 和 Modules。"
---

> [English version →](/ocadmin/en/architecture/)

OCAdmin **底層怎麼組織程式碼**——Portal、Core、Modules 三層架構的設計脈絡。

## 1. 先說清楚：底座是 Laravel

OCAdmin **不是另一個框架**。它就是一個 Laravel 專案，路由、認證、ORM、佇列、快取——全部交給 Laravel。

OCAdmin 自己的部分是什麼？**一套基於 Laravel 的架構約定**，主要解決一個問題：

> 同一個 Laravel 專案怎麼漂亮地同時服務「後台管理員」「HR / 員工」「門市銷售」「客戶官網」這些不同對象？

答案叫 **Portal**。

## 2. Portal：應用入口的抽象

把同一個 Laravel 專案想像成一棟大樓，**Portal 就是大樓的不同入口**：

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Ocadmin   │  │     Hrm     │  │     Web     │  │     POS     │
│   Portal    │  │   Portal    │  │   Portal    │  │   Portal    │
│             │  │             │  │             │  │             │
│ 公司內部後台 │  │ HR / 員工自助│ │  大眾官網    │  │ 門市銷售系統 │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
  role: admin.*    role: hrm.*      role: web.*      role: pos.*
```

每個 Portal 有自己的：

- **URL 前綴**（`/admin/...`、`/hrm/...`、`/pos/...`）
- **角色與權限**（`admin.user.edit`、`hrm.employee.access`、`pos.order.create`…）
- **技術棧**（後台用 Blade，門市可以用 Vue，官網可以另外選）
- **登入頁**（門市平板可以直接書籤 `/pos/login`）

底層共用同一份資料庫、同一批 Models、同一批 Services。

### 為什麼是 Portal，不是「多個 Laravel 專案」？

「每個 Portal 都自成一個 Laravel 專案」技術上可行，但實際開發會非常痛苦：

| 困擾 | Portal 架構解決 | 多專案會怎樣 |
|---|---|---|
| Models 跨業務領域共用 | 共用 `app/Models/` | 要嘛複製、要嘛抽 package、要嘛搞 microservice |
| 同一個使用者跨 Portal 操作 | 同一份 `users` 表 + Portal 角色判定 | 多套帳號系統、SSO 串接 |
| 同一個變更要同步 | 一個 PR 改完 | 三個 repo 三個 PR |
| 開發環境 | `php artisan serve` 起一台 | 起 N 台、port 對應 |

Portal 架構讓你**保留「一個 codebase」的協作便利**，同時擁有「N 個應用入口」的彈性。

### 需要 API？再開一個 Portal 就好

Portal 架構**不強迫**所有入口用同一種前端方式。每個 Portal 自己決定要走 Blade SSR 還是 SPA + API。

**Ocadmin Portal 選擇不做前後端分離**（用 Blade + jQuery + Bootstrap 5），原因：

- 後台是內部使用，沒有 SEO 需求
- 沒有 Mobile App 要共用 API
- 沒有獨立前端團隊要併行開發
- 表單表格密集、桌機為主——傳統 SSR 最有效率，多餘的 SPA 抽象反而增加複雜度

但**其他 Portal 可以走完全相反的路**。需要 Mobile App、第三方串接、或版本化 API 時，直接開新 Portal：

```
app/Portals/
├── Ocadmin/         ← Blade + jQuery + Bootstrap（後台，傳統 SSR）
├── WebApi/          ← 純 RESTful API（Mobile / 第三方）
├── WebApiV1/        ← 版本化 API：v1 維護中
└── WebApiV2/        ← 版本化 API：v2 新功能並存
```

每個 Portal 共用 `app/Models/` 和 `app/Services/`，差異只在「路由 / 視圖 / Middleware / 認證策略」這層。**新開一個 API Portal 不需要重構任何既有 Portal**——加一個資料夾、註冊一個 ServiceProvider 就完成。

這就是 Portal 架構真正的價值：**讓「我這個入口要 SSR」跟「我那個入口要 API」不再需要二選一**，也讓 API 版本控制（v1 / v2 並存）自然落在資料夾結構裡，不用在路由中靠 prefix hack。

## 3. Portal 之間共用什麼、隔離什麼

設計關鍵：**後端共用、前端隔離**。

| 資源 | 共用 | 為什麼 |
|---|---|---|
| `app/Models/` | ✓ | 業務實體只該有一份定義（User、Order、Product…） |
| `app/Services/` | ✓ | 跨 Portal 共用的業務邏輯（「建立訂單」可從後台 / 門市 / 官網觸發） |
| `app/Helpers/` | ✓ | 工具類無 Portal 屬性 |
| `database/migrations/` | ✓ | 資料庫只有一套 |
| `lang/` | ✓ | 多語檔集中管理 |
| Blade Views | ✗ | 每個 Portal 完全獨立 |
| 前端資源（JS / CSS） | ✗ | 各 Portal 各自的 Vite bundle |
| 路由檔 | ✗ | 每個 Portal 自己的 `routes/*.php` |

這個切法的直接好處：**後台的 Bootstrap 5 跟門市的 Vue 不會 CSS 撞 class，但兩邊都能 `User::find($id)`**。

## 4. config/portals.php：解耦的設定

每個 Portal 在 `config/portals.php` 註冊：

```php
return [
    'ocadmin' => [
        'url_slug'          => 'admin',          // → /{locale}/admin/...
        'role_prefix'       => 'admin',
        'permission_prefix' => 'admin',
        'dir'               => 'Ocadmin',
    ],
    'pos' => [
        'url_slug'          => 'pos',            // → /{locale}/pos/...
        'role_prefix'       => 'pos',
        'permission_prefix' => 'pos',
        'dir'               => 'Pos',
    ],
    'web' => [
        'url_slug'          => '',               // 空字串 → /{locale}/...（使用根目錄，不加 Portal 路徑段）
        'role_prefix'       => 'web',
        'permission_prefix' => 'web',
        'dir'               => 'Web',
    ],
];
```

> `permission_prefix` 通常跟 `role_prefix` 設成一樣，獨立欄位是預留彈性——少數情境會需要讓「進 Portal 的角色命名」跟「授權字串前綴」可以不同，9 成情況下兩者相同。

**`url_slug`、`role_prefix`、`dir` 這三個值刻意完全解耦**。看起來像沒必要，但實際用時很救命：

```php
// 想把 /admin 改成 /backend？
// 只動 url_slug，role_prefix 不動 → 既有授權完全不受影響
'ocadmin' => [
    'url_slug'    => 'backend',    // ← 只改這
    'role_prefix' => 'admin',
    'dir'         => 'Ocadmin',
],
```

或者更極端——大版本改版、新舊後台並存過渡：

```php
'ocadmin'    => ['url_slug' => 'ocadmin', 'role_prefix' => 'admin', 'dir' => 'Ocadmin'],
'ocadmin-v2' => ['url_slug' => 'admin',   'role_prefix' => 'admin', 'dir' => 'OcadminV2'],
```

兩個 Portal **共用同一套角色**（`admin.*`），但對應不同程式目錄、不同 URL。如果三者綁死（例如「`/admin/` 必然對應 `admin.*` 角色和 `Admin/` 資料夾」），這種過渡就做不到。

> 解耦看起來是「為了未來可能不會發生的事情多寫一點」，但實務上每次重大改名或重構都會用到。一次設計、長久受惠。

## 5. Portal 內部：Core 和 Modules 的兩種哲學

進到一個 Portal 內部（以 `app/Portals/Ocadmin/` 為例），會看到兩個頂層資料夾：

```
app/Portals/Ocadmin/
├── Core/                    ← 模板隨附的標準供應品
│   ├── Controllers/
│   ├── Services/
│   ├── Providers/
│   └── ViewComposers/
│
└── Modules/                 ← 各專案自有的業務模組
    ├── Catalog/
    │   └── Product/
    │       ├── ProductController.php
    │       └── ProductService.php
    ├── Member/
    └── Hrm/
```

**這兩個資料夾的組織方式完全不一樣**——`Core/` 是「按層分」（Controllers/、Services/、Providers/...），`Modules/` 是「按 feature 分」（每個業務一個資料夾，內含該功能所有 layer 的檔案）。

刻意的，原因如下：

| 觀察點 | Core/ | Modules/ |
|---|---|---|
| **內容** | 認證、權限、系統管理等模板隨附功能 | 各專案的業務（商品、訂單、HR…） |
| **規模** | 小（feature 數量有限） | 大（隨業務無限增長） |
| **修改頻率** | 低（穩定基底） | 高（持續開發新功能） |
| **典型修改** | 偶爾改一個 Provider 或 Controller | 開發一個 feature 同時動 Controller + Service + Request |
| **移植到別專案** | 不發生（Core 是共用基底） | 可能（feature 可重用） |

### Core/ 用 layer-grouped（Laravel 標準）

```
Core/
├── Controllers/
├── Services/
├── Providers/
└── ViewComposers/
```

跟 Laravel 慣例對齊，新人直覺找得到檔。feature 數量有限，沒有「資料夾爆炸」的問題。

### Modules/ 用 module-grouped（一個 feature 一個資料夾）

```
Modules/Catalog/Product/
├── ProductController.php
├── ProductService.php
└── Requests/
```

優點：

- **一個功能 = 一個資料夾**，找檔、刪檔、整個資料夾移植都直觀
- 開發一個 feature 時，相關的 Controller / Service / Request 都在同一處，不用在 `Controllers/` `Services/` `Requests/` 之間跳來跳去
- 想把某個 module 抽出去成獨立套件？直接整個資料夾搬

如果 `Modules/` 也用 layer-grouped，每個層底下會堆滿不相關的檔案（`Controllers/ProductController.php`、`Controllers/OrderController.php`、`Controllers/EmployeeController.php`…），開發體驗會很差。

### 為什麼不統一一種？

**因為兩邊的問題本來就不一樣**，沒必要為了「看起來一致」付出代價：

- 統一成 layer-grouped → `Modules/` 失去模組化的好處
- 統一成 module-grouped → `Core/` 每個 module 只有 1-2 個檔，組織反而散

我自己過去用標準 Laravel layer-grouped 寫業務功能時，最有感的痛點是**改一個功能要在不同層的資料夾之間反覆跳**：

```
改 Product 的編輯邏輯要碰：
app/Http/Controllers/Catalog/ProductController.php
app/Services/Catalog/ProductService.php
app/Http/Requests/Catalog/ProductUpdateRequest.php
```

三個檔散在三個層，每次都要先回到 `app/` 再往下挖一次，再回頭、再挖。Module-grouped 直接把它們收在同一個資料夾：

```
app/Portals/Ocadmin/Modules/Catalog/Product/
├── ProductController.php
├── ProductService.php
└── Requests/ProductUpdateRequest.php
```

差異**不只是「方便一點」**：你不用再記「這個功能的 service 在哪個 Services/ 子層」，因為它就在 Controller 旁邊。心智負擔的差異會在每天累積的小操作裡放大。

**「一致」不是設計目標，「合用」才是**。架構決策的本質是「為實際情況設計」，不是「為了好看排版」。

## 6. 接下來

後續會陸續展開：

- **多語系設計**：MVCL 的 L 在 OCAdmin 怎麼運作、lang 檔組織策略
- **權限機制**：角色 / 權限命名規範、Spatie 整合、Portal 隔離
- **列表頁規範**：篩選、排序、分頁、URL 保留的一致設計
- **雙模架構**：Ocadmin 兼任前台 vs 獨立 Web Portal 的取捨

如果你對哪個主題特別感興趣，留言告訴我可以調整寫作順序。
