# Project Structure

A comprehensive guide to understanding the TRH Platform UI project architecture and directory organization.

## High-Level Architecture

```
trh-platform-ui/
├── src/
│   ├── app/              # Next.js App Router (routes & pages)
│   ├── features/         # Feature modules (business logic)
│   ├── components/       # Shared components
│   ├── providers/        # Global context providers
│   └── lib/              # Utilities & configurations
├── docs/                 # Documentation
└── public/               # Static assets
```

---

## Directory Structure

### Complete Structure



---

## Core Directories Explained

### 1. `/app` - Next.js App Router

**Purpose**: Defines the application's routing structure and page components.

**Key Characteristics**:
- File-based routing (Next.js convention)
- Each folder represents a route segment
- Special files: `page.tsx`, `layout.tsx`, `not-found.tsx`
- Dynamic routes using `[param]` syntax

**Example**:
```
app/
├── rollup/
│   ├── page.tsx           → /rollup (list page)
│   ├── [id]/
│   │   └── page.tsx       → /rollup/123 (detail page)
│   └── create/
│       └── page.tsx       → /rollup/create (create page)
```


### 2. `/features` - Feature Modules

**Purpose**: Encapsulates business logic and feature-specific code.

**Key Characteristics**:
- **Self-contained**: Each feature is independent
- **Standardized structure**: components, hooks, services, schemas
- **Nested features**: Complex features can have sub-features
- **Public API**: Only exports what's needed via `index.ts`

**Typical Feature Structure**:
```
features/
└── rollup/
    ├── components/          # UI components for this feature
    │   ├── RollupList.tsx
    │   ├── RollupItem.tsx
    │   └── steps/           # Sub-grouping for related components
    ├── hooks/               # Custom hooks
    │   └── useRollups.ts
    ├── services/            # API calls & business logic
    │   └── rollupService.ts
    ├── context/             # Feature-specific contexts
    │   └── RollupCreationContext.tsx
    ├── schemas/             # Zod validation schemas
    │   └── rollupSchema.ts
    ├── utils/               # Feature-specific utilities
    ├── api/                 # API endpoint definitions
    └── index.ts             # Public exports
```

**Example - Configuration Feature**:

The `configuration` feature demonstrates nested sub-features:

```
features/configuration/
├── components/              # Shared components across all config
│   └── ConfigurationTabs.tsx
├── api-keys/                # Sub-feature: API key management
│   ├── components/
│   ├── hooks/
│   └── services/
├── aws-credentials/         # Sub-feature: AWS credentials
│   ├── components/
│   ├── hooks/
│   └── services/
├── rpc-management/          # Sub-feature: RPC URL management
│   ├── components/
│   ├── hooks/
│   └── services/
└── shared/                  # Shared within configuration only
    └── components/
```

### 3. `/components` - Shared Components

**Purpose**: Reusable components used across multiple features.

**Organization by Abstraction Level**:

```
components/
├── ui/              # Atomic components (lowest level)
├── molecules/       # Composite components (medium level)
├── layout/          # Layout components
├── auth/            # Auth-related shared components
└── icon/            # Icon components
```

#### 3.1 `/ui` - Base UI Components

**Characteristics**:
- Built with Radix UI primitives
- Styled with Tailwind CSS
- Highly reusable and generic
- No business logic

**Examples**: `Button`, `Input`, `Dialog`, `Card`, `Select`

#### 3.2 `/molecules` - Composite Components

**Characteristics**:
- Combine multiple UI components
- Implement common UI patterns
- Still generic enough to be reused
- May contain some logic

**Examples**:
- `Sidebar`: Navigation sidebar
- `ApiKeySelector`: Dropdown for selecting API keys
- `RPCSelector`: Dropdown for selecting RPC URLs
- `PasswordInput`: Input with show/hide toggle

#### 3.3 `/icon` - Icon Components

**Characteristics**:
- SVG icon components
- Consistent sizing and styling
- Used throughout the app

**Examples**: `LogoIcon`, `DashboardItemIcon`, `UserIcon`

#### 3.4 `/auth` - Auth Components

**Characteristics**:
- Authentication/authorization utilities
- Used across multiple routes
- Higher-order components and utilities

**Examples**:
- `ProtectedRoute`: Requires authentication
- `RequireRole`: Requires specific role
- `withAuth`: HOC for auth protection

### 4. `/providers` - Global Providers

**Purpose**: Application-wide context providers and configurations.

**Files**:
```
providers/
├── auth-provider.tsx       # Authentication state
├── query-provider.tsx      # TanStack Query configuration
├── toaster-provider.tsx    # Toast notifications
└── index.ts                # Exports all providers
```

**Usage in `app/layout.tsx`**:
```tsx
import { Providers } from '@/providers';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <Providers>
          {children}
        </Providers>
      </body>
    </html>
  );
}
```

**When to create a provider**:
- State needed across the entire app
- Configuration that wraps the app
- Third-party library setup (React Query, etc.)

### 5. `/lib` - Utilities & Configurations

**Purpose**: Shared utilities, helpers, and configurations.

**Files**:
```
lib/
├── api.ts          # Axios instance, interceptors, base config
└── utils.ts        # Utility functions (e.g., cn for classnames)
```

**Example - `api.ts`**:
```typescript
// Centralized API configuration
export const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  // ... interceptors, auth headers, etc.
});
```

**Example - `utils.ts`**:
```typescript
// Utility for merging Tailwind classes
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

---

## Feature-Based Organization

### Why Feature-Based?

Traditional layer-based architecture:
```
❌ Layer-Based (Hard to maintain)
src/
├── components/      # All components
├── hooks/           # All hooks
├── services/        # All services
└── utils/           # All utils
```

Problems:
- Files for one feature scattered across folders
- Hard to understand feature boundaries
- Difficult to extract or remove features
- Team conflicts when working on same folders

Feature-based architecture:
```
✅ Feature-Based (Maintainable)
src/
└── features/
    ├── auth/        # Everything auth-related
    ├── rollup/      # Everything rollup-related
    └── config/      # Everything config-related
```

Benefits:
- All code for a feature in one place
- Easy to understand feature scope
- Simple to extract as a package
- Clear team ownership

### Feature Module Pattern

Each feature follows a consistent structure:

```
features/my-feature/
├── components/           # Feature UI components
│   ├── MyFeatureList.tsx
│   ├── MyFeatureItem.tsx
│   └── MyFeatureForm.tsx
│
├── hooks/                # Feature hooks
│   ├── index.ts
│   └── useMyFeature.ts
│
├── services/             # API & business logic
│   └── myFeatureService.ts
│
├── context/              # Feature contexts (optional)
│   └── MyFeatureContext.tsx
│
├── schemas/              # Validation schemas (optional)
│   └── myFeatureSchema.ts
│
├── utils/                # Feature utilities (optional)
│   └── myFeatureUtils.ts
│
├── types.ts              # Feature types (optional)
├── constants.ts          # Feature constants (optional)
└── index.ts              # Public API
```
