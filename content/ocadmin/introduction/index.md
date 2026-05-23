---
title: "OCAdmin 介紹：為什麼選 OpenCart 後台作為設計藍本"
date: 2026-05-24T14:00:00+08:00
draft: false
tags: ["OCAdmin", "OpenCart", "Laravel", "後台設計"]
categories: ["OCAdmin"]
summary: "OCAdmin 是我用 Laravel 重寫的管理系統，但 UI 結構直接借用 OpenCart 後台。為什麼不從零自己刻、不用 WordPress、不用 AdminLTE？這篇講選擇的理由。"
---

> [English version →](/ocadmin/en/introduction/)

這是 **OCAdmin 系列**的第一篇。先解釋「OCAdmin 是什麼」「為什麼長這樣」，後續再展開各個模組與設計決策。

## OCAdmin 是什麼

簡單講：**Laravel 寫的後台系統，但 UI 結構照搬 OpenCart 後台**。

不是 fork OpenCart，也不是 OpenCart 的 plugin。它是從零用 Laravel 蓋起來的專案，只是把 OpenCart 後台那套「列表頁長什麼樣、編輯頁有哪些 tab、按鈕放哪裡」當作 UI 設計藍本，避免每個專案都從零思考這些 UI 細節。

## 為什麼是 OpenCart，不是別的

實務上選擇大概三種：

### WordPress？

也是免費、PHP、社群大。但它的程式架構是「**為部落格演化出的後台**」——hook 系統、template hierarchy、post 概念到處滲透。要拿來改成商業/企業系統，等於得對抗很多歷史包袱，二次開發成本不低。

### AdminLTE / CoreUI / Tabler 等純 admin template？

提供漂亮的 UI 元件（按鈕、表格、表單欄位、卡片），但**它們只給你材料，沒有給你「典型頁面長什麼樣」的範本**。

換句話說：你還是要自己決定列表頁要不要有搜尋、篩選放側邊還是頂部、編輯頁怎麼分區、多語欄位放哪、批量操作 UX 是什麼。每個專案重新思考一次，沒效率。

### OpenCart：開源、頁面元素完整、結構易懂

OpenCart 是個能實際運作的電商系統，**後台已經把上面那些問題解決過幾百次**。商品、分類、訂單、客戶、各種設定頁……各種類型的資料管理頁面都已經設計好。

直接借用這些「已經被驗證可用」的 pattern，比自己重新發明快很多。對熟悉購物網站後台的人來說，OpenCart 後台一打開就知道每個按鈕大概要幹嘛——這就是「結構易懂」的價值。

## OpenCart 後台只有兩種頁面

整個後台拆解下來，**幾乎所有功能頁都是「列表頁」和「編輯頁」的變體**。

### 列表頁

```
┌──────────────────────────────────────────────┐
│ 標題                       [新增] [批量刪除]    │ ← 右上角按鈕
├──────────────────────────────────────────────┤
│ ☐  欄位 1   欄位 2   欄位 3            操作    │
│ ☐  ...      ...      ...           [編輯]    │ ← 每列右邊
│ ☐  ...      ...      ...           [編輯]    │
└──────────────────────────────────────────────┘
       [分頁]                       [總筆數]
```

固定有：篩選、排序、分頁、勾選批量操作。

### 編輯頁（以「商品」最具代表性）

商品這個編輯頁基本上把後台會遇到的各種情境都示範過一遍：

- **語言 tab**：商品名稱、meta_title、meta_description（**多語欄位**）
- **基本資料**：SKU、Model、Categories…（**單語、結構化欄位**）
- **選項 tab**：商品規格（**一對多的複雜子資料**）
- **圖片 tab**：主圖 + 額外圖（**檔案上傳管理**）
- **點數 tab**：紅利點數設定（**條件式設定**）
- **特價 tab**：促銷價（**時間範圍 + 條件**）

**只要把這個頁面的 pattern 內化，幾乎任何業務資料的編輯頁都能照這個模子做出來，不用再自己組合**。要開發新模組時，先問「我比較像商品的哪幾個 tab？」就有答案。

## 程式結構：前後台分離 + MVCL

