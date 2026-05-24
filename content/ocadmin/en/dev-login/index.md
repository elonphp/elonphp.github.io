---
title: "DevLogin: A Login Shortcut for CLI / Postman / AI Agents"
date: 2026-05-29T10:00:00+08:00
draft: false
tags: ["OCAdmin", "Laravel", "Authentication", "Testing", "AI Agent"]
categories: ["OCAdmin"]
weight: 10
summary: "OCAdmin exposes a POST /dev/login endpoint in the dev environment (APP_ENV=local): give it email + token and it calls Auth::login() directly, skipping password, CSRF, and the OAuth flow. The motivation: CLI / Postman / AI Coding Agent scenarios can't easily run the normal login path. This post covers why it uses Auth::login() instead of actingAs (to keep the execution path identical), the four security gates that block public-facing accidents, why failures always return 404 (not exposing endpoint existence), and how this differs in essence from account impersonation."
build:
  list: local
---

> [→ 繁體中文版](/ocadmin/dev-login/)

When writing tests / running automation / using AI Agents to verify controllers in the dev environment, **logging in is annoying**:

- `php artisan tinker` wants a simulated login state — no web request, no session
- curl / Postman / Insomnia hitting a controller — has to deal with CSRF tokens, cookie jars, passwords
- End-to-end tests — `actingAs()` skips session and isn't realistic enough; running a full login is too heavy
- AI Coding Agents (Claude Code, Cursor, etc.) verifying page renders — Agents can't operate interactive login forms
- Verifying multi-role ACL by repeatedly switching accounts — the logout / login friction adds up

OCAdmin exposes a shortcut endpoint when `APP_ENV=local`:

```
POST /dev/login
Body: email=admin@example.com & token=<DEV_LOGIN_TOKEN>
→ Auth::login(user) + JSON response
```

No password, no CSRF, no OAuth flow. Four security gates block public-facing accidents and never expose endpoint existence.

## 1. Design core: use `Auth::login()`, never bypass ACL

DevLogin is "**logging in by means other than password / OAuth**". After login, the target user **is the actual current user** — `Auth::user()` / `auth()->id()` / `request()->user()` all return that user. Every controller / middleware / Gate sees that user.

**There's no "simulation" layer, no concept of "operating on their behalf"**.

The only difference vs regular login is **how you got in**:

| Login method | Entry | Subsequent behavior |
|---|---|---|
| Regular login (password / OAuth) | Credential form / OAuth provider redirect | `Auth::login()` |
| **DevLogin** | `POST /dev/login` + token + email | `Auth::login()` (**identical**) |

Both entries leave equivalent login state in the session. The downstream controller, `Auth::user()`, and `Gate::check` results are indistinguishable.

### Why not `actingAs()` or hand-rolled session injection

Laravel testing commonly uses `$this->actingAs($user)`, or people roll their own session injection — both have problems:

- **`actingAs()` skips session**: only works inside PHPUnit, not for external tools (curl / Postman / AI Agents). The side effects of skipping session middleware (session-based locale, CSRF token generation, etc.) diverge from production
- **Hand-rolled session injection**: you'd write the session driver's storage format yourself, easy to drift from Laravel's internal contract

Going through `Auth::login()` keeps **the execution path in dev identical to production**. Avoids the "passes in dev, breaks in prod" false-positive trap.

### Permissions still apply after login

DevLogin is purely a bypass of "**the login entry**". **All permission checks afterward apply normally**. If the target user is super_admin, they get super_admin permissions; if they're a limited role, they only see the limited menu. This mechanism does not bypass ACL.

Echoes the Gate design in [Permission Mechanism](/ocadmin/en/permissions/) — `super_admin` is granted by `Gate::before`, other roles follow Spatie permissions. DevLogin only decides "who logged in", not "what they can do after".

## 2. Endpoint spec

```
POST /dev/login
Body (form):
  email=<target user email>
  token=<DEV_LOGIN_TOKEN value>

Response:
  200 OK { "success": true, "user": { id, email, ... } }   passed
  404                                                       any security gate failed
  403  "Invalid token"                                      gates 1-3 passed, token wrong
  422  "email is required"                                  missing email
  404  "User not found: ..."                                user doesn't exist
```

`POST` chosen over `GET`: token in body doesn't show up in access logs. A token in the query string leaks via referer / proxy logs.

## 3. Four security gates

Every request checks four gates in order. Any failure → `abort(404)`:

| # | Gate | Source | Fail code |
|---|---|---|---|
| 1 | `APP_ENV=local` | `app()->environment()` | 404 |
| 2 | `DEV_LOGIN_TOKEN` is non-empty | `config('auth.dev_login.token')` | 404 |
| 3 | Source IP ∈ allowlist | `config('auth.dev_login.allowed_ips')` | 404 |
| 4 | POST token passes `hash_equals` | request body | **403** |

