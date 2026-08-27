# 🔒 Security Audit Report — Algero Platform

**Date:** May 7, 2026  
**Scope:** Full-stack (Laravel backend + React frontend)  
**Auditor:** AI Security Scan

---

## Summary

| Severity | Count |
|----------|-------|
| 🔴 **Critical** | 3 |
| 🟠 **High** | 4 |
| 🟡 **Medium** | 5 |
| 🔵 **Low** | 4 |
| ✅ **Passed** | 8 |

---

## 🔴 Critical Issues

### 1. Stored XSS via `dangerouslySetInnerHTML`

**Files:**
- [BlogViewPage.tsx](file:///c:/Users/office/Documents/shuvo/algero-frontend/frontend/src/app/blog/BlogViewPage.tsx#L286)
- [ProjectViewPage.tsx](file:///c:/Users/office/Documents/shuvo/algero-frontend/frontend/src/app/projects/ProjectViewPage.tsx#L237)

**Problem:** Blog `content` and project `content` are rendered as raw HTML from the database without any sanitization. A compromised admin account (or a regular admin injecting scripts before Super Admin approval) can inject `<script>` tags, event handlers (`onerror`, `onload`), or `<iframe>` elements that execute arbitrary JavaScript in every visitor's browser.

```tsx
// VULNERABLE — renders any HTML from database
dangerouslySetInnerHTML={{ __html: project.content }}
```

**Impact:** Full session hijacking, cookie theft, keylogging, phishing overlays, defacement.

**Fix:** Sanitize HTML server-side before storage **or** client-side before rendering:
```bash
npm install dompurify
```
```tsx
import DOMPurify from 'dompurify';
dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(project.content) }}
```
Or better — sanitize on the backend in `storeBlog` / `storeProject` using a library like `mews/purifier`.

---

### 2. Unrestricted SVG Upload → Stored XSS

**File:** [AdminController.php:579-593](file:///c:/Users/office/Documents/shuvo/algero-frontend/backend/app/Http/Controllers/AdminController.php#L579-L593)

**Problem:** The image upload allows `svg` files:
```php
'image' => 'required|image|mimes:jpeg,jpg,png,gif,webp,svg|max:5120',
```

SVG files can contain embedded JavaScript (`<script>`, `onload`, `xlink:href="javascript:..."`) which executes when the SVG is loaded in a browser. Since uploaded files are served from `storage/uploads/`, this is a direct stored XSS vector.

**Impact:** Any authenticated admin can upload a malicious SVG that executes JavaScript when any user views the page containing that image.

**Fix:** Remove `svg` from allowed mimes, or serve SVGs with `Content-Type: image/svg+xml` and `Content-Disposition: attachment`, or sanitize SVGs with a library like `enshrined/svg-sanitize`.

---

### 3. APP_KEY and Secrets Committed to Git

**File:** [.env](file:///c:/Users/office/Documents/shuvo/algero-frontend/.env#L29)

**Problem:** The `.env` file contains production-capable secrets:
- `APP_KEY=base64:RJQs6ciORAsJ83/Hq9a9qipY/M+jemfOBtqEeywN/bU=`
- `POSTGRES_PASSWORD=algero_secret`
- `ADMIN_PASSWORD=password`
- `RECAPTCHA_SECRET_KEY=6LeMs9osAAAAANSHd4yPNAX0UzabAgf6cPtclNuA`

While `.env` is in `.gitignore`, the `.env` file itself exists in the workspace and appears to contain actual production keys. The **reCAPTCHA secret key** is a real key (starts with `6Le...`), not a placeholder.

> [!CAUTION]
> If this `.env` was ever committed (even accidentally in a past commit), all secrets should be rotated immediately.

**Fix:**
- Verify `.env` was **never** committed: `git log --all -- .env`
- Rotate the APP_KEY: `php artisan key:generate`
- Change all DB passwords and reCAPTCHA keys
- Use `ADMIN_PASSWORD` only in development; remove from production

---

## 🟠 High Issues

### 4. No Rate Limiting on Public Endpoints

**Files:** [api.php](file:///c:/Users/office/Documents/shuvo/algero-frontend/backend/routes/api.php), [web.php](file:///c:/Users/office/Documents/shuvo/algero-frontend/backend/routes/web.php)

**Problem:** No `throttle` middleware on any route:
- `POST /api/login` — Brute-force attacks
- `POST /api/register` — Mass account creation spam
- `POST /api/subscribe` — Email spam
- `POST /api/contact` — Contact form flooding
- `POST /admin/login` — Admin panel brute-force

reCAPTCHA helps for subscribe/contact, but is insufficient alone (tokens can be farmed).

**Fix:**
```php
// api.php
Route::middleware('throttle:5,1')->group(function () {
    Route::post('/login', [AuthController::class, 'login']);
    Route::post('/register', [AuthController::class, 'register']);
});

Route::middleware('throttle:10,1')->group(function () {
    Route::post('/subscribe', [SubscriberController::class, 'subscribe']);
    Route::post('/contact', [ContactLeadController::class, 'store']);
});
```

---

### 5. Lead Status Update Not Validated (Web Route)

**File:** [AdminController.php:510-514](file:///c:/Users/office/Documents/shuvo/algero-frontend/backend/app/Http/Controllers/AdminController.php#L510-L514)

**Problem:** The web route `updateLeadStatus` directly uses `$request->input('status')` without validation:
```php
$lead->update(['status' => $request->input('status')]);
```
While the API version validates `'status' => 'required|in:new,contacted,closed'`, the web route does not. An attacker can set status to any arbitrary value.

**Fix:** Add validation matching the API controller.

---

### 6. API Write Routes Lack Role Enforcement

**File:** [api.php:50-78](file:///c:/Users/office/Documents/shuvo/algero-frontend/backend/routes/api.php#L50-L78)

**Problem:** Protected API routes only require `auth:sanctum` but don't check role or active status. Any authenticated user (even one with `is_active=false` if they obtained a token before deactivation) can:
- Create/update/delete blogs, team members, services
- Access subscriber and lead data

**Fix:** Add middleware to verify `is_active` and enforce role checks on destructive API endpoints.

---

### 7. `resolveContentModel` Type Parameter Injection

**File:** [AdminController.php:597-604](file:///c:/Users/office/Documents/shuvo/algero-frontend/backend/app/Http/Controllers/AdminController.php#L597-L604)

**Problem:** The `$type` parameter from URL (`/approvals/{type}/{id}/approve`) is used in a `match` expression. While `match` with `default => abort(404)` is safe, the URL parameter `{type}` is not validated in the route definition. A malicious request with a crafted type string generates a 404 rather than exploiting anything, but it's a defense-in-depth gap.

**Fix:** Constrain the route parameter:
```php
Route::post('/approvals/{type}/{id}/approve', ...)->where('type', 'blog|team|project');
```

---

## 🟡 Medium Issues

### 8. View Count Inflatable (No Deduplication)

**Files:** [BlogController.php](file:///c:/Users/office/Documents/shuvo/algero-frontend/backend/app/Http/Controllers/Api/BlogController.php), [ProjectController.php](file:///c:/Users/office/Documents/shuvo/algero-frontend/backend/app/Http/Controllers/Api/ProjectController.php)

**Problem:** `$blog->increment('views')` is called on every request. An attacker can inflate view counts by repeatedly hitting `/api/blogs/{slug}` or `/api/projects/{slug}`.

**Fix:** Deduplicate by IP + session (use cache):
```php
$cacheKey = 'views:blog:' . $blog->id . ':' . $request->ip();
if (!Cache::has($cacheKey)) {
    $blog->increment('views');
    Cache::put($cacheKey, true, now()->addMinutes(30));
}
```

---

### 9. Missing `url` Validation on URL Fields

**Files:** [AdminController storeProject/updateProject](file:///c:/Users/office/Documents/shuvo/algero-frontend/backend/app/Http/Controllers/AdminController.php#L389-L449)

**Problem:** Fields like `url`, `github_url`, `demo_url`, `image`, `social_linkedin` are validated as `string|max:500` but not as valid URLs. This allows injecting `javascript:` URIs:
```
url = "javascript:alert('XSS')"
```
These values are rendered as `href` attributes in both admin views and the public frontend.

**Fix:** Use the `url` validation rule:
```php
'url'        => 'nullable|url|max:500',
'github_url' => 'nullable|url|max:500',
'demo_url'   => 'nullable|url|max:500',
```

---

### 10. CORS Allows Wildcard Headers

**File:** [cors.php](file:///c:/Users/office/Documents/shuvo/algero-frontend/backend/config/cors.php)

```php
'allowed_methods' => ['*'],
'allowed_headers' => ['*'],
```

**Problem:** While origins are restricted, wildcard methods/headers is overly permissive.

**Fix:** Restrict to the methods and headers actually needed:
```php
'allowed_methods' => ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
'allowed_headers' => ['Content-Type', 'Authorization', 'Accept', 'X-Requested-With', 'X-CSRF-TOKEN'],
```

---

### 11. Weak Default Admin Password

**File:** [.env:40](file:///c:/Users/office/Documents/shuvo/algero-frontend/.env#L40)

```
ADMIN_PASSWORD=password
```

Even for a seeder, this is dangerous if `PromoteSuperAdminSeeder` is run on production with this value.

**Fix:** Use a strong password or generate one at seed time and print it to console.

---

### 12. APP_DEBUG=true in Production

**File:** [.env:9](file:///c:/Users/office/Documents/shuvo/algero-frontend/.env#L9)

```
APP_DEBUG=true
```

**Problem:** If deployed to production with this value, Laravel exposes full stack traces, environment variables, and database credentials on error pages.

**Fix:** Ensure `APP_DEBUG=false` and `APP_ENV=production` in production.

---

## 🔵 Low Issues

### 13. No `max` on Reorder Array Size
The `reorderTeamMembers` endpoint accepts an unbounded `order` array. Sending thousands of items causes N database queries.

**Fix:** Add `'order' => 'required|array|max:200'`

---

### 14. Session Fixation — Partial Protection
Login regenerates sessions ✅, but the register endpoint doesn't log users in, so this is fine. The logout properly invalidates and regenerates tokens ✅.

---

### 15. Missing `Content-Security-Policy` Header
No CSP headers are set, making XSS more impactful if it occurs.

**Fix:** Add CSP via middleware or web server config.

---

### 16. Uploaded Files Path Disclosure
`uploadImage` returns `'path' => $path` which exposes internal storage paths.

**Fix:** Return only the public URL, not the internal path.

---

## ✅ What's Done Well

| Area | Status |
|------|--------|
| Password hashing (bcrypt 12 rounds) | ✅ Secure |
| Session regeneration on login/logout | ✅ Correct |
| CSRF protection on all web routes | ✅ Enabled |
| `$fillable` used (no mass-assignment) | ✅ Safe |
| `password` in `$hidden` | ✅ Never exposed |
| Super Admin middleware | ✅ Properly enforced |
| Inactive user check on login | ✅ Both web + API |
| SQL injection protection (Eloquent) | ✅ No raw queries |

---

## Priority Action Plan

| Priority | Action | Effort |
|----------|--------|--------|
| **P0** | Sanitize HTML content (DOMPurify / mews/purifier) | 30 min |
| **P0** | Remove `svg` from upload mimes | 1 min |
| **P0** | Rotate secrets if .env was ever committed | 15 min |
| **P1** | Add `throttle` to login/register/public POST routes | 10 min |
| **P1** | Add validation to web `updateLeadStatus` | 2 min |
| **P1** | Add role/active middleware to API write routes | 15 min |
| **P1** | Use `url` validation rule on URL fields | 5 min |
| **P2** | Deduplicate view counts | 15 min |
| **P2** | Restrict CORS headers/methods | 5 min |
| **P2** | Set `APP_DEBUG=false` for production | 1 min |
| **P2** | Add CSP headers | 10 min |
| **P3** | Constrain route params, cap array sizes | 5 min |
