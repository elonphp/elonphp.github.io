---
title: "DevLogin：給 CLI / Postman / AI Agent 的登入短路通道"
date: 2026-05-29T10:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "登入", "Testing", "AI Agent"]
categories: ["OCAdmin"]
weight: 10
summary: "OCAdmin 在 dev 環境（APP_ENV=local）提供一個 POST /dev/login 端點：給 email + token 就直接 Auth::login()，跳過密碼、CSRF、OAuth flow。動機是 CLI / Postman / AI Coding Agent 場景無法走正常登入路徑。這篇講為什麼用 Auth::login 而不是 actingAs（保留執行路徑一致）、4 道安全閘怎麼擋公網誤觸發、為什麼失敗一律 404 不洩漏 endpoint 存在、跟帳號模擬的本質差異。"
---

> [English version →](/ocadmin/en/dev-login/)

在 dev 環境寫測試 / 跑自動化 / 用 AI Agent 驗 controller 時，**登入這件事很麻煩**：

- `php artisan tinker` 想模擬登入態——沒 web request、沒 session
- curl / Postman / Insomnia 測 controller——要處理 CSRF token + cookie jar + 密碼
- 端對端測試——`actingAs()` 跳過 session 不夠真實、跑真登入太重
- AI Coding Agent（如 Claude Code、Cursor）想驗證頁面渲染——Agent 無法操作互動式登入
- 多角色 ACL 反覆切帳號驗證——登出 / 登入摩擦大

OCAdmin 在 `APP_ENV=local` 提供一個短路端點解這個摩擦：

```
POST /dev/login
Body: email=admin@example.com & token=<DEV_LOGIN_TOKEN>
→ Auth::login(user) + JSON 回應
```

無需密碼、無需 CSRF、無需 OAuth flow。4 道安全閘擋住公網誤觸發 + 不洩漏 endpoint 存在。

## 1. 設計核心：用 `Auth::login()`、不繞 ACL

DevLogin 是「**用密碼 / OAuth 之外的方式完成登入**」。登入完成後，target user **就是真正的當前登入者**——`Auth::user()` / `auth()->id()` / `request()->user()` 全部都是該 user，所有 controller / middleware / Gate 看到的都是該 user。

**沒有「模擬」層、沒有「以他身分代理操作」的概念**。

跟正規登入唯一的差別在「**怎麼進來**」：

| 登入方式 | 入口 | 後續行為 |
|---|---|---|
| 正規登入（密碼 / OAuth） | 帳密表單 / OAuth provider redirect | `Auth::login()` |
| **DevLogin** | `POST /dev/login` + token + email | `Auth::login()`（**完全相同**） |

兩種入口在 session 裡留下完全等價的登入態。後續 controller 看到的 request、`Auth::user()`、Gate::check 結果都不可區分。

### 為什麼不用 `actingAs()` 也不自己塞 session

Laravel 測試常用的 `$this->actingAs($user)` 跟「自己塞 session 假裝登入」這兩條路都做過——都有問題：

- **`actingAs()` 跳過 session**：只能在 PHPUnit 內用，無法給外部工具（curl / Postman / AI Agent）；而且跳過 session middleware 的副作用（如 session-based locale、CSRF token 生成）會跟正式環境不一致
- **自己塞 session**：要手寫 session driver 的格式、容易跟 Laravel 內部 contract drift

走 `Auth::login()` 的好處：**dev 環境跟 production 環境的執行路徑完全一致**。避免「dev 過了 prod 掛了」的測試環境 false-positive。

### 登入後權限照常

DevLogin 純粹是「**登入入口**」的繞過，**登入後一切權限檢查照常**。target user 是 super_admin 就有 super_admin 權限；是受限角色就只看得到受限選單。本機制不繞過 ACL。

