# Axios & API Architecture Guide

## What is Axios?

**Axios** is a promise-based HTTP client for JavaScript. It provides:
- Clean, promise-based API
- Request/response interceptors
- Automatic JSON transformation
- Request cancellation
- TypeScript support
- Better error handling than `fetch`

### Why Axios over Fetch?

```typescript
// ❌ Fetch: More boilerplate
const response = await fetch('/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`, // Manual auth
  },
  body: JSON.stringify(data), // Manual serialization
});

if (!response.ok) {
  throw new Error(`HTTP error! status: ${response.status}`);
}

const result = await response.json(); // Manual parsing

// ✅ Axios: Cleaner, more features
const response = await axios.post('/api/users', data, {
  // Automatically adds auth via interceptor
  // Automatically serializes JSON
  // Automatically throws on error status
});

const result = response.data; // Already parsed
```

**Key Benefits**:
- Automatic JSON transformation
- Request/response interceptors (global auth, error handling)
- Better TypeScript support
- Request cancellation
- Progress tracking

---

## Project API Architecture

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    React Components                      │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│               React Query (TanStack Query)              │
│            (Caching, Refetching, State)                 │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   Service Layer                          │
│            (features/*/services/*.ts)                    │
│         Business logic, data transformation              │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                  API Helper Layer                        │
│          (lib/api.ts: apiGet, apiPost, etc.)            │
│              Generic HTTP methods                        │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                Axios Instance (apiClient)                │
│         Interceptors, base config, timeout              │
└─────────────────────────────────────────────────────────┘
                            ↓
                    Backend API Server
```

### Directory Structure

```
src/
├── lib/
│   └── api.ts                    # Axios config, interceptors, helpers
├── features/
│   ├── auth/
│   │   └── services/
│   │       └── authService.ts    # Auth API calls
│   ├── rollup/
│   │   └── services/
│   │       └── rollupService.ts  # Rollup API calls
│   └── configuration/
│       └── api-keys/
│           └── services/
│               └── apiKeysService.ts
```

**Key Principle**: Separation of concerns through layers.

---

## Axios Configuration

### Base Configuration

```typescript
import axios, { AxiosInstance } from "axios";
import { env } from "next-runtime-env";

// API base configuration
const API_BASE_URL = `${
  env("NEXT_PUBLIC_API_BASE_URL") || "http://localhost:8000"
}/api/v1/`;

// Create axios instance with default configuration
const apiClient: AxiosInstance = axios.create({
  baseURL: API_BASE_URL,
  timeout: 10000, // 10 seconds
  headers: {
    "Content-Type": "application/json",
  },
});
```

**Configuration Options Explained**:

| Option | Value | Purpose |
|--------|-------|---------|
| **baseURL** | `http://localhost:8000/api/v1/` | Base URL for all requests |
| **timeout** | 10000 ms (10 sec) | Request timeout |
| **headers** | `Content-Type: application/json` | Default headers |

**Key Learning Points**:
- Single axios instance for the entire app
- Environment-based configuration
- Default timeout prevents hanging requests
- JSON as default content type

### Why a Single Instance?

```typescript
// ✅ Good: Single configured instance
const apiClient = axios.create({ baseURL, timeout });
apiClient.get('/users'); // Uses baseURL + timeout

// ❌ Bad: Create new instance every time
axios.create({ baseURL }).get('/users');
```

Benefits:
- Consistent configuration across app
- Centralized interceptors
- Easy to modify global settings

---

## Interceptors

Interceptors allow you to modify requests/responses globally.

### Request Interceptor

```typescript
// Request interceptor
apiClient.interceptors.request.use(
  (config) => {
    // Add auth token if available
    const token =
      typeof window !== "undefined"
        ? localStorage.getItem("accessToken")
        : null;

    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }

    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);
```

**What This Does**:

