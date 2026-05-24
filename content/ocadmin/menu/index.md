---
title: "選單機制：權限過濾、結構獨立、Code/DB 雙方案"
date: 2026-05-26T10:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "選單", "Sidebar", "ACL"]
categories: ["OCAdmin"]
weight: 7
summary: "後台 sidebar 要依使用者權限動態顯示，這聽起來簡單，但展開之後牽動兩個設計決策：選單權限要不要跟資料權限合在一起、選單樹要不要對齊程式結構。OCAdmin 用獨立的 *.menu action 解耦前者、刻意把選單樹排除在五層座標對齊之外解耦後者，並提供 Code Driven / DB Driven 兩套並存方案靠 MENU_DRIVER 切換。"
---

> [English version →](/ocadmin/en/menu/)

後台 sidebar 不能寫死——使用者只能看到自己有權限的選單項目。聽起來簡單，但展開之後牽動到兩個設計決策：**選單權限要不要跟資料權限合在一起**、**選單樹要不要對齊程式結構**。

## 1. 設計決策一：`*.menu` action — 選單可見性獨立於資料權限

直覺做法是「使用者有 `product.access`，就顯示產品選單」。這個合併聽起來合理，但實務上會卡住。

考慮這幾種情境：

- 管理員能**透過 URL 直接編輯**某個產品（有 access），但 sidebar 上**不想多出一項分散注意**
- 業務角色有 `product.access`（可看），但選單故意給隱藏，要他從特定流程進入
- 某個臨時功能上線，權限早就開了，sidebar 暫時還不想顯示

把選單可見性綁在資料權限上，這些情境都得多繞一圈——例如「假裝撤回 access、靠另一個 hack 控可見性」。

OCAdmin 把選單可見性拆成獨立的 `{resource}.menu` action：

| Action | 用途 | 由誰判定 |
|---|---|---|
| `{resource}.menu` | 選單是否顯示在 sidebar | MenuComposer |
| `{resource}.access` | 列表 + 單筆檢視（基礎資料權限） | Controller `authorize()` |
| `{resource}.access_all` | 看全部（不限 created_by） | Controller scope |
| `{resource}.modify` / `.delete` | 各操作 | Controller `authorize()` |

跟 [權限機制](/ocadmin/permissions/) 講的四段式命名一脈相承——`admin.catalog.product.menu` 跟 `admin.catalog.product.access` 是兩個獨立 permission，可以分別勾。

### 為什麼這樣解耦

- **語意分離**：UI 可見性 ≠ 資料權限，兩件事就該分兩個 flag
- **不寫隱式邏輯**：不在程式碼寫「access_all 蘊含 access」「access 蘊含 menu」之類的 fallback 推導，所有蘊含關係都靠 role 指派時明確勾選
- **後台 UI 一勾一件事**：每個 checkbox 各自獨立，意圖明確

當然 95% 的角色設計都是「有 access 也有 menu」。但**那 5% 的例外存在，就值得拆兩個欄位**——不然遇到時就要 hack。

## 2. 設計決策二：選單樹獨立於五層座標對齊

[命名規範](/ocadmin/naming/) 講過 OCAdmin 有「五層座標對齊」——URL prefix、route name、permission、module folder、view folder 五層在同一個 resource 上要對齊。

**選單樹刻意不算在內**。

### 為什麼

選單樹是 **UX 層**——分組邏輯反映「使用者腦中的心智模型」，跟程式結構的「自然分類」不一樣。

實際例子：`Taxonomy` / `Term` 兩個 resource 五層全對齊（`Core/Controllers/System/TaxonomyController.php`、`TermController.php` 兩段平鋪），**Controller 沒開中介層**。但 sidebar 把這兩條收進「詞彙管理」中介分組：

```
系統管理
├── 訪問控制（5 條 ACL link）        ← Controller 也有 Acl 中介層
├── 詞彙管理（2 條 Taxonomy/Term）   ← Controller 沒中介層
├── 參數設定
├── 資料表結構
└── 選單設定
```