**IP allowlist default** (loopback + RFC1918 private ranges + IPv6 ULA):

```
127.0.0.0/8      # IPv4 loopback
10.0.0.0/8       # RFC1918 A
172.16.0.0/12    # RFC1918 B
192.168.0.0/16   # RFC1918 C
::1              # IPv6 loopback
fc00::/7         # IPv6 ULA
```

Not in the allowlist means the request came from the public internet — blocked even if `APP_ENV=local` is accidentally set in production with a token still configured.

## 4. Why failures always return 404 (except gate 4)

**Gates 1-3 fail → 404**. This is deliberate: **never expose whether the endpoint is enabled**. An attacker scanning `/dev/login` gets the exact same response as scanning any non-existent path (`/foo-bar-baz`) — no "this endpoint exists but you're not allowed" signal.

**Gate 4 fails → 403 "Invalid token"**. This is the dev-team-typo scenario: passing the first three gates means "I'm in dev, I know the endpoint exists, I'm on an allowlisted IP" — the only remaining cause of failure is a typo in `.env`. Returning 403 lets the developer know it's a token issue and not a routing problem.

### Why hiding endpoint existence matters

Consider two attack scenarios:

1. **`APP_ENV=staging` accidentally set to `local`**: gate 1 passes, gate 2 passes (token is set), gate 3 fails because the public IP isn't in allowlist → 404. The attacker can't tell the endpoint exists
2. **`.env` token accidentally committed to a public repo**: gate 1 sees production → 404. Even if the token leaks, there's no entry point

If failure returned 401/403, an attacker scanning `/dev/login` would see "this site has a dev login endpoint, keep trying tokens". A uniform 404 makes the signal identical to scanning `/random-nonexistent-path` — **no follow-up information to feed an attack**.

## 5. Configuration

### .env

```bash
# DevLogin shortcut token (only effective when APP_ENV=local)
# Empty = mechanism disabled (route not registered)
# Generate: openssl rand -hex 32  or  php -r "echo bin2hex(random_bytes(32));"
DEV_LOGIN_TOKEN=
```

**Empty = disabled** — off by default; you must explicitly set a token to enable.

### config/auth.php

```php
'dev_login' => [
    // .env not set → endpoint disabled
    'token' => env('DEV_LOGIN_TOKEN'),

    // Allowed IPs (loopback + RFC1918 + IPv6 ULA)
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

### Generate token in one line

```bash
# Linux / macOS
echo "DEV_LOGIN_TOKEN=$(openssl rand -hex 32)" >> .env

# PowerShell
$NEW = -join ((1..64) | %{ '{0:x}' -f (Get-Random -Max 16) })
Add-Content .env "DEV_LOGIN_TOKEN=$NEW"
```

### Route is conditionally registered

```php
// routes/web.php
if (app()->environment('local') && config('auth.dev_login.token')) {
    Route::post('/dev/login', [DevLoginController::class, 'login'])
        ->withoutMiddleware([\Illuminate\Foundation\Http\Middleware\VerifyCsrfToken::class]);
}
```

In production / without a token set, the route isn't even registered — **the attack surface doesn't exist in the first place**.

## 6. Practical usage

### curl in two steps

```bash
TOKEN='your-dev-token'

# 1. Log in (cookies saved to jar)
curl -c /tmp/devcookies.txt \
  -d "email=admin@example.com&token=$TOKEN" \
  http://127.0.0.1:8000/dev/login

# 2. Access pages with the cookie
curl -b /tmp/devcookies.txt -L \
  "http://127.0.0.1:8000/en/admin/catalog/products?filter_name=apple" \
  -o /tmp/page.html
```

### Switching roles

```bash
rm -f /tmp/devcookies.txt
curl -c /tmp/devcookies.txt \
  -d "email=editor@example.com&token=$TOKEN" \
  http://127.0.0.1:8000/dev/login
```

### Postman / Insomnia

Same as curl, but **the cookie jar is managed by the tool**:

1. New collection, set environment variable `DEV_LOGIN_TOKEN`
2. Add a POST `/dev/login` request with form-data body: `email=admin@example.com` + `token={{DEV_LOGIN_TOKEN}}`
3. All subsequent requests automatically carry the session cookie — hit `/en/admin/...` paths directly in a logged-in state

### AI Coding Agent scenario

When Claude Code / Cursor / similar AI Agents finish modifying a controller and want to verify the page is correct, they can call DevLogin to grab a session and curl the page to diff the HTML:

```bash
# Agent runs this itself:
TOKEN=$(grep '^DEV_LOGIN_TOKEN=' .env | cut -d= -f2)
curl -s -c /tmp/c.txt -d "email=admin@example.com&token=$TOKEN" \
  http://127.0.0.1:8000/dev/login