```
Every API request:
1. Check if auth token exists in localStorage
2. If yes, add "Authorization: Bearer <token>" header
3. Send request with token

Example:
apiClient.get('/users')
  ↓
Request Interceptor adds: { Authorization: "Bearer abc123" }
  ↓
Actual request: GET /users with auth header
```

**Key Learning Points**:
- Automatic authentication for all requests
- No need to manually add auth header every time
- SSR-safe check: `typeof window !== "undefined"`

**Before/After Comparison**:

```typescript
// ❌ Without interceptor: Manual auth every time
const token = localStorage.getItem('accessToken');
axios.get('/users', {
  headers: { Authorization: `Bearer ${token}` }
});

// ✅ With interceptor: Automatic
apiClient.get('/users'); // Token added automatically!
```

### Response Interceptor

```typescript
// Response interceptor
apiClient.interceptors.response.use(
  (response) => {
    // Success: just return response
    return response;
  },
  (error: AxiosError) => {
    // Handle common error cases
    if (error.response?.status === 401) {
      // Unauthorized - clear token and redirect
      if (typeof window !== "undefined") {
        localStorage.removeItem("accessToken");
        // Could redirect to login here
      }
    }

    if (error.response?.status === 403) {
      // Forbidden - handle access denied
      console.error("Access denied");
    }

    return Promise.reject(error);
  }
);
```

**What This Does**:

```
Every API response:
1. If successful (2xx) → return response
2. If 401 Unauthorized → clear token, redirect to login
3. If 403 Forbidden → log error
4. Reject promise with error
```

**Key Learning Points**:
- Global error handling for common cases
- Automatic token cleanup on auth failure
- Centralized place for error logic

**Common Status Codes**:

| Status | Meaning | Handling |
|--------|---------|----------|
| **200-299** | Success | Return response |
| **401** | Unauthorized | Clear token, redirect to login |
| **403** | Forbidden | Show access denied message |
| **404** | Not Found | Handle in component |
| **500-599** | Server Error | Show error message |

---

## API Helper Functions

Generic, type-safe wrappers around HTTP methods.

### GET Request

```typescript
// Generic GET request helper
export const apiGet = async <T = any>(
  url: string,
  config?: AxiosRequestConfig
): Promise<ApiResponse<T>> => {
  try {
    const response: AxiosResponse<ApiResponse<T>> = await apiClient.get(
      url,
      config
    );
    return response.data;
  } catch (error) {
    throw handleApiError(error as AxiosError);
  }
};
```

**Usage**:
```typescript
// Type-safe GET request
const response = await apiGet<User[]>('/users');
// response.data is typed as User[]

// With config
const response = await apiGet<User[]>('/users', {
  params: { page: 1, limit: 10 },
  timeout: 5000,
});
```

### POST Request

```typescript
// Generic POST request helper
export const apiPost = async <T = any>(
  url: string,
  data?: any,
  config?: AxiosRequestConfig
): Promise<ApiResponse<T>> => {
  try {
    const response: AxiosResponse<ApiResponse<T>> = await apiClient.post(
      url,
      data,
      config
    );
    return response.data;
  } catch (error) {
    throw handleApiError(error as AxiosError);
  }
};
```

**Usage**:
```typescript
// Type-safe POST request
const response = await apiPost<LoginResponse>('/auth/login', {
  email: 'user@example.com',
  password: 'password123',
});
// response.data is typed as LoginResponse

// With extended timeout
const response = await apiPost<DeployResponse>('/deploy', data, {
  timeout: 30000, // 30 seconds for long operations
});
```

### Other HTTP Methods

```typescript
// PUT - Update entire resource
export const apiPut = async <T = any>(
  url: string,
  data?: any,
  config?: AxiosRequestConfig
): Promise<ApiResponse<T>> => { /* ... */ };

// PATCH - Partial update
export const apiPatch = async <T = any>(
  url: string,
  data?: any,
  config?: AxiosRequestConfig
): Promise<ApiResponse<T>> => { /* ... */ };

// DELETE - Remove resource
export const apiDelete = async <T = any>(
  url: string,
  config?: AxiosRequestConfig
): Promise<ApiResponse<T>> => { /* ... */ };
```

