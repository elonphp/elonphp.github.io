---
title: "架構分層職責：業務分歧 vs 資料編排兩軸獨立"
date: 2026-05-27T10:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "架構設計", "Service", "Repository"]
categories: ["OCAdmin"]
weight: 8
summary: "OCAdmin 的分層把『業務分歧』跟『資料編排』拆成兩個獨立軸——Service 處理 Portal 內業務流程（catering 派送 vs retail 找零各自不同），Repository 處理 entity 多表 CRUD（跨 Portal 共用）。同 entity 的多 Controller（OrderController + OrderPaymentController）共用同一個 Service。這篇講為什麼這樣切、各層職責、跟 OpenCart 前後台複製碼設計的對照。"
---

> [English version →](/ocadmin/en/layered-architecture/)

OpenCart 寫法：前後台各自有 OrderModel、邏輯 90% 相同 10% 不同；一個 raw SQL 的 `addOrder()` 處理多張表。聽起來合理，但實務痛點明顯——後台改 tax 計算邏輯、前台沒同步改，bug 飛起來。Laravel 給的工具不一樣（Eloquent、Scope、DI），OCAdmin 借這些工具設計分層、**不照抄 OpenCart 的歷史包袱**。

核心觀念一句話：**把「業務分歧」跟「資料編排」拆成兩個獨立軸**。

具體落到三個角色：

- **Module Services**（`app\Portals\{Portal}\Modules\...\{Entity}Service.php`）：**內部業務編排**，按 Module 分、放在 Portal 裡面（catering / retail 各 Portal 各自一套）
- **Global Services**（`app\Services\{Entity}Service.php`）：**串接外部服務的業務介面**，跨 Portal 共用（發票、簡訊、金流通知統一介面）
- **Global Repositories**（`app\Repositories\{Entity}Repository.php`）：**共用的內部資料編排**（entity 多表 CRUD），跨 Portal 共用

整個分層就是這三個位置撐起來。後續逐個展開。

## 1. 兩軸觀：為什麼不能壓成一軸

我們以前的做法把所有分層判斷壓在一個 Service 軸上：「**跨 Portal 共用**就抽 `app\Services\`、單 Portal 就 Controller 直接打 Eloquent」。這個方向在簡單情境下夠用，但遇到下面兩個情況就破功：

### 情況 A：多 Portal 業務本質不同

假設一個餐飲商既做團體餐配送、也做零售門市，需要共用採購進貨與財務會計。系統開三個 Portal：**Ocadmin**（後台管理）、**PosCatering**（團體餐）、**PosRetail**（零售）。下文範例都基於這個假設情境。

catering 跟 retail 都建 order，但業務本質不同：

- catering：算配送費、validate 地址、配送時段配對
- retail：找零、即時開發票、店內庫存即時扣

兩邊業務流程完全不同。如果硬要塞同一個 `app\Services\OrderService`，得在 method 內 `if (portal == 'catering') { ... } else { ... }`——這就是把分歧壓到一個 class 的代價。

### 情況 B：同 entity 多表 CRUD 共用

一張 order 關聯多張表：`orders`、`order_products`、`order_options`、`order_payments`、`order_shipping_infos`。**這部分 catering 跟 retail 完全一樣**——都要在 transaction 內 sync 這些表。

如果為了「業務分歧」逼出 portal-scoped Service，多表 CRUD 就會被迫各 Portal 複製一份 → 回到 OpenCart 的複製碼痛點。

### 解法：拆兩軸

| 軸 | 處理什麼 | 用哪一層 | 跨不跨 Portal |
|---|---|---|---|
| **業務分歧軸** | Portal 特有的業務 rule（派送費、找零、發票時機）| **Service** | 不跨——每個 Portal 自己 |
| **資料編排軸** | 同 entity 的多表 CRUD（sync products / options / payments）| **Repository** | **跨**——所有 Portal 共用 |

兩軸獨立判斷、各自 evolve。

```
catering OrderService    retail OrderService    其他 Portal OrderService
        ↓                       ↓                        ↓
              ┌─────── OrderRepository ───────┐
              │   saveWithRelations()         │
              │   syncProducts() / syncPayments() ...
              └─────────────────┬─────────────┘
                                ↓
                              Models
