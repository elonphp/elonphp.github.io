---
title: "Layered Architecture: Two Independent Axes — Business Divergence vs Data Orchestration"
date: 2026-05-27T10:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "Architecture", "Service", "Repository"]
categories: ["OCAdmin"]
weight: 8
summary: "OCAdmin's layering splits 'business divergence' and 'data orchestration' into two independent axes — Services handle Portal-specific business flow (catering delivery vs retail change), Repositories handle entity multi-table CRUD (shared across Portals). Multiple Controllers for the same entity (OrderController + OrderPaymentController) share one Service. This post explains why we split this way, each layer's responsibility, and the contrast with OpenCart's front/back-office duplication."
build:
  list: local
---

> [→ 繁體中文版](/ocadmin/layered-architecture/)

OpenCart's approach: front-office and back-office each have their own OrderModel — logic is 90% identical and 10% different; a raw-SQL `addOrder()` handles multiple tables. Sounds reasonable, but the pain in practice is obvious — change the tax calculation in the back-office, forget to sync the front-office, bugs everywhere. Laravel gives different tools (Eloquent, Scope, DI), and OCAdmin uses these tools to design its layering — **without inheriting OpenCart's historical baggage**.

The core idea in one sentence: **split "business divergence" and "data orchestration" into two independent axes**.

Concretely, three roles:

- **Module Services** (`app\Portals\{Portal}\Modules\...\{Entity}Service.php`): **internal business orchestration**, organized by Module and placed inside the Portal (catering / retail each have their own per Portal)
- **Global Services** (`app\Services\{Entity}Service.php`): **business interfaces wrapping external integrations**, shared across Portals (unified invoicing / SMS / payment-notification APIs)
- **Global Repositories** (`app\Repositories\{Entity}Repository.php`): **shared internal data orchestration** (entity multi-table CRUD), shared across Portals

The entire layering rests on these three locations. The rest of this post unfolds each.

## 1. The two-axis view: why you can't collapse them into one

We used to put all layering decisions on a single Service axis: "**shared across Portals** → extract into `app\Services\`; single Portal → Controller calls Eloquent directly". This works in simple scenarios but breaks in two situations:

### Case A: Different Portals have fundamentally different business

Imagine a food-and-beverage business that runs both group-catering delivery and retail in-store sales, and needs to share purchasing and accounting across them. The system has three Portals: **Ocadmin** (back-office), **PosCatering** (group catering), **PosRetail** (retail). Examples throughout this post are built on this scenario.

Both catering and retail create orders, but the business is fundamentally different:

- catering: calculate delivery fees, validate addresses, match delivery time slots
- retail: calculate change, issue receipts immediately, deduct in-store stock in real time

Two fundamentally different business flows. Forcing both into the same `app\Services\OrderService` requires `if (portal == 'catering') { ... } else { ... }` — the cost of compressing divergence into one class.

### Case B: Shared multi-table CRUD for the same entity

An order links to many tables: `orders`, `order_products`, `order_options`, `order_payments`, `order_shipping_infos`. **This part is identical between catering and retail** — both need to sync these tables in a transaction.

If "business divergence" forces portal-scoped Services, the multi-table CRUD has to be copy-pasted across Portals → back to OpenCart's duplication pain.

### Solution: split into two axes

| Axis | Handles | Layer | Cross-Portal? |
|---|---|---|---|
| **Business-divergence axis** | Portal-specific business rules (delivery fee, change, receipt timing) | **Service** | No — each Portal has its own |
| **Data-orchestration axis** | Multi-table CRUD for the same entity (sync products / options / payments) | **Repository** | **Yes** — shared across all Portals |

Two axes, judged independently, evolve independently.

```
catering OrderService    retail OrderService    other Portal OrderService
        ↓                       ↓                          ↓
              ┌─────── OrderRepository ───────┐
              │   saveWithRelations()         │
              │   syncProducts() / syncPayments() ...
              └─────────────────┬─────────────┘
                                ↓
                              Models
```

Business divergence sits in the upper Service layer, data orchestration sits in the lower Repository layer — **business evolution doesn't affect CRUD, CRUD changes don't affect business**.

## 2. Service: business orchestration layer

Service has two placement locations, handling different concerns:

