---
doc_tier: 1
doc_type: report
doc_status: archived
created: 2026-01-03
last_reviewed: 2026-01-06
owner: platform-team
tags: [auth, api-contract, dashboard]
---

# Auth API Contract Fixes

## Investigation Summary

Reviewed the auth API contract between Rails API (port 3050) and Next.js dashboard (port 3051) to identify mismatches causing authentication issues.

## Critical Issues Found

### 1. ❌ CRITICAL: use-auth-query.ts Response Envelope Mismatch

**Problem**: Dashboard hook expected `{ data: User }` but Rails API returns `{ user: User }`

**Files**:
- `/Users/felixp/code/karibew/crewkit/dashboard/src/hooks/use-auth-query.ts`

**Rails API response** (`GET /api/v1/auth/me`):
```json
{
  "user": {
    "id": "external_id",
    "name": "John Doe",
    "email": "john@example.com",
    "platform_role": "intermediate",
    "created_at": "2025-01-15T..."
  }
}
```

**Before (incorrect)**:
```typescript
return apiClient<{ data: User }>('/api/v1/auth/me', ...)
// Later: user: data?.data  ❌ WRONG - data.data is undefined
```

**After (fixed)**:
```typescript
return apiClient<{ user: User }>('/api/v1/auth/me', ...)
// Later: user: data?.user  ✅ CORRECT
```

**Impact**: This was causing `undefined` user data on page refresh, breaking protected routes.

---

### 2. ⚠️ MEDIUM: AuthTokens Type Incomplete

**Problem**: TypeScript type didn't include fields that Rails API returns

**Files**:
- `/Users/felixp/code/karibew/crewkit/dashboard/src/types/api.ts`

**Rails API response** (`POST /api/v1/auth/refresh`, `POST /api/v1/auth/login`, etc.):
```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "abc123...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**Before (incomplete)**:
```typescript
export interface AuthTokens {
  access_token: string
  refresh_token: string
}
```

**After (complete)**:
```typescript
export interface AuthTokens {
  access_token: string
  refresh_token: string
  token_type?: string // Optional - API returns "Bearer"
  expires_in?: number // Optional - API returns 3600 (seconds)
}
```

**Impact**: Extra fields were ignored (no runtime error), but TypeScript couldn't access them for better token expiry handling.

---

### 3. 💡 LOW: Token Expiry Using Client Clock

**Problem**: Client calculated token expiry time instead of using server-provided value

**Files**:
- `/Users/felixp/code/karibew/crewkit/dashboard/src/lib/auth/context.tsx`

**Before (client-only)**:
```typescript
const expiryTime = Date.now() + TOKEN_EXPIRY_SECONDS * 1000
```

**After (server-aware)**:
```typescript
const expirySeconds = tokens.expires_in || TOKEN_EXPIRY_SECONDS
const expiryTime = Date.now() + expirySeconds * 1000
```

**Impact**: Better handling of clock skew and flexible token expiry times from server.

---

## Verified Correct Patterns

### ✅ Organizations API

**Rails**: `GET /api/v1/organizations`
```json
{
  "data": [
    {
      "id": "org_abc123",
      "name": "Acme Corp",
      "slug": "acme-corp",
      "created_at": "...",
      "permission_level": "owner"
    }
  ]
}
```

**Dashboard**:
```typescript
const response = await apiClient<{ data: Organization[] }>('/api/v1/organizations', ...)
return response.data  // ✅ CORRECT
```

### ✅ Login/Register

**Rails**: `POST /api/v1/auth/login`, `POST /api/v1/auth/register`
```json
{
  "access_token": "...",
  "refresh_token": "...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "user": {
    "id": "...",
    "name": "...",
    "email": "...",
    "platform_role": "..."
  }
}
```

**Dashboard**:
```typescript
export interface LoginResponse extends AuthTokens {
  user: User
}
// ✅ CORRECT - includes user field
```

### ✅ CORS Configuration

**Rails** (`config/initializers/cors.rb`):
```ruby
origins ENV.fetch("ALLOWED_ORIGINS", "http://localhost:3000,http://localhost:3051").split(",")

resource "/api/*",
  headers: :any,
  methods: [:get, :post, :put, :patch, :delete, :options, :head],
  expose: ["Authorization"],
  credentials: true  # ✅ Allows cookies
```

**Dashboard** (`lib/api/client.ts`):
```typescript
const res = await fetch(url, {
  ...options,
  headers,
  credentials: 'include',  // ✅ CORRECT - sends cookies
})
```

### ✅ Token Storage Strategy

Dashboard stores tokens in:
- **sessionStorage**: `access_token`, `token_expiry` (session-only)
- **localStorage** OR **sessionStorage**: `refresh_token` (based on "Remember Me")
- **cookies**: All tokens (for middleware access)

Retrieval priority:
```typescript
const accessToken = sessionStorage.getItem(TOKEN_KEY)
const refreshToken = sessionStorage.getItem(REFRESH_TOKEN_KEY) || localStorage.getItem(REFRESH_TOKEN_KEY)
```

This is correct - sessionStorage takes priority, fallback to localStorage.

---

## Files Modified

1. `/Users/felixp/code/karibew/crewkit/dashboard/src/hooks/use-auth-query.ts`
   - Fixed response envelope: `{ data: User }` → `{ user: User }`
   - Fixed user extraction: `data?.data` → `data?.user`

2. `/Users/felixp/code/karibew/crewkit/dashboard/src/types/api.ts`
   - Added `token_type?: string` to `AuthTokens`
   - Added `expires_in?: number` to `AuthTokens`

3. `/Users/felixp/code/karibew/crewkit/dashboard/src/lib/auth/context.tsx`
   - Updated `storeTokens()` to use server-provided `expires_in`
   - Updated `refreshAccessToken()` to use server-provided `expires_in`

---

## Testing

Run the test script to verify all endpoints return correct formats:

```bash
# Start Rails API (terminal 1)
cd ~/code/karibew/crewkit/api && bin/dev