呼應 [權限機制](/ocadmin/permissions/) 的 Gate 設計——`super_admin` 走 `Gate::before` 放行所有檢查，其他角色按 Spatie 權限走。DevLogin 只決定「誰登入了」、不決定「登入後能做什麼」。

## 2. 端點 spec

```
POST /dev/login
Body (form):
  email=<target user email>
  token=<DEV_LOGIN_TOKEN value>

Response:
  200 OK { "success": true, "user": { id, email, ... } }   通過
  404                                                       任一安全閘不過
  403  "Invalid token"                                      閘 1-3 過、token 不對
  422  "email is required"                                  缺 email
  404  "User not found: ..."                                user 不存在
```

選 `POST` 不選 `GET`：token 進 body 不寫 access log。query string 的 token 容易被 referer / proxy log 外洩。

## 3. 4 道安全閘

每個 request 進來時依序檢查 4 道閘，任一失敗 → `abort(404)`：

| # | 閘 | 來源 | 失敗碼 |
|---|---|---|---|
| 1 | `APP_ENV=local` | `app()->environment()` | 404 |
| 2 | `DEV_LOGIN_TOKEN` 非空 | `config('auth.dev_login.token')` | 404 |
| 3 | 來源 IP ∈ allowlist | `config('auth.dev_login.allowed_ips')` | 404 |
| 4 | POST token `hash_equals` 比對通過 | request body | **403** |

**IP allowlist 預設**（loopback + RFC1918 私有網段 + IPv6 ULA）：

```
127.0.0.0/8      # IPv4 loopback
10.0.0.0/8       # RFC1918 A
172.16.0.0/12    # RFC1918 B
192.168.0.0/16   # RFC1918 C
::1              # IPv6 loopback
fc00::/7         # IPv6 ULA
```

不在 allowlist 表示請求來自公網——即使 `APP_ENV=local` 誤上線、token 還在，也擋。

## 4. 為什麼失敗一律 404（除了閘 4）

**前 3 道閘不過 → 404**。這是刻意設計：**不洩漏 endpoint 是否啟用**。攻擊者掃 `/dev/login` 跟掃任何不存在的 path（`/foo-bar-baz`）體驗完全相同——看不到「這個 endpoint 存在但你沒權限」這種訊息。

**閘 4 不過 → 403 "Invalid token"**。這是內網開發者誤觸發的場景：前 3 閘都過代表「我在 dev 環境、我知道 endpoint 存在、我在 allowlist IP」，只是 `.env` token typo——回 403 讓開發者知道是 token 問題、不用懷疑路由是不是註冊了。

### 不洩漏「endpoint 啟用」的意義

考慮兩個攻擊情境：

1. **`APP_ENV=staging` 誤填 `local`**：閘 1 過、閘 2 過（token 設了）、閘 3 看公網 IP 不通 → 404。攻擊者看不出 endpoint 存在
2. **正式環境 `.env` 誤 commit token**：閘 1 看到 production → 404。即使 token 流出也沒入口

如果失敗給 401/403，攻擊者掃到 `/dev/login` 就知道「這站有 dev login endpoint、繼續試 token」。一律 404 後攻擊者得到的 signal 跟掃 `/random-nonexistent-path` 一樣——**沒有可以繼續攻擊的線索**。

## 5. 設定

### .env

```bash
# DevLogin 短路登入 token（僅 APP_ENV=local 有效）
# 留空 = 機制關閉（路由不註冊）
# 產生：openssl rand -hex 32  或  php -r "echo bin2hex(random_bytes(32));"
DEV_LOGIN_TOKEN=
```

**留空 = 關閉**——預設不啟用、必須手動加 token 才會 active。

### config/auth.php

```php
'dev_login' => [
    // .env 沒設 → 等同 endpoint 關閉
    'token' => env('DEV_LOGIN_TOKEN'),

    // 允許 IP（loopback + RFC1918 私有網段 + IPv6 ULA）
    'allowed_ips' => [
        '127.0.0.0/8',
        '10.0.0.0/8',
        '172.16.0.0/12',
        '192.168.0.0/16',
        '::1',
        'fc00::/7',
    ],
],
```

