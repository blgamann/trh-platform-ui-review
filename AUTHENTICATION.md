# Authentication System Documentation

This document provides a comprehensive overview of the authentication system implementation in the TRH Platform UI.

## Architecture Overview

The authentication system is built using a layered architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    Next.js Middleware                        │
│              (Server-side Route Protection)                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    AuthProvider (Context)                    │
│              (Application-wide Auth State)                   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                     useAuth Hook                             │
│        (State Management & Business Logic)                   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    AuthService                               │
│            (API Calls & Token Operations)                    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                     API Client                               │
│        (Axios with Token Injection)                          │
└─────────────────────────────────────────────────────────────┘
```

## Authentication Flow

### 1. Login Flow

```
User enters credentials
       ↓
AuthForm validates input (Zod schema)
       ↓
useAuth.login() called
       ↓
AuthService.login() → POST /auth/login
       ↓
Backend validates credentials
       ↓
Token + User data returned
       ↓
Token stored (Cookie + localStorage)
       ↓
Auth state updated (isAuthenticated: true)
       ↓
Redirect to dashboard or original destination
```

### 2. Token Validation Flow (Page Load)

```
App loads / Page refresh
       ↓
AuthProvider mounts → useAuth runs
       ↓
useEffect checks for stored token
       ↓
Token found?
  ├─ Yes → AuthService.getCurrentUser()
  │         ↓
  │    GET /auth/profile (with token)
  │         ↓
  │    Token valid?
  │      ├─ Yes → Set user data & isAuthenticated: true
  │      └─ No → Remove token, isAuthenticated: false
  │                ↓
  │           AuthenticatedLayout detects !isAuthenticated
  │                ↓
  │           Redirect to /auth
  └─ No → isAuthenticated: false
```

### 3. Token Expiration Handling

```
User performs action → API request
       ↓
API returns 401 Unauthorized
       ↓
Axios interceptor catches error
       ↓
Token removed from storage
       ↓
Page redirects to /auth
       ↓
User must log in again
```

## Core Components

### 1. AuthService (`src/features/auth/services/authService.ts`)

Singleton service handling all authentication operations.

**Key Methods:**
- `login(credentials)` - Authenticates user, returns token and user data
- `getCurrentUser()` - Validates token and fetches user profile
- `getStoredToken()` - Retrieves token from cookies or localStorage
- `storeToken(token)` - Stores token in both cookie and localStorage
- `removeToken()` - Clears all authentication tokens

**Features:**
- Dual storage strategy (cookie for SSR, localStorage for client)
- Zod schema validation for API responses
- Comprehensive error handling with user-friendly messages
- Singleton pattern for consistent state

### 2. useAuth Hook (`src/features/auth/hooks/useAuth.ts`)

React hook managing authentication state and operations.

**Provides:**
```typescript
{
  user: User | null
  token: string | null
  isAuthenticated: boolean
  isLoading: boolean
  isLoggingIn: boolean
  login: (credentials) => void
  logout: () => void
}
```

**Key Features:**
- React Query integration for API mutations
- Automatic token validation on mount
- Toast notifications for user feedback
- Automatic redirect handling

### 3. AuthProvider (`src/providers/auth-provider.tsx`)

React Context provider for application-wide auth state.

**Usage:**
```tsx
// Wrap your app
<AuthProvider>
  <YourApp />
</AuthProvider>

// Access auth context anywhere
const { isAuthenticated, user, logout } = useAuthContext();
```

### 4. AuthForm (`src/features/auth/components/AuthForm.tsx`)

Login form component with validation.

**Features:**
- Email/password validation using react-hook-form + Zod
- Remember me functionality
- Loading states
- Error display
- Forgot password link

### 5. Schemas (`src/features/auth/schemas.ts`)

Zod validation schemas for type safety.

**Key Schemas:**
```typescript
// Login request validation
const loginRequestSchema = z.object({
  email: z.string().email(),
  password: z.string().min(6)
})

// User roles
enum Role {
  Admin = "admin",
  User = "user"
}

// User model
const userSchema = z.object({
  id: z.string(),
  email: z.string().email(),
  role: z.nativeEnum(Role)
})
```

## Token Management

### Storage Strategy

The system uses a **dual storage approach**:

1. **HTTP-only Cookie** (Primary for SSR)
   - Name: `auth-token`
   - Path: `/`
   - Max-Age: 7 days
   - SameSite: Strict
   - Used by Next.js middleware for server-side route protection

2. **localStorage** (Fallback for client-side)
   - Key: `accessToken`
   - Used by API client for request authentication
   - Maintains backward compatibility

### Token Lifecycle

```
Login → Token stored in both locations
  ↓
Each API request → Token added to Authorization header
  ↓
Token expires → API returns 401
  ↓
Interceptor catches 401 → Remove tokens
  ↓
Redirect to /auth
```

### Token Injection

The API client (`src/lib/api.ts`) automatically injects tokens:

```typescript
// Request interceptor
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem("accessToken");
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

## Route Protection

### Multi-Layer Protection

The system implements **three layers** of route protection:

#### Layer 1: Next.js Middleware (`middleware.ts`)

**Server-side protection** - First line of defense

```typescript
// Protected routes requiring authentication
const protectedRoutes = [
  "/dashboard",
  "/admin",
  "/settings",
  "/explore",
  "/analytics",
  "/users",
  "/configuration",
  "/notification",
  "/support"
];
```

**Behavior:**
- Checks for `auth-token` cookie
- Redirects to `/auth?redirect=[original-path]` if no token
- Redirects `/` to `/dashboard` if authenticated
- Bypasses API routes and static assets