```

業務分歧落在上層 Service、資料編排落在下層 Repository——**業務 evolve 不影響 CRUD、CRUD 改不影響業務**。

## 2. Service：業務編排層

Service 有兩個放置位置、處理不同對象：

- **Module Service**（`app\Portals\{Portal}\Modules\{Module}\{Entity}\`）：**內部業務編排**，按 Module 分、放在 Portal 裡面。知道自己屬於哪個 Portal、處理 Portal 特有的業務 rule（catering 派送費、retail 找零）。多表 CRUD 不歸它管——交給 Repository。**本節主要討論的就是這個**。
- **全域 Service**（`app\Services\` flat）：跨 Portal 共用的對外串接業務介面，**不知道 Portal**。例如 `InvoiceService` 包裝多家發票廠商、`NotificationService` 包裝多種通知管道——呼叫端拿到「開發票」「發通知」這種業務動作，不知道用哪家廠商。Module Service 跟 Controller 都可以注入全域 Service。

> **Service 是業務介面**（開發票、發通知），**Integration 是廠商實作**（EcpayClient、Every8dClient）。換廠商只動 Integration、Service 介面不變。全域 Service flat 結構、按需開即可、規則簡單，不另外展開。

下文聚焦 **Module Service** 的規則。

### 粒度規則（原則）

一個 Portal 內、同一業務 entity **原則上對應一個 Service**：

- 路徑：`app\Portals\{Portal}\Modules\{Module}\{Entity}\{Entity}Service.php`
- 主 Controller 與**同 entity 的 sub-resource Controller** 共用同一個 Service

例：

```
app\Portals\PosCatering\Modules\Sale\Order\
├── OrderController.php           # 主 Controller
├── OrderPaymentController.php    # sub-resource Controller，注入 OrderService
├── OrderShippingController.php   # sub-resource Controller，注入 OrderService
└── OrderService.php              # Portal + 模組 + entity 的業務編排
```

「原則」表示有例外，**不是「一律」**——這樣的措辭給規範留逃生門。

### 「同 entity」怎麼判定

新人會問「OrderPaymentController 算同 entity 嗎？」靠下列 marker 判斷：

| Marker | 符合 → 同 entity |
|---|---|
| URL sub-resource 結構 | `/orders/{order}/payments` → payment 是 order 的 child |
| DB belongsTo 關聯 | `OrderPayment` schema `belongsTo Order` → payment 是 order 的 child |
| 業務 invariant 連動 | 改 payment 會影響 order 狀態（fully_paid → status='paid'）|

核心 marker：**業務動作的 boundary 一致** → 同 entity；**boundary 不同** → 各自 Service。

### Service 流程典型樣態

```php
namespace App\Portals\PosCatering\Modules\Sale\Order;

class OrderService
{
    public function __construct(
        private OrderRepository $orderRepo,
        private SmsClient $sms,
    ) {}

    public function checkout(array $data, User $user): Order
    {
        // 1. Portal 業務 rule（catering 特有）
        $this->validateDeliveryAddress($data['shipping']);
        $data['delivery_fee'] = $this->calcDeliveryFee($data['shipping'], $data['items']);

        // 2. 多表 CRUD 編排 → 交給 Repository（跨 Portal 共用）
        $order = $this->orderRepo->saveWithRelations(new Order(), $data, $user);

        // 3. Portal 業務後續（catering 特有）
        $this->dispatchToDeliveryQueue($order);
        $this->sms->send($order->customer->mobile, "訂單 {$order->code} 已確認");

        return $order;
    }
}
```

retail 版本完全不同：

```php
namespace App\Portals\PosRetail\Modules\Sale\Order;