### 一行生成 token

```bash
# Linux / macOS
echo "DEV_LOGIN_TOKEN=$(openssl rand -hex 32)" >> .env

# PowerShell
$NEW = -join ((1..64) | %{ '{0:x}' -f (Get-Random -Max 16) })
Add-Content .env "DEV_LOGIN_TOKEN=$NEW"
```

### 路由是條件式註冊

```php
// routes/web.php
if (app()->environment('local') && config('auth.dev_login.token')) {
    Route::post('/dev/login', [DevLoginController::class, 'login'])
        ->withoutMiddleware([\Illuminate\Foundation\Http\Middleware\VerifyCsrfToken::class]);
}
```

正式環境 / 沒設 token，路由根本沒註冊——攻擊面從一開始就不存在。

## 6. 實際用法

### curl 兩步驟

```bash
TOKEN='your-dev-token'

# 1. 登入（cookies 寫進 jar）
curl -c /tmp/devcookies.txt \
  -d "email=admin@example.com&token=$TOKEN" \
  http://127.0.0.1:8000/dev/login

# 2. 帶 cookie 訪問頁面
curl -b /tmp/devcookies.txt -L \
  "http://127.0.0.1:8000/zh-hant/admin/catalog/products?filter_name=apple" \
  -o /tmp/page.html
```

### 切換角色

```bash
rm -f /tmp/devcookies.txt
curl -c /tmp/devcookies.txt \
  -d "email=editor@example.com&token=$TOKEN" \
  http://127.0.0.1:8000/dev/login
```

### Postman / Insomnia

跟 curl 一樣，但 **cookie jar 由工具自動管理**：

1. 開新 collection、設 environment variable `DEV_LOGIN_TOKEN`
2. 加 POST `/dev/login` request，body 用 form-data：`email=admin@example.com` + `token={{DEV_LOGIN_TOKEN}}`
3. 後續所有 request 自動帶 session cookie——直接打 `/zh-hant/admin/...` 路徑就是 logged-in 狀態

### AI Coding Agent 場景

Claude Code / Cursor 等 AI Agent 改完 controller 想驗證頁面正確時，可以呼叫 DevLogin 拿 session 後 curl 頁面比對 HTML：

```bash
# Agent 自己跑：
TOKEN=$(grep '^DEV_LOGIN_TOKEN=' .env | cut -d= -f2)
curl -s -c /tmp/c.txt -d "email=admin@example.com&token=$TOKEN" \
  http://127.0.0.1:8000/dev/login
curl -s -b /tmp/c.txt "http://127.0.0.1:8000/zh-hant/admin/foo" | grep "預期字串"
```

**這是寫這個機制最初的動機**——AI Agent 修完代碼想驗證實際行為時，從前要不就 actingAs 跑單元測試（不夠真實）、要不就請人類手動開瀏覽器確認（破壞 Agent 自動化）。DevLogin 提供 Agent 一條「自助驗證頁面」的路。

### 在 PHP process 內？不需要 DevLogin

tinker / artisan command 已經在 process 內、不走 HTTP，直接：

```php
Auth::login(User::where('email', 'admin@example.com')->first());
```

DevLogin 是給 **HTTP 層** 短路用的。process 內已能直接 `Auth::login()`、不需要這個機制。

## 7. 安全護欄總表

| 護欄 | 作用 |
|---|---|
| `APP_ENV=local` 環境鎖 | staging / production 即使誤填 `.env` 也是 404 |
| `.env` 不可 commit | `.gitignore` 預設規則；token 等同 RCE 級權限 |
| IP allowlist | 公網請求即使知道 token 也是 404 |
| `hash_equals` 比對 | 防 timing attack |
| `Log::warning` 留紀錄 | 每次使用都記 user_id / email / ip / roles 進 log |
| 4 道閘獨立判斷 | 任一不過即 404，不洩漏哪一閘擋掉 |