### Standard Response Type

```typescript
// Generic API response type
export interface ApiResponse<T = any> {
  data: T;           // Actual data
  message?: string;  // Optional message from server
  success: boolean;  // Success flag
}
```

**Example Response**:
```json
{
  "data": {
    "id": "123",
    "name": "John Doe"
  },
  "message": "User fetched successfully",
  "success": true
}
```

---

## Service Layer Pattern

The service layer encapsulates API logic and provides a clean interface.

### Pattern: Singleton Service Class

```typescript
export class AuthService {
  private static instance: AuthService;

  private constructor() {}

  // Singleton pattern
  public static getInstance(): AuthService {
    if (!AuthService.instance) {
      AuthService.instance = new AuthService();
    }
    return AuthService.instance;
  }

  async login(credentials: LoginRequest): Promise<LoginResponse> {
    try {
      const response = await apiPost<LoginResponse>("auth/login", credentials);
      // Validate response with Zod schema
      const validatedResponse = loginResponseSchema.parse(response);
      return validatedResponse;
    } catch (error) {
      const apiError = error as ApiError;

      // Transform errors to user-friendly messages
      if (apiError.status === 401) {
        throw new Error("Invalid email or password");
      }
      if (apiError.status === 400) {
        throw new Error("Invalid request data");
      }
      if (apiError.status >= 500) {
        throw new Error("Server error. Please try again later.");
      }
      throw new Error(apiError.message || "Login failed");
    }
  }

  async getCurrentUser(): Promise<User> {
    try {
      const token = this.getStoredToken();
      if (!token) {
        throw new Error("No authentication token found");
      }

      const response = await apiGet<User>("auth/profile");
      // Validate with Zod
      return userSchema.parse(response);
    } catch (error) {
      console.error("Failed to get current user:", error);
      throw new Error("Failed to authenticate user");
    }
  }

  // Helper methods
  getStoredToken(): string | null {
    if (typeof window !== "undefined") {
      return localStorage.getItem("accessToken");
    }
    return null;
  }

  storeToken(token: string): void {
    if (typeof window !== "undefined") {
      localStorage.setItem("accessToken", token);
    }
  }

  removeToken(): void {
    if (typeof window !== "undefined") {
      localStorage.removeItem("accessToken");
    }
  }
}

// Export singleton instance
export const authService = AuthService.getInstance();
```

**Key Learning Points**:
1. **Singleton Pattern**: One instance for the entire app
2. **Encapsulation**: API logic hidden from consumers
3. **Error Transformation**: Convert API errors to user-friendly messages
4. **Validation**: Use Zod to validate API responses
5. **Helper Methods**: Token management included

### Pattern: Exported Functions

```typescript
// Simple exported functions (not a class)
export const deployRollup = async (request: RollupDeploymentRequest) => {
  const response = await apiPost<{ id: string }>("stacks/thanos", request, {
    timeout: 30000, // Extended timeout for deployment
  });
  return response;
};

export const getThanosStacks = async (): Promise<ThanosStack[]> => {
  const response = await apiGet<GetAllThanosStacksResponse>("stacks/thanos");
  return response.data.stacks;
};

export const getThanosStackById = async (id: string): Promise<ThanosStack> => {
  const response = await apiGet<GetThanosStackResponse>(`stacks/thanos/${id}`);
  return response.data.stack;
};

export const deleteRollup = async (id: string) => {
  const response = await apiDelete(`stacks/thanos/${id}`);
  return response;
};
```

**When to Use Each Pattern**:

| Pattern | Use When | Example |
|---------|----------|---------|
| **Class (Singleton)** | Complex logic, state management, many related methods | AuthService (token management) |
| **Exported Functions** | Simple API wrappers, stateless operations | RollupService (CRUD operations) |