class OrderService
{
    public function checkout(array $data, User $user): Order
    {
        $this->validateInStoreInventory($data['items']);
        $data['change_amount'] = $this->calcChange($data['paid'], $data['total']);

        $order = $this->orderRepo->saveWithRelations(new Order(), $data, $user);

        $this->issueReceiptImmediately($order);
        $this->deductStoreStock($order);

        return $order;
    }
}
```

兩個 Service 業務天差地遠、但都靠**同一個** `OrderRepository::saveWithRelations()` 把資料寫進多表。**這就是兩軸獨立的最大價值**：業務分歧自然隔離 + 資料層共用零複製。

### 何時開 Service

不是每個 Controller 都要有 Service。**任一條件才建檔**：

1. 多 Model 編排（建單 sync products / options / payments）
2. Portal 業務分歧
3. 同 entity 多 Controller 共用
4. 跨 HTTP context 重用（Queue Job、排程）
5. Controller 單一方法超過 ~100 行的編排

**單 Portal 簡單 CRUD 不需 Service**。但**單 Portal 也允許開 Service**（規範用「原則」不用「一律」）——一個操作要 sync 多張表時開 Service 避免 fat controller，比死守「跨 Portal 才開」的硬規則務實。

## 3. Repository：entity 多表 CRUD 編排

**這層按需才開、大部分情況不需要 Repository**。簡單 CRUD 用 Eloquent 直接打就好（`$model->fill()->save()`），單表寫入也不該抽 Repository。**只在「entity 跨多張表、需要 transaction 編排」時才開**——例如建單同時 sync products / options / payments / shipping_infos 這類情境。為了「分層乾淨」而抽 Repository 會落入過度設計、回到 query-wrapper Repository 那種反模式。

Repository 是「**entity 多表 CRUD 編排層**」——管 entity 跨多張表的 transaction、**不知道自己被哪個 Portal 呼叫**、不做 Portal-specific 業務 rule。

跟典型 Repository pattern（`$repo->find()` 包 `Model::find()`）不一樣——**我們只用 Repository 處理多表編排**、**不做 query wrapping**。

### 位置：flat in `app\Repositories\`

不分 Portal、不分 Module（因為 Repository 本來就跨 Portal 共用）：

```
app\Repositories\
├── OrderRepository.php
├── ProductRepository.php
├── PurchaseRepository.php       # 採購進貨（跨品牌共用）
└── MaterialRepository.php       # 料件共用
```

### 典型樣態

```php
namespace App\Repositories;

class OrderRepository
{
    public function saveWithRelations(Order $order, array $data, User $user): Order
    {
        return DB::transaction(function () use ($order, $data, $user) {
            $order->fill($data);

            if ($order->exists) {
                $order->updated_by = $user->id;
            } else {
                $order->code = $this->generateCode();
                $order->created_by = $user->id;
            }
            $order->save();

            $this->syncProducts($order, $data['products'] ?? []);
            $this->syncShippingInfo($order, $data['shipping'] ?? null);
            $this->syncPayments($order, $data['payments'] ?? []);

            return $order;
        });
    }

    protected function syncProducts(Order $order, array $items): void { /* ... */ }
    protected function syncShippingInfo(Order $order, ?array $shipping): void { /* ... */ }
    protected function syncPayments(Order $order, array $payments): void { /* ... */ }
}
```

### Repository ↔ Service 依賴方向

| 規則 | 為什麼 |
|---|---|
| Service → Repository | Service 編排業務流程、Repository 是它的 CRUD 工具 |
| **Repository 不能呼叫 Service** | Repository 應該不知道 Portal、保持 Portal 無關 |
| Repository → 其他 Repository | 跨 entity 的多表 transaction（如 Order 內也要寫 Inventory）|
| Service → Model（不經 Repository）| 簡單單表動作不需要 Repository |

**單向依賴**是這個分層的核心約束——Repository 一旦知道 Portal，跨 Portal 共用就破功。

### 多品牌共用範例

假設同一個系統服務兩個獨立品牌（BrandA 跟 BrandB），共用採購進貨、財務會計、料件主檔，但各品牌的訂單流程、行銷活動獨立。分層佈局如下：

```
app\Portals\
├── PosCateringBrandA\Modules\Sale\Order\OrderService.php   # 品牌 A 訂單編排
├── PosCateringBrandB\Modules\Sale\Order\OrderService.php   # 品牌 B 訂單編排
├── OcadminBrandA\Modules\Purchase\PurchaseController.php   # 品牌 A 採購管理
└── OcadminBrandB\Modules\Purchase\PurchaseController.php   # 品牌 B 採購管理