「訪問控制」群組對應 Controller 的 `Acl/` 中介層，這是**巧合**——兩邊各自的開層條件都滿足（Controller 是因「五層對齊 + 多 sibling」、選單是因 UX 群組需要）。「詞彙管理」群組則只在選單存在——Controller / URL 都沒開中介層，純粹是 sidebar 為了視覺整齊把這兩條收一起。

判斷選單何時開中介層是**純 UX 考量**（同類 link ≥ 2、避免母選單下平鋪過多項影響掃描），**不會也不該帶動 Controller / view / URL / permission 同步開中介層**。

### 業界對照

選單樹獨立於底層結構，業界本來就是這樣做的：

- **GitHub Settings**：sidebar 約 15 分群，URL 全是 `/settings/{section}` 沒中間群段
- **VS Code 左側活動欄**：Explorer / Search / Source Control 是 UI 群組，不對應內部 API namespace
- **Magento Admin**：Marketing / Content / Reports 大分群，Controller 散在各 module 裡

選單分組是 UX 決策、不是程式結構決策——硬要對齊反而會逼出「明明 sidebar 該開中介層、結果為了對齊也得開資料夾」這種不必要的層數膨脹。

## 3. 父層自動隱藏：子項驅動、不用父權限

選單樹常有純分組節點（如「系統管理」、「商品型錄」），沒有對應的 URL、不對應單一 controller。**這種父層節點不需要設 permission**。

### 為什麼

考慮「商品型錄」分組，子項是「商品」「選項」「規格」三條，各自掛 `catalog.product.menu` / `catalog.option.menu` / `catalog.specification.menu`：

- 使用者有任一子項 menu 權限 → 「商品型錄」**該顯示**
- 三個都沒有 → 「商品型錄」**該隱藏**

如果為「商品型錄」單獨設一個 `catalog.menu` 父權限，要嘛它要永遠跟子項權限同步（自動化負擔），要嘛要管理員自己勾（多事還容易忘）。**兩者都比「沒有父權限」更糟**。

OCAdmin 的 MenuComposer 用「子項驅動」的遞迴過濾：

```php
protected function filterByPermission(array $item, $user): ?array
{
    // 1. 項目本身有 permission 且使用者沒權限 → 移除
    if (!empty($item['permission']) && !$user->can($item['permission'])) {
        return null;
    }

    // 2. 遞迴過濾子項
    if (!empty($item['children'])) {
        $item['children'] = collect($item['children'])
            ->map(fn ($child) => $this->filterByPermission($child, $user))
            ->filter()
            ->values()
            ->toArray();

        // 3. 純分組節點（href 空）+ 子項全被過濾光 → 父層也隱藏
        if (empty($item['children']) && empty($item['href'])) {
            return null;
        }
    }

    return $item;
}
```

關鍵在第 3 步：**父層 `href` 空 + `children` 也被過濾光 → 整個父層自動消失**。父層完全不需要 permission 欄位、不用設群組權限、不用任何同步邏輯，純靠子項向上推導。

`super_admin` 走 [權限機制](/ocadmin/permissions/) 講的 `Gate::before`，`can()` 永遠回 true，整個選單樹全部顯示——免去「super_admin 也要被選單過濾擋住」的特例處理。

## 4. 兩種方案：Code Driven vs DB Driven

選單「定義在哪」有兩個選擇。OCAdmin 範例專案**預設用 Code Driven 的單檔風格（A1）**。DB Driven 留作「需要 runtime 編輯選單」時切過去的進階選項。

### 方案 A — Code Driven（範例專案預設）

選單以 PHP 檔案定義、編譯期就決定結構。又分 A1（單檔，範例專案預設）與 A2（每模組一檔，模組多時的變體）兩種寫法——下面先講 A1。

#### A1：全 portal 一個檔（範例專案預設，OpenCart 風格）

OpenCart 4 把整個後台 sidebar 寫死在 `admin/controller/common/column_left.php` 一個檔——所有選單項目在同一個大 array 裡，按權限過濾後輸出。OCAdmin 範例專案沿用這個風格，整個 sidebar 直接寫進 `MenuComposer::buildMenus()`：