curl -s -b /tmp/c.txt "http://127.0.0.1:8000/en/admin/foo" | grep "expected string"
```

**This is the original motivation for the mechanism** — when an AI Agent finishes editing code and wants to verify actual behavior, the alternatives used to be: run unit tests with `actingAs()` (not realistic enough), or ask a human to open a browser and confirm (breaks Agent autonomy). DevLogin gives Agents a "self-service page verification" path.

### Inside a PHP process? No DevLogin needed

tinker / artisan commands are already inside the process and not going through HTTP — just call directly:

```php
Auth::login(User::where('email', 'admin@example.com')->first());
```

DevLogin is for **the HTTP layer**. In-process code can call `Auth::login()` directly and doesn't need this mechanism.

## 7. Security guardrails summary

| Guardrail | Effect |
|---|---|
| `APP_ENV=local` environment lock | staging / production stay 404 even with `.env` mistakes |
| `.env` must not be committed | `.gitignore` default rules; the token grants RCE-grade privileges |
| IP allowlist | public-internet requests stay 404 even with a leaked token |
| `hash_equals` comparison | timing-attack resistance |
| `Log::warning` per use | each invocation logs user_id / email / ip / roles |
| 4 gates judged independently | any failure → 404, no indication which gate failed |

### Token = RCE-grade privileges

**Token leak = anyone can be logged in**. If your test super_admin is admin@example.com and the token leaks — the attacker becomes super_admin. So:

- Never commit `.env` (`.gitignore` covers this by default, but review)
- Don't screenshot the token, don't paste it in chat, don't put it in README
- Tightly restrict SSH access to remote dev machines
- On suspicion of leak, rotate immediately:

```bash
# Generate new token, overwrite existing line
NEW=$(openssl rand -hex 32)
sed -i "s/^DEV_LOGIN_TOKEN=.*/DEV_LOGIN_TOKEN=$NEW/" .env
```

The old token becomes invalid instantly (`hash_equals` no longer matches); dev environments re-read `.env` per request, so **no server restart needed**.

## 8. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Token leak grants super_admin login privileges | Don't commit `.env`; rotate on suspected leak; restrict remote dev SSH |
| Public-source IP spoofing (reverse proxy / ngrok tunnels) | `$request->ip()` reads the trusted-proxy-processed IP; temporarily remove `DEV_LOGIN_TOKEN` when using ngrok |
| Shared dev DB across multiple developers | A coworker's email alone is enough to log in as them. Log entries are recorded but **not reversible**; shared environments should use per-developer tokens or just disable |
| CSRF middleware exception forgotten | `bootstrap/app.php` centralizes the setting; review the except list when upgrading CSRF middleware |
| Log noise | Every dev login is warning-level; filter logs by `dev login` when reading |

## 9. This is NOT "account impersonation" — don't confuse them

OCAdmin has a separate "account impersonation" mechanism — where super_admin **"views"** the system as another user for debugging purposes. The two are easy to confuse but fundamentally different:

| Item | DevLogin | Account Impersonation |
|---|---|---|
| Entry | dev environment HTTP shortcut endpoint | production backend UI button |
| Environment | dev only | dev + production |
| Concept | "log in" — target user becomes the actual current user | "act as" — there's a "super_admin viewing as that user" layer |
| Responsibility | The target user (no proxy layer) | super_admin themselves (impersonation flag in logs) |
| Guards | Not needed (dev env + 4 gates) | Read-only guard, two-way lock, exit mechanism |

**DevLogin has no "proxy layer"** — after login you ARE the user, and the responsibility is that user's (though dev environments usually don't audit). Impersonation has a full proxy mechanism and leaves "super_admin X operated as user Y" logs.

The two mechanisms operate independently and don't reference each other. If you want a safe "view-as-user" capability in production, use impersonation; if you want to one-click switch roles for dev testing, use DevLogin.

## Closing

DevLogin solves "**the real login friction in dev environments**" — it's not "a fake login tool for the test environment", and it's not "production account impersonation".

Its design core: **keep the execution path in dev identical to production** (use `Auth::login()`, don't skip session); combined with 4 gates + uniform 404 responses that never expose endpoint existence to guarantee "safe by default even when misconfigured".

This matters especially in the era of AI Agents and automation — the previously manual friction of opening a browser to log in can now be handled by the Agent itself. The review-fix-verify loop changes from "human involved at every step" to "Agent runs everything, human just checks the final outcome".