- **Module Service** (`app\Portals\{Portal}\Modules\{Module}\{Entity}\`): **internal business orchestration**, organized by Module and placed inside the Portal. Knows which Portal it belongs to and handles Portal-specific business rules (catering delivery fee, retail change). Multi-table CRUD is not its job — it delegates to the Repository. **This section primarily discusses this.**
- **Global Service** (`app\Services\` flat): cross-Portal business interfaces wrapping external integrations, **Portal-agnostic**. Examples: `InvoiceService` wrapping multiple invoice vendors, `NotificationService` wrapping multiple notification channels — callers receive "issue an invoice" / "send a notification" as business actions, without knowing which vendor is behind it. Both Module Services and Controllers can inject Global Services.

> **Service is the business interface** (issue invoice, send notification); **Integration is the vendor implementation** (EcpayClient, Every8dClient). Switching vendors only touches Integration; the Service interface stays stable. Global Services are flat-structured, opened on demand, and don't warrant much additional rule.

The rest of this section focuses on **Module Service** rules.

### Granularity rule (as a principle)

Within one Portal, the same business entity **maps to one Service, as a principle**:

- Path: `app\Portals\{Portal}\Modules\{Module}\{Entity}\{Entity}Service.php`
- The main Controller and **sub-resource Controllers of the same entity** share the same Service

Example:

```
app\Portals\PosCatering\Modules\Sale\Order\
├── OrderController.php           # main Controller
├── OrderPaymentController.php    # sub-resource Controller, injects OrderService
├── OrderShippingController.php   # sub-resource Controller, injects OrderService
└── OrderService.php              # Portal + Module + entity business orchestration
```

"As a principle" implies exceptions exist — **not "always"**. This wording deliberately leaves room for the convention to remain useful long-term.

### How to decide "same entity"

A common question is "does OrderPaymentController count as the same entity?" Decide via these markers:

| Marker | Match → same entity |
|---|---|
| URL sub-resource structure | `/orders/{order}/payments` → payment is a child of order |
| DB belongsTo relation | `OrderPayment` schema `belongsTo Order` → payment is a child of order |
| Business invariant coupling | Changing payment affects order status (fully_paid → status='paid') |

Core marker: **boundary of business actions matches** → same entity; **boundary differs** → separate Services.

### Typical Service flow

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
        // 1. Portal-specific business rule (catering only)
        $this->validateDeliveryAddress($data['shipping']);
        $data['delivery_fee'] = $this->calcDeliveryFee($data['shipping'], $data['items']);

        // 2. Multi-table CRUD → delegate to Repository (shared across Portals)
        $order = $this->orderRepo->saveWithRelations(new Order(), $data, $user);

        // 3. Portal-specific business follow-up (catering only)
        $this->dispatchToDeliveryQueue($order);
        $this->sms->send($order->customer->mobile, "Order {$order->code} confirmed");

        return $order;
    }
}
```

The retail version is completely different:

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

Two Services with entirely different business — but both rely on **the same** `OrderRepository::saveWithRelations()` to write data into multiple tables. **This is the biggest payoff of the two-axis split**: natural business isolation + zero data-layer duplication.

### When to create a Service

Not every Controller needs a Service. **Create one only when any of these apply**:

1. Multi-Model orchestration (create order + sync products / options / payments)
2. Portal-specific business divergence
3. Multiple Controllers for the same entity share orchestration
4. Reuse outside HTTP context (Queue Jobs, scheduled tasks)
5. A single Controller method exceeds ~100 lines of orchestration

**A single Portal with simple CRUD doesn't need a Service**. But **single-Portal projects are also allowed to open Services** — the convention says "as a principle", not "always". If one operation has to sync multiple tables, open a Service to avoid a fat controller — more pragmatic than rigidly forcing "cross-Portal only".

## 3. Repository: entity multi-table CRUD orchestration

**This layer is opened on demand — most situations don't need a Repository**. Simple CRUD calls Eloquent directly (`$model->fill()->save()`); single-table writes shouldn't be wrapped in a Repository either. **Open it only when "an entity spans multiple tables and needs transaction orchestration"** — for example, creating an order while syncing products / options / payments / shipping_infos. Pulling out a Repository "for layering tidiness" is over-engineering and lands you back in the query-wrapper anti-pattern.

A Repository is "**entity multi-table CRUD orchestration layer**" — it manages the entity's transaction across multiple tables, **doesn't know which Portal calls it**, and doesn't handle Portal-specific business rules.

This differs from the typical Repository pattern (`$repo->find()` wrapping `Model::find()`) — **we only use Repositories for multi-table orchestration**, **not for query wrapping**.

### Location: flat in `app\Repositories\`

No Portal grouping, no Module grouping (because Repositories are inherently shared across Portals):

```
app\Repositories\
├── OrderRepository.php
├── ProductRepository.php
├── PurchaseRepository.php       # purchasing (shared across brands)
└── MaterialRepository.php       # materials (shared)
```

### Typical shape

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

### Repository ↔ Service dependency direction

| Rule | Why |
|---|---|
| Service → Repository | Service orchestrates business; Repository is its CRUD tool |
| **Repository cannot call Service** | Repository must remain Portal-agnostic |
| Repository → other Repositories | Cross-entity multi-table transactions (e.g. Order also writes Inventory) |
| Service → Model (skipping Repository) | Simple single-table operations don't need a Repository |

**One-way dependency** is the core constraint of this layering — once Repository knows about a Portal, cross-Portal sharing breaks.

### Multi-brand shared example

Imagine the same system serves two independent brands (BrandA and BrandB), sharing purchasing, accounting, and material master data, but with independent order workflows and marketing per brand. The layering:

```
app\Portals\
├── PosCateringBrandA\Modules\Sale\Order\OrderService.php   # Brand A order orchestration
├── PosCateringBrandB\Modules\Sale\Order\OrderService.php   # Brand B order orchestration
├── OcadminBrandA\Modules\Purchase\PurchaseController.php   # Brand A purchasing
└── OcadminBrandB\Modules\Purchase\PurchaseController.php   # Brand B purchasing

app\Repositories\
├── OrderRepository.php       # Multi-table CRUD for both brands' orders
├── PurchaseRepository.php    # Purchasing (shared)
├── MaterialRepository.php    # Materials (shared)
└── AccountingRepository.php  # Accounting (shared)
```

**Business divergence sits in Module Services**: the two brands can have completely different order workflows (different SOPs, different discount rules, different notification templates).
**Data orchestration sits in Repositories**: the two brands' order-writing logic is identical, sharing one Repository.

Two axes, independent — BrandA's flow change doesn't affect BrandB; shared purchasing CRUD changes once, both brands evolve together. This is the most concrete example of "Repository shared across Portals".

## 4. Why we reject query-wrapper Repository (but accept entity-orchestration Repository)

The word "Repository" has a bad reputation in the Laravel community — we used to explicitly reject it too. The key to clarify: data operations have three modes, and our Repository **only handles one of them**.

### The three modes of reading / writing data

| Mode | Example | Goes through | Through Repository? |
|---|---|---|---|
| **(1) Single-record / single-table operation** | `Order::find($id)`, `$order->save()` | Eloquent directly | No |
| **(2) Domain query (incl. complex filters)** | `Order::filterByPhone($phone)->paginate()` | Model Scope (domain knowledge in the Model) | No |
| **(3) Multi-table transaction orchestration** | `$orderRepo->saveWithRelations(...)` | Repository | **Yes** (entity-orchestration) |

**Repository only handles mode (3)**. Modes (1) and (2) go through Eloquent / Scope directly — callers use them directly, no Repository indirection.

### Query-wrapper Repository is the anti-pattern (rejected)

A "Query-wrapper Repository" wraps modes (1) and (2) into Repository methods like `$repo->find()` / `$repo->findActive()` — that's the anti-pattern we reject:

| Anti-pattern (rejected) | What we do instead |
|---|---|
| `$orderRepo->find($id)` wrapping `Order::find($id)` | Just call `Order::find($id)` |
| `$orderRepo->findActive()` wrapping `Order::where('is_active', true)->get()` | Model Scope: `Order::active()->get()` |
| `$orderRepo->findByPhone($phone)` wrapping a Scope | Model Scope: `Order::filterByPhone($phone)->...` |

**Reason**: Laravel Eloquent is already Active Record + Query Builder. `Order::find($id)` is already concise and readable; wrapping it as `$repo->find($id)` adds nothing — only one layer of indirection, three places to change for each new query (Controller / Service / Repository), and inconsistencies like "this Controller goes through Service, that one calls Repository directly".

### Entity-orchestration Repository (accepted)

Our Repository doesn't do pass-through `find()`. It does **mode (3)** — multi-table transaction orchestration, sitting on top of Eloquent, something the ORM doesn't solve for you:

```php
$orderRepo->saveWithRelations($order, $data, $user);
// → DB::transaction → fill + save + syncProducts + syncShipping + syncPayments
```

This is "Domain Service in Repository's name" — the naming isn't strictly Fowler-original (Fowler's Repository is a collection-like query abstraction). But **the team is used to this usage**, so we keep the name and define the responsibility boundary clearly in documentation.