app\Repositories\
├── OrderRepository.php       # 兩品牌訂單的多表 CRUD
├── PurchaseRepository.php    # 採購進貨共用
├── MaterialRepository.php    # 料件共用
└── AccountingRepository.php  # 財務會計共用
```

**業務分歧落在 Module Service**：兩品牌的訂單流程可以完全不同（不同的 SOP、不同的優惠規則、不同的通知模板）。
**資料編排落在 Repository**：兩品牌 order 寫入多表的邏輯一致，共用同一份 Repository。

兩軸獨立——BrandA 改訂單流程不影響 BrandB、共用的採購 CRUD 改一處兩品牌一起 evolve。這是「Repository 跨 Portal 共用」最具體的範例。

## 4. 為什麼不採用 query-wrapper Repository（但接受 entity-orchestration Repository）

「Repository」一詞在 Laravel 圈有惡名——我們以前也明確拒絕過。要釐清的關鍵是：實務上資料操作有三種運作方式，我們的 Repository **只負責其中一種**。

### 讀寫資料的三種運作方式

| 運作方式 | 範例 | 走哪裡 | 經 Repository？|
|---|---|---|---|
| **(1) 單筆 / 單表操作** | `Order::find($id)`、`$order->save()` | 直接用 Eloquent | 不經 |
| **(2) 領域查詢（含複雜過濾）** | `Order::filterByPhone($phone)->paginate()` | Model Scope（領域知識集中在 Model）| 不經 |
| **(3) 多表 transaction 編排** | `$orderRepo->saveWithRelations(...)` | Repository | **經**（entity-orchestration）|

**Repository 只管第 (3) 種**。(1) (2) 走 Eloquent / Scope，呼叫端直接用、不要經過 Repository 中介。

### Query-wrapper Repository 是反模式（拒絕）

「Query-wrapper Repository」就是把 (1) (2) 也包進 Repository 物件、提供 `$repo->find()` / `$repo->findActive()` 之類包裝 API——這正是我們拒絕的反模式：

| 反模式（拒絕） | 我們的做法 |
|---|---|
| `$orderRepo->find($id)` 包 `Order::find($id)` | 直接 `Order::find($id)` |
| `$orderRepo->findActive()` 包 `Order::where('is_active', true)->get()` | Model Scope：`Order::active()->get()` |
| `$orderRepo->findByPhone($phone)` 包 Scope | Model Scope：`Order::filterByPhone($phone)->...` |

**理由**：Laravel Eloquent 本身就是 Active Record + Query Builder——`Order::find($id)` 已經夠簡潔可讀，再包一層 `$repo->find($id)` 沒有任何增益。只增加一層間接性、新增查詢要改三處（Controller / Service / Repository），而且常出現「同一個 Controller 有時走 Service、有時直接走 Repository」的不一致。

### Entity-orchestration Repository（接受）

我們的 Repository 不做 `find()` 那種 pass-through。它做的是**第 (3) 種**——多表 transaction 編排，這在 Eloquent 上層、是 ORM 沒幫你解的問題：

```php
$orderRepo->saveWithRelations($order, $data, $user);
// → DB::transaction → fill + save + syncProducts + syncShipping + syncPayments
```

這實際上是「Domain Service 用 Repository 之名」——名實有點不符（Fowler 原意的 Repository 是 collection-like query abstraction）。但**團隊熟悉這個用法**，保留名字、文件定義清楚就好。

### 我們最終採用的分層

```
✓ Controller → Model                              （簡單 (1)）
✓ Controller → Model::Scope                       （領域查詢 (2)）
✓ Controller → Service → Repository → Model       （業務 + 多表 (3)）
✓ Service → Integration Client                    （外部 API）
✓ Service → Supports / Helpers                    （純運算 helper）
```

**Repository 不做 query wrapping、只做 entity 持久化編排**——重新定義名字的職責邊界，留下我們需要的部分（多表 transaction 編排）、拒絕反模式的部分（query wrapping）。

## 5. Supports vs Helpers：純運算 utility

`app\Services\` reserve 給 Portal 業務編排。但有些東西**跟 Portal 無關、不操作 DB、純運算**——像是給 order 算配送費的 `ShippingFeeCalculator`、給 OrderProduct 產出標籤資料的 `OrderLabelFormatter`、給 SKU 產出 barcode 的 `BarcodeBuilder`。

這些東西叫 `Service` 會跟 Module Service 混淆。我們用兩個目錄區分：

| | `app\Helpers\` | `app\Supports\` |
|---|---|---|
| 內容性質 | 橫切技術工具，**無 domain knowledge** | domain 純運算，**認 entity / business rule** |
| 引用的東西 | 只認 raw data type（string、date、array）| 認 Model、enum、domain 概念 |
| 跟業務關係 | 跟系統業務無關，換到別專案也能用 | 跟業務緊密綁定 |
| 典型成員 | DateHelper、StringHelper、UrlHelper | OrderLabelFormatter、TaxCalculator |
| 內部分類 | flat 或按 raw type 分 | 按業務 domain 分子目錄 |

**判斷規則**：「這個東西有沒有引用 Model 或 domain enum？」

- 沒引用 → `app\Helpers\`（橫切技術工具）
- 有引用 → `app\Supports\{Domain}\`（domain 純運算）

```
app\Supports\
├── Sale\
│   ├── OrderLabelFormatter.php
│   ├── ShippingFeeCalculator.php
│   └── TaxCalculator.php
├── Catalog\
│   └── BarcodeBuilder.php
└── Hr\
    └── LeaveBalanceCalculator.php