```php
// app/Portals/Ocadmin/Core/ViewComposers/MenuComposer.php
class MenuComposer
{
    protected function buildMenus(): array
    {
        $tree = [
            [
                'id'       => 'menu-catalog',
                'icon'     => 'fa-solid fa-tags',
                'name'     => '商品型錄',
                'href'     => '',
                'children' => [
                    ['name' => '商品', 'permission' => permName('catalog.product.menu'),
                     'href' => route('lang.ocadmin.catalog.product.index'), 'children' => []],
                    ['name' => '選項', 'permission' => permName('catalog.option.menu'),
                     'href' => route('lang.ocadmin.catalog.option.index'), 'children' => []],
                ],
            ],
            // 其餘模組以此類推
        ];

        return collect($tree)
            ->map(fn ($item) => $this->filterByPermission($item, auth()->user()))
            ->filter()
            ->values()
            ->toArray();
    }
}
```

不用 Menu class、不用註冊表，整個 sidebar 一檔到底。範例專案預設 A1 主要三個理由：

- **clone 即跑**：不依賴 DB，不用先 migrate / seed 才看得到 sidebar
- **單檔好懂**：改選單編一個 PHP 檔即可，不必摸 `sys_menus` schema
- **OpenCart 原型一致**：作為「以 OpenCart 為設計藍本」的系統，預設沿用原型最自然

#### A2：每模組一個 Menu class（模組多時的變體）

每個模組在自己目錄下定義 `XxxMenu.php`，回傳該模組的選單樹：

```php
// app/Portals/Ocadmin/Modules/Finance/FinanceMenu.php
class FinanceMenu
{
    public static function items(): array
    {
        return [
            'id'   => 'menu-finance',
            'icon' => 'fa-solid fa-coins',
            'name' => '財務會計',
            'href' => '',
            'children' => [
                [
                    'name'       => '付款',
                    'permission' => permName('finance.payment.menu'),
                    'href'       => route('lang.ocadmin.finance.payment.index'),
                    'children'   => [],
                ],
                // ...
            ],
        ];
    }
}
```

MenuComposer 註冊一個 class 陣列、依序收集：

```php
protected array $menuClasses = [
    CatalogMenu::class,
    InventoryMenu::class,
    MasterMenu::class,
    // ...
];
```

#### A1 vs A2 怎麼選

| | A1（單檔，範例專案預設） | A2（每模組一檔，模組多時的變體） |
|---|---|---|
| 看完整 sidebar | 一個檔到底 | 翻多個 class |
| 新增模組 | 在大陣列中插入一段 | 新增 class + Composer 註冊一行 |
| 刪除模組 | 要手動清理 array 對應段，容易漏 | 移除模組目錄即可，Composer 註冊也跟著移 |
| 模組 PR 變動 | 多個模組 PR 同時改大檔，git 容易衝突 | 該模組的 menu 變動跟 PR 在一起 |
| 移植成本 | 整檔複製，容易帶到不需要的項目 | 一個模組打包帶走 |
| 適合 | 模組少 / 已穩定（範例專案取向、OpenCart 取向） | 模組多、各自迭代節奏不同 |

範例專案預設 A1 是因為「易上手、模組數量可控」這個前提；衍生專案模組長到一定規模後再切 A2。兩者輸出格式相同、共用同一個 `filterByPermission()`，切換只是改寫資料來源、不動其他邏輯。

### 方案 B — DB Driven（進階：需要 runtime 編輯時才開）

選單存進 DB，跟權限表分離：

```
sys_menus                            sys_menu_translations
├── id                               ├── menu_id (FK)
├── portal                           ├── locale
├── parent_id                        └── display_name
├── permission_name ──→ 關聯權限表
├── route_name (動態生成 URL)
├── href (外部連結)
├── icon
├── sort_order
└── is_active
```

關鍵欄位：

