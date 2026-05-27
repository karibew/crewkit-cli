# Authentication E2E Integration Analysis

**Date**: 2025-12-08 (Updated: 2026-01-02)
**Dashboard**: Next.js 16 (Port 3051)
**API**: Rails 8 (Port 3050)

> **Note**: Token TTL increased from 1 hour to 4 hours (January 2026) to support longer CLI sessions.

## Executive Summary

This document analyzes the authentication flow between the Next.js dashboard and Rails API, identifies integration points, and documents testing strategy.

## Authentication Architecture

### Components

1. **Dashboard (Next.js)**
   - Location: `dashboard/src/`
   - Auth Context: `lib/auth/context.tsx`
   - API Client: `lib/api/auth.ts`, `lib/api/client.ts`
   - Forms: `components/features/auth/`

2. **API (Rails)**
   - Controller: `api/app/controllers/api/v1/auth_controller.rb`
   - Routes: `/api/v1/auth/*`

### Auth Flow Diagram

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│  Dashboard  │         │   API        │         │  Database   │
│  (Next.js)  │         │   (Rails)    │         │  (Postgres) │
└─────────────┘         └──────────────┘         └─────────────┘
       │                       │                        │
       │  POST /auth/register  │                        │
       │ ─────────────────────>│  Create user           │
       │                       │ ──────────────────────>│
       │                       │                        │
       │  200 + tokens         │  User created          │
       │ <─────────────────────│ <──────────────────────│
       │                       │                        │
       │  Store tokens in      │                        │
       │  sessionStorage/      │                        │
       │  localStorage         │                        │
       │                       │                        │
       │  GET /auth/me         │                        │
       │  (with Bearer token)  │                        │
       │ ─────────────────────>│  Validate JWT          │
       │                       │                        │
       │  200 + user data      │                        │
       │ <─────────────────────│                        │
       │                       │                        │
```

## Token Management

### Token Types

1. **Access Token** (JWT)
   - Expires: 4 hours (14400 seconds)
   - Storage: `sessionStorage.crewkit_access_token`
   - Format: JWT (3 parts: header.payload.signature)
   - Used in: `Authorization: Bearer <token>` header

2. **Refresh Token**
   - Expires: 30 days
   - Storage:
     - If "Remember Me": `localStorage.crewkit_refresh_token`
     - Otherwise: `sessionStorage.crewkit_refresh_token`
   - Used to: Get new access token without re-login
   - Security: Rotation on each refresh (old token revoked)

### Token Refresh Strategy

```typescript
// Auto-refresh triggered when:
// 1. Token expires in < 5 minutes
// 2. Check runs every 60 seconds (in AuthContext)
// 3. On page reload

const shouldRefreshToken = () => {
  const expiry = sessionStorage.getItem('crewkit_token_expiry')
  const timeUntilExpiry = expiry - Date.now()
  return timeUntilExpiry < (5 * 60 * 1000) // 5 minutes
}
```

## API Endpoints

### POST /api/v1/auth/register
**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "password_confirmation": "SecurePass123!",
  "name": "John Doe"
}
```

**Response (201):**
```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "abc123...",
  "token_type": "Bearer",
  "expires_in": 14400,
  "user": {
    "id": "ext_abc123",
    "name": "John Doe",
    "email": "user@example.com",
    "platform_role": "developer"
  }
}
```

**Error (422):**
```json
{
  "error": "Validation failed",
  "details": {
    "email": ["has already been taken"],
    "password": ["is too short (minimum 8 characters)"]
  }
}
```