```

檔名用 `XxxFormatter` / `XxxBuilder` / `XxxCalculator` 後綴——讓讀者一眼看出職責，不會跟 Service 混淆。

## 6. 各專案規模的適用對照

新分層的通用性涵蓋多種專案類型：

| 專案類型 | 怎麼吃 |
|---|---|
| **單 Portal 簡單專案** | 多數 Controller 直接打 Model；單一 Controller 操作多表時開 Service |
| **多 Portal 同業務**（Ocadmin + POS 做同件事）| Repository 共用 + Service 各自 Portal（甚至業務一致時 Service 不需開）|
| **多品牌共用部分**（兩品牌共用採購 / 財務 / 料件）| 採購 / 財務 / 料件走 Repository 共用；訂單 / 行銷各品牌 Portal 自己的 Service |
| **多 Portal 業務分歧**（PosCatering vs PosRetail）| Service 隔離業務、Repository 共用 entity 持久化 |

**我們以前的做法把所有「共用 vs 不共用」壓到 Service 軸上判斷**——當業務分歧大時就破功。**新分層拆兩軸**，每軸獨立 evolve。

## 7. 速查：什麼放哪

| 你要做 | 放哪 |
|---|---|
| 簡單 list / single read | Controller + Model Scope |
| 多表 transaction 寫入 | Repository |
| Portal 特有業務流程 | Service |
| 同 entity 多 Controller 共用編排 | Service（共用同一個）|
| 純運算（認 entity）| `app\Supports\{Domain}\` |
| 純運算（不認 entity）| `app\Helpers\` |
| 外部 API 呼叫（廠商 client）| `app\Integrations\` |
| 跨 Portal 共用對外串接業務介面（發票、通知統一 API）| `app\Services\` flat |
| 表單預設值（純欄位）| `Model::$attributes` 或 `Model::defaults()` |
| 表單預設值（需 DB 查詢）| `Repository::newModel()` |
| 跨品牌共用採購 / 財務 / 料件 | Repository（flat）|

## 8. 一些設計細節背後的權衡

幾個容易被質疑的選擇：

**「原則上一個 entity 一個 Service」用「原則」不用「一律」**——規範措辭刻意留逃生門。純 read-only Controller、Service 膨脹到要拆、sub-resource 演化成獨立 entity 都是允許的例外。**寫「一律」會逼出 hack 來繞規定、寫「原則」反而能讓規範長久好用**。

**Repository 不能呼叫 Service**——這條約束很重要。一旦違反，Repository 就知道 Portal、跨 Portal 共用立刻破功。約束在「禁止依賴反轉」這件事上，比看似的好處（少寫一行 import）值得多。

**Service 跟 Portal 內 Module 結構同層**（同 entity 目錄、跟 Controller 並列）——這跟我們以前把 Service 放 `app\Services\` flat 結構的做法不一樣。同目錄的好處：看到 OrderController 就知道 OrderService 在隔壁，不用跨資料夾找。代價：跨 Portal 不會誤共用 Service（這正是我們要的）。

## 結語

OCAdmin 借 OpenCart 的 UI 設計、但**不繼承它的程式結構限制**。OpenCart 受限於 raw SQL 沒有 ORM，前後台只能各自複製一份 Model；Laravel 有 Eloquent、Scope、DI，這些工具讓「業務分歧（Service）」跟「資料編排（Repository）」可以拆兩軸獨立 evolve。

用「**兩軸**」思考比一軸線性判斷更貼近實務的設計問題。Service 軸關心的是業務分歧（不同 Portal 業務本質是否不同），Repository 軸關心的是資料編排（多表 CRUD 是否複雜到需要集中）。**兩軸獨立**——同一個 entity 在不同 Portal 業務不同、但多表 CRUD 共用——這是大多數實務專案的真實樣態。

完整規範文件：內部 `docs/common/10016_架構分層與Model職責.md`。這篇文章只挑重點講決策脈絡，細節（命名規則、Service 不該做的事清單、Repository 觸發條件全表）以規範文件為準。