# Start Next.js dashboard (terminal 2)
cd ~/code/karibew/crewkit/dashboard && npm run dev

# Run tests (terminal 3)
cd ~/code/karibew/crewkit && ./test_auth_contract.sh
```

**Expected output**:
```
✓ User registered successfully
✓ Correct format: { user: { ... } }
✓ Token refreshed successfully
✓ Correct format: { data: [...] }
✓ Organization created successfully
✓ Logout successful
```

---

## Manual Testing

Test auth flow in dashboard:

1. **Register**: http://localhost:3051/register
   - Create account → should redirect to `/kit`
   - Check sessionStorage has `crewkit_access_token`

2. **Login**: http://localhost:3051/login
   - Login with created account
   - Should redirect to `/kit`
   - Check sessionStorage AND localStorage (if "Remember Me" checked)

3. **Protected Routes**: http://localhost:3051/kit
   - Should show dashboard (not redirect to login)
   - Check Network tab: `GET /api/v1/auth/me` returns `{ user: {...} }`

4. **Refresh Token**:
   - Wait 1 hour (or manually expire token in sessionStorage)
   - Dashboard should auto-refresh token
   - Check Network tab: `POST /api/v1/auth/refresh`

5. **Organization Creation**: http://localhost:3051/kit/organizations
   - Create new organization
   - Should appear in dropdown
   - Check Network tab: `POST /api/v1/organizations` returns `{ data: {...} }`

6. **Logout**: Click logout button
   - Should redirect to `/login`
   - sessionStorage/localStorage cleared
   - Check Network tab: `DELETE /api/v1/auth/logout`

---

## API Contract Reference

### Authentication Endpoints

| Endpoint | Method | Request | Response | Notes |
|----------|--------|---------|----------|-------|
| `/api/v1/auth/register` | POST | `{email, password, password_confirmation, name}` | `{access_token, refresh_token, token_type, expires_in, user}` | Auto-login after register |
| `/api/v1/auth/login` | POST | `{email, password}` | `{access_token, refresh_token, token_type, expires_in, user}` | Email/password login |
| `/api/v1/auth/me` | GET | - | `{user: {...}}` | Requires `Authorization: Bearer <token>` |
| `/api/v1/auth/refresh` | POST | `{refresh_token}` | `{access_token, refresh_token, token_type, expires_in}` | Rotates refresh token |
| `/api/v1/auth/logout` | DELETE | `{refresh_token}` | `{message}` | Revokes refresh token |

### Organization Endpoints

| Endpoint | Method | Request | Response | Notes |
|----------|--------|---------|----------|-------|
| `/api/v1/organizations` | GET | - | `{data: [...]}` | List user's orgs |
| `/api/v1/organizations` | POST | `{organization: {name, slug}}` | `{data: {...}}` | Create org (user becomes owner) |

### Common Response Formats

**Success (data)**:
```json
{
  "data": {...} | [...]
}
```

**Success (message)**:
```json
{
  "message": "Success message"
}
```

**Error**:
```json
{
  "error": "Error message",
  "details": {...}
}
```

---

## Common Pitfalls

### ❌ Don't mix response envelopes

```typescript
// BAD - inconsistent
apiClient<{ data: User }>('/api/v1/auth/me', ...)  // Wrong - API uses "user"
apiClient<{ user: User }>('/api/v1/organizations', ...)  // Wrong - API uses "data"

// GOOD - match API contract
apiClient<{ user: User }>('/api/v1/auth/me', ...)  // ✓
apiClient<{ data: Organization[] }>('/api/v1/organizations', ...)  // ✓
```

### ❌ Don't access tokens without fallback

```typescript
// BAD - only checks sessionStorage
const token = sessionStorage.getItem('crewkit_refresh_token')

// GOOD - checks both (rememberMe uses localStorage)
const token = sessionStorage.getItem('crewkit_refresh_token') ||
              localStorage.getItem('crewkit_refresh_token')
```

### ❌ Don't ignore server-provided expiry

```typescript
// BAD - client-side only (clock skew issues)
const expiryTime = Date.now() + 3600 * 1000

// GOOD - use server value
const expirySeconds = tokens.expires_in || 3600
const expiryTime = Date.now() + expirySeconds * 1000
```

---

## Next Steps

- [x] Fix critical response envelope mismatch in `use-auth-query.ts`
- [x] Add missing fields to `AuthTokens` type
- [x] Use server-provided `expires_in` for better accuracy
- [ ] Test E2E auth flow (register → login → protected route → refresh → logout)
- [ ] Test organization creation after login
- [ ] Verify token refresh on page reload
- [ ] Verify "Remember Me" persists across browser sessions

---

## Related Files

**Rails API**:
- `api/app/controllers/api/v1/auth_controller.rb` - Auth endpoints
- `api/app/controllers/api/v1/base_controller.rb` - JWT verification
- `api/app/controllers/api/v1/organizations_controller.rb` - Org endpoints
- `api/config/initializers/cors.rb` - CORS config

**Dashboard**:
- `dashboard/src/lib/api/auth.ts` - Auth API client
- `dashboard/src/lib/api/client.ts` - Base API client
- `dashboard/src/lib/auth/context.tsx` - Auth context provider
- `dashboard/src/lib/auth/organization-context.tsx` - Org context provider
- `dashboard/src/hooks/use-auth-query.ts` - TanStack Query hook for /auth/me
- `dashboard/src/types/api.ts` - TypeScript types

**Tests**:
- `test_auth_contract.sh` - Bash script to verify API contract