### POST /api/v1/auth/login
**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!"
}
```

**Response (200):**
Same as registration

**Error (401):**
```json
{
  "error": "Invalid email or password"
}
```

### GET /api/v1/auth/me
**Headers:**
```
Authorization: Bearer <access_token>
```

**Response (200):**
```json
{
  "user": {
    "id": "ext_abc123",
    "name": "John Doe",
    "email": "user@example.com",
    "platform_role": "developer",
    "created_at": "2025-12-08T10:00:00Z"
  }
}
```

**Error (401):**
```json
{
  "error": "Invalid or expired token"
}
```

### POST /api/v1/auth/refresh
**Request:**
```json
{
  "refresh_token": "abc123..."
}
```

**Response (200):**
```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "def456...",  // New token (rotation)
  "token_type": "Bearer",
  "expires_in": 14400
}
```

**Error (401):**
```json
{
  "error": "Refresh token has been revoked"
}
```

### DELETE /api/v1/auth/logout
**Request:**
```json
{
  "refresh_token": "abc123..."
}
```

**Response (200):**
```json
{
  "message": "Token successfully revoked"
}
```

## Dashboard Implementation

### AuthContext (`lib/auth/context.tsx`)

**Key Features:**
- Token storage in sessionStorage/localStorage
- Automatic token refresh (5 min before expiry)
- Remember me functionality
- Auto-redirect after login/register

**Issues Found:**

1. **❌ Inconsistent hook exports**
   - `hooks/use-auth.ts` exports `useAuth()` that uses TanStack Query
   - `lib/auth/context.tsx` also exports `useAuth()` with different API
   - **Impact**: Name collision, unclear which to use
   - **Fix**: Rename one (e.g., `useAuthQuery` vs `useAuth`)

2. **❌ Missing Authorization header in useAuth hook**
   - `hooks/use-auth.ts` calls `/api/v1/auth/me` without Bearer token
   - **Impact**: Endpoint will return 401
   - **Fix**: Pass access token from storage

3. **✓ Token expiry calculation** (FIXED)
   - Dashboard now uses server-provided `expires_in` from API responses
   - Falls back to 4-hour default (`TOKEN_EXPIRY_SECONDS = 14400`)
   - Cookies are set with correct maxAge from server response

4. **⚠️ No CORS configuration visible**
   - API needs CORS headers for `localhost:3051`
   - Check `api/config/initializers/cors.rb`

### API Client (`lib/api/client.ts`)

**Configuration:**
```typescript
const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3050'
```

**Features:**
- ✓ Includes credentials (cookies)
- ✓ JSON content-type
- ✓ Error parsing
- ✓ Throws ApiError with status/details

**Issues Found:**

1. **❌ apiClient doesn't automatically add auth header**
   - Each call must manually add `Authorization: Bearer ...`
   - **Fix**: Add interceptor or wrapper function

## API Implementation

### AuthController (`api/app/controllers/api/v1/auth_controller.rb`)

**Features:**
- ✓ Email/password login
- ✓ User registration (with skip_confirmation)
- ✓ JWT token generation (HS256)
- ✓ Refresh token rotation
- ✓ Token revocation
- ✓ /me endpoint
- ✓ Password reset flow
- ✓ Passkey (WebAuthn) support
- ✓ Device flow for CLI

**Security:**
- ✓ Timing-safe password validation
- ✓ Token rotation on refresh
- ✓ Refresh token revocation
- ✓ IP tracking
- ✓ User agent tracking

**Issues Found:**

1. **✓ No issues in API implementation** - well structured

### CORS Configuration

**Check needed:**
```bash
cat api/config/initializers/cors.rb
```

Expected:
```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'localhost:3051', 'http://localhost:3051'
    resource '/api/*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options],
      credentials: true
  end
end
```

## Test Coverage

### Manual Test Script

Location: `e2e/manual-auth-test.sh`

**Tests:**
1. ✓ User registration
2. ✓ Login with email/password
3. ✓ Get current user (/me)
4. ✓ Token refresh
5. ✓ Verify new token works
6. ✓ Logout (revoke token)
7. ✓ Revoked token rejected
8. ✓ Invalid credentials rejected

**Run:**
```bash
cd ~/code/karibew/crewkit/dashboard
chmod +x e2e/manual-auth-test.sh
./e2e/manual-auth-test.sh
```

### Playwright E2E Tests

Location: `e2e/auth.spec.ts`

**Tests:**
1. User registration flow
2. Login flow with token storage
3. Authenticated /me endpoint
4. Token refresh flow
5. Logout flow
6. Invalid credentials
7. Expired token handling
8. Token persistence (remember me)
9. Registration validation
10. Concurrent refresh handling

**Run:**
```bash
# First install browsers
npx playwright install chromium

# Run tests
npm run test:e2e

# Debug mode
npm run test:e2e:debug