### Our final layering

```
✓ Controller → Model                              (simple mode (1))
✓ Controller → Model::Scope                       (domain query mode (2))
✓ Controller → Service → Repository → Model       (business + multi-table mode (3))
✓ Service → Integration Client                    (external API)
✓ Service → Supports / Helpers                    (pure-computation helpers)
```

**Repository doesn't do query wrapping — only entity-persistence orchestration**. We redefine the responsibility boundary of the name: keep what we need (multi-table transaction orchestration), reject what's anti-pattern (query wrapping).

## 5. Supports vs Helpers: pure-computation utilities

`app\Services\` is reserved for Portal business orchestration. But some things are **Portal-agnostic, don't touch the DB, and are pure computation** — like `ShippingFeeCalculator` for a delivery fee, `OrderLabelFormatter` for label data from an OrderProduct, `BarcodeBuilder` from an SKU.

Calling these `Service` would confuse with Module Services. We use two directories:

| | `app\Helpers\` | `app\Supports\` |
|---|---|---|
| Content | Cross-cutting technical tools, **no domain knowledge** | Domain pure computation, **knows entity / business rule** |
| What it references | Only raw data types (string, date, array) | Knows Model, enum, domain concepts |
| Relation to business | Business-agnostic, portable across projects | Tightly tied to business |
| Typical members | DateHelper, StringHelper, UrlHelper | OrderLabelFormatter, TaxCalculator |
| Internal layout | Flat or by raw type | By business domain subfolder |

**Judging rule**: ask "does this reference a Model or a domain enum?"

- No → `app\Helpers\` (cross-cutting technical tool)
- Yes → `app\Supports\{Domain}\` (domain pure computation)

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

Filename suffixes `XxxFormatter` / `XxxBuilder` / `XxxCalculator` — readers know the role at a glance, no confusion with Service.

## 6. Project-scale applicability

The new layering covers many project types:

| Project type | How it fits |
|---|---|
| **Single-Portal simple project** | Most Controllers call Model directly; open a Service when a single Controller operates on multiple tables |
| **Multi-Portal same business** (Ocadmin + POS doing the same) | Repository shared + Service per Portal (or even no Service if business is uniform) |
| **Multi-brand partial sharing** (two brands sharing purchasing / accounting / materials) | Purchasing / accounting / materials in Repository; orders / marketing in each brand's Module Service |
| **Multi-Portal divergent business** (PosCatering vs PosRetail) | Service isolates business, Repository shares entity persistence |

**Our earlier approach compressed all "share-or-not" decisions onto the Service axis** — broke down when business divergence was real. **The new layering splits two axes**, each evolves independently.

## 7. Quick reference: where things go

| You want to | Put it in |
|---|---|
| Simple list / single read | Controller + Model Scope |
| Multi-table transaction writes | Repository |
| Portal-specific business flow | Service |
| Same-entity multi-Controller orchestration | Service (shared) |
| Pure computation (knows entity) | `app\Supports\{Domain}\` |
| Pure computation (doesn't know entity) | `app\Helpers\` |
| External API calls (vendor clients) | `app\Integrations\` |
| Cross-Portal external-integration business interfaces (unified invoice / notification API) | `app\Services\` flat |
| Form defaults (plain fields) | `Model::$attributes` or `Model::defaults()` |
| Form defaults (needs DB query) | `Repository::newModel()` |
| Cross-brand shared purchasing / accounting / materials | Repository (flat) |

## 8. Design trade-offs worth pointing out

A few choices that often get questioned:

**"One Service per entity, as a principle" — "as a principle", not "always"** — the wording deliberately leaves an escape hatch. Read-only Controllers, Services growing too large to split, sub-resources evolving into independent entities — all are accepted exceptions. **"Always" forces hacks to bypass rules; "as a principle" lets conventions stay useful long-term**.

**Repository cannot call Service** — this constraint matters. Once violated, Repository starts knowing about Portals, and cross-Portal sharing breaks immediately. The constraint of "no inverted dependency" is worth far more than the apparent benefit (one less import line).

**Service sits next to Module structure in the Portal** (same entity folder, parallel to Controller) — different from what we used to do (Services flat in `app\Services\`). Same-folder upside: see OrderController, find OrderService next door, no folder-hopping. Trade-off: no accidental cross-Portal sharing of Services — which is exactly what we want.

## Closing

OCAdmin borrows OpenCart's UI design but **doesn't inherit its program-structure limitations**. OpenCart was constrained by raw SQL with no ORM — front-office and back-office could only duplicate the Model. Laravel has Eloquent, Scope, and DI; these tools let "business divergence (Service)" and "data orchestration (Repository)" evolve as two independent axes.

Thinking in **two axes** matches the real shape of design problems better than a one-axis linear judgment. The Service axis is about business divergence (whether different Portals have fundamentally different business). The Repository axis is about data orchestration (whether multi-table CRUD is complex enough to centralize). **Two axes, independent** — the same entity having different business across Portals while sharing multi-table CRUD — is the actual shape of most real-world projects.

Full convention document: internal `docs/common/10016_架構分層與Model職責.md`. This post only picks the design-rationale highlights; details (naming rules, "Service shouldn't do" lists, full Repository trigger conditions) follow the convention document.