#### Layer 2: AuthenticatedLayout (`src/components/layout/AuthenticatedLayout.tsx`)

**Client-side validation** - Validates token authenticity

```tsx
useEffect(() => {
  if (!isLoading && !isAuthenticated) {
    router.push("/auth");
  }
}, [isAuthenticated, isLoading, router]);
```

**Behavior:**
- Validates token with backend via `/auth/profile`
- Shows loading spinner during validation
- Redirects if token is invalid or expired
- Renders sidebar and content if authenticated

#### Layer 3: Component-Level Protection

**Granular access control**

**ProtectedRoute** (`src/components/auth/ProtectedRoute.tsx`):
```tsx
<ProtectedRoute redirect="/dashboard">
  <YourComponent />
</ProtectedRoute>
```

**RequireRole** (`src/components/auth/RequireRole.tsx`):
```tsx
<RequireRole roles={[Role.Admin]}>
  <AdminPanel />
</RequireRole>
```

**withAuth HOC** (`src/components/auth/withAuth.tsx`):
```tsx
const ProtectedComponent = withAuth(MyComponent);
```

### Protection Comparison

| Layer | Execution | Checks | Best For |
|-------|-----------|--------|----------|
| Middleware | Server (Edge) | Token existence | Initial route access, SEO |
| Layout | Client (React) | Token validity | Token expiration, UX |
| Component | Client (React) | Role-based access | Fine-grained permissions |

## API Integration

### API Client Configuration (`src/lib/api.ts`)

Centralized Axios instance with authentication integration.

**Configuration:**
```typescript
const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_BASE_URL || "http://localhost:3001/api",
  timeout: 10000,
  headers: {
    "Content-Type": "application/json"
  }
});
```

**Request Interceptor:**
- Automatically adds `Authorization: Bearer <token>` header
- Retrieves token from localStorage

**Response Interceptor:**
- Handles 401 errors → Clears token and redirects to login
- Handles 403 errors → Shows "Forbidden" message
- Provides structured error responses

**Helper Functions:**
```typescript
apiGet<T>(endpoint: string): Promise<T>
apiPost<T>(endpoint: string, data: any): Promise<T>
apiPut<T>(endpoint: string, data: any): Promise<T>
apiPatch<T>(endpoint: string, data: any): Promise<T>
apiDelete<T>(endpoint: string): Promise<T>
```

### API Endpoints

| Endpoint | Method | Purpose | Auth Required |
|----------|--------|---------|---------------|
| `/auth/login` | POST | Authenticate user | No |
| `/auth/profile` | GET | Get current user | Yes |

## File Structure

```
trh-platform-ui/
├── middleware.ts                          # Server-side route protection
├── src/
│   ├── app/
│   │   └── auth/
│   │       └── page.tsx                   # Login page
│   ├── components/
│   │   ├── auth/
│   │   │   ├── ProtectedRoute.tsx         # Route access control
│   │   │   ├── RequireRole.tsx            # Role-based access
│   │   │   ├── withAuth.tsx               # HOC for auth
│   │   │   └── index.ts                   # Auth component exports
│   │   ├── layout/
│   │   │   └── AuthenticatedLayout.tsx    # Layout with auth check
│   │   └── molecules/
│   │       └── LogoutButton.tsx           # Logout button component
│   ├── features/
│   │   └── auth/
│   │       ├── components/
│   │       │   └── AuthForm.tsx           # Login form
│   │       ├── hooks/
│   │       │   └── useAuth.ts             # Auth state hook
│   │       ├── services/
│   │       │   └── authService.ts         # Auth API service
│   │       ├── schemas.ts                 # Zod schemas & types
│   │       └── index.ts                   # Auth feature exports
│   ├── providers/
│   │   ├── auth-provider.tsx              # Auth context provider
│   │   └── index.ts                       # Provider exports
│   └── lib/
│       └── api.ts                         # API client with auth
```

## Usage Examples

### 1. Protect a Page

```tsx
// app/dashboard/page.tsx
import { AuthenticatedLayout } from "@/components/layout";

export default function DashboardPage() {
  return (
    <AuthenticatedLayout>
      <h1>Dashboard</h1>
      {/* Your content */}
    </AuthenticatedLayout>
  );
}
```

### 2. Access Auth State

```tsx
"use client";
import { useAuthContext } from "@/providers";

export function UserProfile() {
  const { user, isAuthenticated, logout } = useAuthContext();

  if (!isAuthenticated) {
    return <div>Please log in</div>;
  }

  return (
    <div>
      <p>Welcome, {user?.email}</p>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

### 3. Protect Component by Role

```tsx
import { RequireRole } from "@/components/auth";
import { Role } from "@/features/auth/schemas";

export function AdminPanel() {
  return (
    <RequireRole roles={[Role.Admin]}>
      <div>Admin-only content</div>
    </RequireRole>
  );
}
```

### 4. Make Authenticated API Requests

```tsx
import { apiGet, apiPost } from "@/lib/api";

// Token is automatically included
async function fetchUserData() {
  const data = await apiGet("/users/me");
  return data;
}

async function updateProfile(updates: any) {
  const result = await apiPost("/users/profile", updates);
  return result;
}
```

### 5. Custom Login Form

```tsx
"use client";
import { useAuthContext } from "@/providers";
import { useState } from "react";

export function CustomLoginForm() {
  const { login, isLoggingIn } = useAuthContext();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    login({ email, password });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
      />
      <button type="submit" disabled={isLoggingIn}>
        {isLoggingIn ? "Logging in..." : "Login"}
      </button>
    </form>
  );
}
```