# UI mode
npm run test:e2e:ui
```

## Integration Issues Summary

### Critical Issues (Must Fix)

1. **Hook name collision** (`useAuth` in two places)
   - **Location**: `hooks/use-auth.ts` vs `lib/auth/context.tsx`
   - **Fix**: Rename `hooks/use-auth.ts` to `useAuthQuery`

2. **Missing auth header** in TanStack Query hook
   - **Location**: `hooks/use-auth.ts` line 10
   - **Fix**: Get token from storage and pass in Authorization header

3. **CORS configuration** (need to verify)
   - **Location**: `api/config/initializers/cors.rb`
   - **Fix**: Ensure localhost:3051 is allowed

### Medium Priority

4. **API client interceptor** for auth headers
   - **Location**: `lib/api/client.ts`
   - **Enhancement**: Auto-add Bearer token to requests

5. **✓ JWT expiry handling** (FIXED)
   - **Location**: `lib/auth/context.tsx`
   - **Status**: Dashboard now uses server-provided `expires_in` value

### Low Priority

6. **Error handling consistency**
   - Some errors use toast, some don't
   - Consider unified error handler

## Recommended Fixes

### Fix 1: Rename TanStack Query hook

```typescript
// hooks/use-auth-query.ts (renamed from use-auth.ts)
'use client'

import { useQuery } from '@tanstack/react-query'
import { apiClient } from '@/lib/api/client'
import type { User } from '@/types'

export function useAuthQuery() {
  const getToken = () => {
    if (typeof window === 'undefined') return null
    return sessionStorage.getItem('crewkit_access_token')
  }

  return useQuery({
    queryKey: ['auth', 'me'],
    queryFn: async () => {
      const token = getToken()
      if (!token) throw new Error('No access token')

      return apiClient<{ data: User }>('/api/v1/auth/me', {
        headers: {
          Authorization: `Bearer ${token}`,
        },
      })
    },
    retry: false,
    staleTime: 5 * 60 * 1000,
    enabled: !!getToken(),
  })
}
```

### Fix 2: CORS Configuration

Verify `api/config/initializers/cors.rb` has:

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins ENV.fetch('DASHBOARD_URL', 'http://localhost:3051').gsub(/^https?:\/\//, '')

    resource '/api/v1/*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: true,
      expose: ['Authorization']
  end
end
```

### Fix 3: API Client Interceptor

```typescript
// lib/api/client.ts
export async function apiClient<T>(
  endpoint: string,
  options: RequestInit = {}
): Promise<T> {
  const url = `${API_URL}${endpoint}`

  // Auto-add auth token if available
  const token = typeof window !== 'undefined'
    ? sessionStorage.getItem('crewkit_access_token')
    : null

  const headers: HeadersInit = {
    'Content-Type': 'application/json',
    ...options.headers,
  }

  // Add auth header if token exists and not already set
  if (token && !headers['Authorization']) {
    headers['Authorization'] = `Bearer ${token}`
  }

  const res = await fetch(url, {
    ...options,
    headers,
    credentials: 'include',
  })

  // Rest of implementation...
}
```

## Next Steps

1. **Fix critical issues** (hook collision, missing auth header, CORS)
2. **Run manual test script** to verify API endpoints work
3. **Start both servers** (API + Dashboard)
4. **Run Playwright tests** to verify full E2E flow
5. **Document results** and any additional issues found

## Running the Test Suite

### Prerequisites

```bash
# Terminal 1: Start API
cd ~/code/karibew/crewkit/api && bin/dev

# Terminal 2: Start Dashboard
cd ~/code/karibew/crewkit/dashboard && npm run dev

# Terminal 3: Verify both are running
curl http://localhost:3050/up
curl http://localhost:3051
```

### Run Tests

```bash
# Manual API test
cd ~/code/karibew/crewkit/dashboard
./e2e/manual-auth-test.sh

# Playwright E2E tests
npm run test:e2e

# Interactive mode
npm run test:e2e:ui
```

## Conclusion

The auth integration between Dashboard and API is **mostly well-architected** with JWT + refresh tokens, proper token rotation, and security measures.

**Main issues to fix:**
1. Hook name collision causing confusion
2. Missing Authorization header in TanStack Query hook
3. CORS configuration verification needed

Once these are fixed, the E2E auth flow should work seamlessly.