| 欄位 | 用途 |
|---|---|
| `portal` | 區分不同 Portal 的選單（ocadmin / pos / www）共用一張表 |
| `parent_id` | 自我關聯，null = L1 根節點 |
| `permission_name` | 指向 `acl_permissions.name`；null = 純分組節點 |
| `route_name` | 走本系統路由（含 locale prefix） |
| `href` | 外部連結，跟 `route_name` 二選一 |
| `is_active` | 軟下線（呼應 [參數設定](/ocadmin/settings/) 的 is_active 設計） |

**適合的情境：**
- 後台需要 UI 編輯選單（拖拉排序、改 icon、停用某項）
- 不同部署實例要有不同選單組合（多租戶風格）
- 翻譯走 DB 而非 PHP 陣列（透過 `sys_menu_translations` 多語）

### 為什麼預設 A1 而不是 B

OCAdmin 是**範例專案**——給人 fork 來開新專案，預設值要對齊「clone 即跑」原則：

- DB Driven 要先 migrate + seed 才看得到 sidebar，上手摩擦大
- 多數衍生專案 sidebar 一旦定好幾年不大改，runtime 編輯是少數需求（SaaS、多租戶）
- OpenCart 原型沒走 DB Driven，預設沿用最符合本系統定位

切換方式：

```env
MENU_DRIVER=code       # 預設，從 MenuComposer hardcoded array 讀取
# 或
MENU_DRIVER=database   # 切到 sys_menus 表
```

兩套 MenuComposer 走不同來源，但**輸出格式相同**、**權限過濾邏輯相同**（同一個 `filterByPermission()`）——切換成本極低，只是改 `.env`。`MenuController` / `MenuTreeController` / `MenuSeeder` 全部保留，需要 runtime 編輯選單時切過去就好。

這個並存設計呼應 [整體架構](/ocadmin/architecture/) 講的「衍生專案應該能輕鬆改換策略」——預設選對 majority，少數情境留逃生門。

### `sys_menus` 跟 `acl_permissions` 為什麼不合併

DB Driven 還有一個關鍵決定：**選單表跟權限表保持分離**，靠 `permission_name` 字串軟關聯。

直覺做法是把選單欄位（icon、href、sort_order）加進 `acl_permissions` 表。看起來省一張表，但有缺點：

- `acl_permissions` 多了一堆「跟權限本身無關」的 UI 欄位
- Spatie 的 `hasPermissionTo()` / `can()` 撈到的物件帶一堆無用欄位
- 純分組節點（無 permission）強塞進權限表，語意奇怪

兩張表各司其職：`acl_permissions` 管「能不能做」、`sys_menus` 管「怎麼呈現」。中間靠 `permission_name` 字串對應，極簡解耦。

## 5. 多 Portal 共用同一張選單表

`sys_menus.portal` 欄位讓多個 Portal 共用同一張表：

| portal | 應用 | 用途 |
|---|---|---|
| `ocadmin` | 後台管理 | 完整 sidebar |
| `pos` | POS 系統 | POS 功能列 |
| `www` | 官網 | 前台導覽列 |

每個 Portal 的 MenuComposer 只查自己 portal 的列：

```php
Menu::where('portal', 'ocadmin')->whereNull('parent_id')->...
```

不同 Portal 完全獨立、不互相干擾。同一個權限（如 `admin.catalog.product.menu`）可以被多個 Portal 的選單節點引用——回到 §1 講的「permission 是條件、選單節點是呈現」這個關係，自然支援。

## 6. 接下來

兩個值得展開的相關主題：

- **角色組合快取**：`super_admin` 走 `Gate::before` 已經很快，但其他角色每 request 撈 user permissions 的成本如何降到 0 query。MenuComposer 每個 request 都跑 `can()`，這個快取直接影響 sidebar 渲染速度
- **設定多層覆寫**：[參數設定](/ocadmin/settings/) 預告的「品牌層 / 門市層」如何不污染 `sys_settings`

選單機制本身的進階主題（前端 sidebar 收合狀態、active link 高亮、麵包屑生成）多半是 view layer 的事，未必值得獨立一篇——看回饋決定。