OpenCart 原生的目錄結構：

```
upload/
├── catalog/          ← 前台（消費者看的購物網站）
│   ├── model/
│   ├── controller/
│   ├── view/
│   └── language/
│
└── admin/            ← 後台（管理員操作）
    ├── model/
    ├── controller/
    ├── view/
    └── language/
```

兩邊各自獨立，各有完整的 **MVCL**——Model / View / Controller / **Language**。

> 比一般 MVC 多出來的 `L`（Language）是 OpenCart 的關鍵設計：**把「多語文字」當作一等公民放進架構裡**，不是丟到 view 裡用 `<?= $lang['xxx'] ?>` 散亂處理。這個決定對需要多國語系的系統來說，省下後續一大堆語言檔組織的問題。OCAdmin 把這個概念直接搬到 Laravel。

OCAdmin 借的是 `admin/` 這套規矩，不關心 `catalog/`。

## 多語系：MVCL 的 L 在 OCAdmin 怎麼用

舉個具體例子，看 OCAdmin 改良後的開發體驗。

Laravel 原生的翻譯呼叫長這樣：

```blade
{{ __('admin/system/acl/permission.column_display_name') }}
```

每次都要寫「**完整檔案路徑 + key**」。一個欄位在頁面上用 5 次，就要打 5 次同樣的前綴；哪天 lang 檔搬位置，所有引用要一起改；路徑長了，整行視覺上一團亂。

OCAdmin 借用 OpenCart 的思路改造：**Controller 宣告載入哪些 lang 檔，View 只管用**。

```php
// Controller：宣告要載入哪些 lang 檔
class PermissionController extends OcadminController
{
    protected function setLangFiles(): array
    {
        return ['system/acl/permission'];
    }
}
```

Controller 會**依序疊載**（後者覆蓋前者，可疊任意層數）：

```
/lang/{locale}/admin/default.php                     ← 共用按鈕、訊息（自動）
/lang/{locale}/admin/system/acl/permission.php       ← 本模組（setLangFiles 宣告）
```

```blade
{{-- View：直接用 key，沒有路徑 --}}
<label>{{ $lang->column_display_name }}</label>
<button>{{ $lang->button_save }}</button>
```

有需要也能在傳給 view 前手動補/覆寫個別 key：

```php
$this->lang->column_name = '自訂顯示';
$data['lang'] = $this->lang;
return view('ocadmin::system.acl.permission.form', $data);
```

**視圖只關心要顯示什麼字、不關心字從哪裡來**——這是 MVCL 的 L 在實務上帶來的乾淨。細節（fallback 機制、key 命名慣例、tab_basic / tab_trans 標準化等）留給後續的「多語系設計」專文。

## OCAdmin 在 OpenCart 之上做了什麼

| 從 OpenCart 借 | 重新用 Laravel 實作 |
|---|---|
| 後台 UI 慣例（列表 + 編輯兩大頁面、tab 結構） | 路由、ORM、Middleware、DI 容器 |
| 多語系作為一等公民的概念 | Eloquent + 翻譯子表 |
| 前後台分離的目錄哲學 | Service Provider 註冊機制 |
| MVCL 的 L 角色 | Laravel 原生 `lang/` 目錄 |

換句話說：**外觀像 OpenCart 後台，骨子裡是 Laravel**。OpenCart 的舊式 MVC 引擎、template 系統、ORM 都不繼承——那些隨著 PHP 生態演進都有更好的替代品了。只繼承「網頁看起來該長什麼樣」這層 UI/UX 決策。

## 接下來的文章

這個系列接下來會陸續展開：

- **整體架構**：Core vs Module 怎麼切，為什麼這樣切
- **為什麼放棄麵包屑（breadcrumb）**：一個刻意「拿掉功能」的決策記錄
- **多語系設計**：lang 檔的組織、預設群組、fallback 機制
- **列表頁規範**：篩選 / 排序 / 分頁 / 網址保留怎麼一致設計
- **表單 AJAX 提交**：為什麼不用 redirect、錯誤怎麼回傳

各篇盡量獨立可讀，挑感興趣的看就行。