### Token 等同 RCE 級權限

**token 外洩 = 任何 user 都能被登入**。如果你的測試 super_admin 是 admin@example.com、token 知道了——對方就是 super_admin。所以：

- `.env` 絕對不能 commit（`.gitignore` 預設已有，但要 review）
- token 不能截圖 / 不能貼到 chat / 不能寫在 README
- 遠端 dev 機器嚴控 SSH 存取
- 懷疑外洩立即輪換：

```bash
# 生成新 token 覆寫
NEW=$(openssl rand -hex 32)
sed -i "s/^DEV_LOGIN_TOKEN=.*/DEV_LOGIN_TOKEN=$NEW/" .env
```

舊 token 立即失效（`hash_equals` 比對不通）；dev 環境每 request 都重讀 `.env`，**無需重啟 server**。

## 8. 風險與緩解

| 風險 | 緩解 |
|---|---|
| token 外洩等同 super_admin 登入權 | `.env` 不 commit；懷疑外洩立即輪換；遠端 dev 機器嚴控 SSH |
| 公網來源 IP 偽造（反向代理 / ngrok 暴露） | `$request->ip()` 取的是 trusted proxy 處理後 IP；用 ngrok 時請暫時拿掉 DEV_LOGIN_TOKEN |
| 多人共用 dev DB | 同事 email 即足以登入該人身分。log 留紀錄但**不可逆**；多人共用環境請各人各自設 token，或乾脆關掉 |
| CSRF middleware 排除被遺忘 | `bootstrap/app.php` 集中設定；CSRF middleware 升級時要 review except 仍有效 |
| log 雜訊 | 每次 dev login 都 warning level；看 log 時 filter `dev login` |

## 9. 不是「帳號模擬」、別混為一談

OCAdmin 另有「帳號模擬機制」（impersonate）——super_admin 為了 debug、以另一個 user 身分**「看」**系統。兩者很容易混淆，但本質完全不同：

| 項目 | DevLogin | 帳號模擬（impersonate）|
|---|---|---|
| 入口 | dev 環境 HTTP 短路 endpoint | production 後台 UI 按鈕 |
| 環境 | 只 dev | dev + production 都可 |
| 概念 | 「登入」——target user 就是當前 user | 「代理」——有「super_admin 以該 user 身分看」這層 |
| 責任歸屬 | 該 target user（無代理層）| super_admin 本人（log 標記 impersonation）|
| 守衛 | 不需（dev 環境 + 4 道閘）| 唯讀守衛、雙向鎖、退出機制 |

**DevLogin 沒有「代理層」**——登入後就是該 user、責任也是該 user 的（雖然 dev 環境通常不追究）。impersonate 有完整代理機制、留下「super_admin X 以 user Y 身分操作」的 log。

兩個機制各做各的、互不引用。如果你想在 production 做安全的「以該 user 身分看」、走 impersonate；如果你想在 dev 一鍵切角色測試、走 DevLogin。

## 結語

DevLogin 解決的是「**dev 環境的真實登入摩擦**」——不是「測試環境的偽登入工具」、也不是「production 的帳號模擬」。

它的設計核心是：**讓 dev 環境的執行路徑跟 production 完全一致**（用 `Auth::login()`、不跳 session）；同時用 4 道閘 + 404 永遠不洩漏 endpoint 存在來保證「設錯也安全」。

對 AI Agent 與自動化測試的時代特別重要——以前手動開瀏覽器登入這個摩擦點，現在 Agent 可以自己處理。當 Agent 改完一個 controller、curl 拿頁面驗證行為時、開發者只需要看最後結果。整個 review-fix-verify 的循環從「人類在中間每步參與」變成「Agent 自己跑、人類只看最後結論」。