---

## Error Handling

### Error Type Definition

```typescript
// Error response type
export interface ApiError {
  message: string;
  status: number;
  errors?: Record<string, string[]>; // Validation errors
}
```

### Error Handler Function

```typescript
// Helper function to handle API errors
export const handleApiError = (error: AxiosError): ApiError => {
  if (error.response) {
    // Server responded with error status (4xx, 5xx)
    return {
      message: (error.response.data as any)?.message || "An error occurred",
      status: error.response.status,
      errors: (error.response.data as any)?.errors,
    };
  } else if (error.request) {
    // Request was made but no response received (network error)
    return {
      message: "No response from server",
      status: 0,
    };
  } else {
    // Something else happened (setup error)
    return {
      message: error.message || "An unexpected error occurred",
      status: 0,
    };
  }
};
```

**Error Categories**:

1. **Response Error** (status 4xx, 5xx)
   - Server responded with error
   - Extract message and status from response

2. **Network Error** (no response)
   - Request sent but no response
   - Network issues, timeout, server down

3. **Request Setup Error**
   - Error before request sent
   - Invalid config, etc.

### Service-Level Error Handling

```typescript
async login(credentials: LoginRequest): Promise<LoginResponse> {
  try {
    const response = await apiPost<LoginResponse>("auth/login", credentials);
    return loginResponseSchema.parse(response);
  } catch (error) {
    const apiError = error as ApiError;

    // Transform error codes to user-friendly messages
    if (apiError.status === 401) {
      throw new Error("Invalid email or password");
    }
    if (apiError.status === 400) {
      throw new Error("Invalid request data");
    }
    if (apiError.status >= 500) {
      throw new Error("Server error. Please try again later.");
    }
    throw new Error(apiError.message || "Login failed");
  }
}
```

**Error Flow**:

```
API Call
  ↓
apiPost helper catches AxiosError
  ↓
handleApiError transforms to ApiError
  ↓
Service catches ApiError
  ↓
Transforms to user-friendly Error
  ↓
React Query/Component handles Error
  ↓
Display to user
```

---

## Type Safety

### Generic Type Parameters

```typescript
// Specify response type
const users = await apiGet<User[]>('/users');
// users.data is User[]

const user = await apiPost<User>('/users', newUser);
// user.data is User

const deleted = await apiDelete<void>('/users/123');
// deleted.data is void
```

### Response Type Structure

```typescript
interface ApiResponse<T> {
  data: T;           // The actual typed data
  message?: string;
  success: boolean;
}

// Example usage
const response = await apiGet<User>('/users/123');
response.data.email  // TypeScript knows this is a string
response.data.age    // TypeScript knows this is a number
```

### Service Return Types

```typescript
// Clear return types in service
export const getThanosStacks = async (): Promise<ThanosStack[]> => {
  const response = await apiGet<GetAllThanosStacksResponse>("stacks/thanos");
  return response.data.stacks; // Extract and return typed data
};

// Usage
const stacks = await getThanosStacks();
// stacks is ThanosStack[]
```

---

## Architecture Layers Recap

```typescript
// 1. Component Layer
function UserList() {
  const { data: users } = useQuery({
    queryKey: ['users'],
    queryFn: getUsers, // Calls service
  });
}

// 2. Service Layer
export const getUsers = async (): Promise<User[]> => {
  const response = await apiGet<UsersResponse>('/users'); // Uses helper
  return response.data.users;
};

// 3. API Helper Layer
export const apiGet = async <T>(url: string) => {
  const response = await apiClient.get(url); // Uses instance
  return response.data;
};

// 4. Axios Instance Layer
const apiClient = axios.create({
  baseURL: API_BASE_URL,
  timeout: 10000,
});

// With interceptors
apiClient.interceptors.request.use(addAuthToken);
apiClient.interceptors.response.use(handleSuccess, handleError);
```
