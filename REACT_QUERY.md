# React Query

A comprehensive guide to understanding how this project uses React Query (TanStack Query) for server state management.

## What is React Query?

**React Query** (now **TanStack Query**) is a powerful data-fetching and state management library for React applications. It manages server state efficiently, handling:

- **Data fetching**: Automatic background refetching
- **Caching**: Smart caching strategies
- **Synchronization**: Keeps UI in sync with server data
- **Loading states**: Built-in loading and error states
- **Mutations**: Optimistic updates and cache invalidation

### Why Use React Query?

**Without React Query**:
```typescript
// ❌ Manual state management - lots of boilerplate
const [data, setData] = useState([]);
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState(null);

useEffect(() => {
  setIsLoading(true);
  fetch('/api/items')
    .then(res => res.json())
    .then(data => setData(data))
    .catch(err => setError(err))
    .finally(() => setIsLoading(false));
}, []);
```

**With React Query**:
```typescript
// ✅ Clean and declarative
const { data, isLoading, error } = useQuery({
  queryKey: ['items'],
  queryFn: () => fetch('/api/items').then(res => res.json()),
});
```

### Key Concepts

- **Query**: Fetching data (GET requests)
- **Mutation**: Modifying data (POST, PUT, DELETE)
- **Query Key**: Unique identifier for cached data
- **Query Function**: Function that returns a Promise
- **Cache**: Stores query results for reuse

---

## Setup & Configuration

### Global Configuration

**File**: [src/providers/query-provider.tsx](../src/providers/query-provider.tsx)

```typescript
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,         // Data is fresh for 1 minute
      gcTime: 10 * 60 * 1000,       // Cache persists for 10 minutes
      retry: 1,                     // Retry failed requests once
      refetchOnWindowFocus: false,  // Don't refetch on window focus
    },
  },
});

export function QueryProvider({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

---

## Query Patterns

Queries are used for **fetching** data (read operations).

### Pattern 1: Basic Query Hook

```typescript
import { useQuery } from "@tanstack/react-query";
import { apiKeysService } from "../services/apiKeysService";

const API_KEYS_QUERY_KEY = "api-keys";

export function useApiKeys() {
  const {
    data: apiKeys = [],
    isLoading,
    error,
    refetch,
  } = useQuery({
    queryKey: [API_KEYS_QUERY_KEY],
    queryFn: () => apiKeysService.getApiKeys(),
    staleTime: 5 * 60 * 1000,  // Override global: 5 minutes
    gcTime: 10 * 60 * 1000,
  });

  return {
    apiKeys,
    isLoading,
    error,
    refetch,
  };
}
```

**Usage**:
```typescript
function APIKeysList() {
  const { apiKeys, isLoading, error } = useApiKeys();

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <ul>
      {apiKeys.map(key => <li key={key.id}>{key.name}</li>)}
    </ul>
  );
}
```

### Pattern 2: Query Keys Factory

```typescript
export const rollupKeys = {
  all: ["rollups"] as const,
  thanosStacks: ["thanosStacks"] as const,
  thanosStack: (id: string) => [...rollupKeys.thanosStacks, id] as const,
  thanosDeployments: (stackId: string) =>
    [...rollupKeys.thanosStack(stackId), "deployments"] as const,
  integrations: (stackId: string) =>
    [...rollupKeys.thanosStack(stackId), "integrations"] as const,
  registerMetadataDAO: (stackId: string) =>
    [...rollupKeys.thanosStack(stackId), "register-metadata-dao"] as const,
  lists: () => [...rollupKeys.all, "list"] as const,
  list: (filters: string) => [...rollupKeys.lists(), { filters }] as const,
  details: () => [...rollupKeys.all, "detail"] as const,
  detail: (id: string) => [...rollupKeys.details(), id] as const,
} as const;
```

**Why Use Query Key Factories?**

1. **Consistency**: Centralized key definitions
2. **Hierarchical Structure**: Related queries share key prefixes
3. **Easy Invalidation**: Invalidate all related queries easily
4. **Type Safety**: `as const` provides better type inference

**Hierarchy Example**:
```
["thanosStacks"]                              ← All stacks
["thanosStacks", "123"]                       ← Specific stack
["thanosStacks", "123", "deployments"]        ← Stack's deployments
["thanosStacks", "123", "integrations"]       ← Stack's integrations
```

**Benefits**:
- Invalidate all stack data: `queryClient.invalidateQueries({ queryKey: rollupKeys.thanosStacks })`
- Invalidate specific stack: `queryClient.invalidateQueries({ queryKey: rollupKeys.thanosStack("123") })`

### Pattern 3: Dynamic Queries

```typescript
export const useRollup = (id: string) => {
  return useQuery({
    queryKey: rollupKeys.detail(id),
    queryFn: () => getRollupById(id),
    refetchInterval: 120000, // Auto-refetch every 2 minutes
  });
};
```

**Key Learning Points**:
- **Dynamic Query Key**: `rollupKeys.detail(id)` includes the ID
- **Automatic Refetching**: `refetchInterval` for real-time data
- Each unique ID creates a separate cache entry

### Pattern 4: Conditional Queries

```typescript
export const useThanosDeploymentsQuery = (id?: string) => {
  return useQuery({
    queryKey: id
      ? rollupKeys.thanosDeployments(id)
      : (["thanosStacks", "deployments", "disabled"] as const),
    queryFn: () => getThanosDeployments(id as string),
    enabled: Boolean(id),  // Only run query if ID exists
    refetchInterval: 10000,
  });
};
```

**Key Learning Points**:
- **enabled**: Query only runs when `id` is truthy
- **Disabled Query Key**: Use descriptive key when disabled
- **Type Assertion**: `id as string` is safe because `enabled` guards it

**Usage**:
```typescript
// Won't fetch until rollupId is available
const { data } = useThanosDeploymentsQuery(rollupId);
```

### Pattern 5: Queries with Options

```typescript
export const useThanosDeploymentLogsQuery = (
  stackId?: string,
  deploymentId?: string,
  options?: {
    limit?: number;
    afterId?: string;
    refetchIntervalMs?: number | false;
  }
) => {
  return useQuery({
    queryKey:
      stackId && deploymentId
        ? [
            ...rollupKeys.thanosDeployments(stackId),
            deploymentId,
            "logs",
            options?.limit ?? 200,
            options?.afterId,
          ]
        : (["thanosStacks", "deployments", "logs", "disabled"] as const),
    queryFn: () =>
      getThanosDeploymentLogs(stackId as string, deploymentId as string, {
        limit: options?.limit,
        afterId: options?.afterId,
      }),
    enabled: Boolean(stackId && deploymentId),
    refetchInterval: options?.refetchIntervalMs ?? 5000,
  });
};
```

**Key Learning Points**:
- **Query Key Includes Options**: Cache is separate for different options
- **Flexible Refetch**: Can override or disable refetch interval
- **Default Values**: `options?.limit ?? 200`

---

## Mutation Patterns

Mutations are used for **modifying** data (create, update, delete).

### Pattern 1: Basic Mutation with Optimistic Update

```typescript
import { useMutation, useQueryClient } from "@tanstack/react-query";
import toast from "react-hot-toast";

export function useCreateApiKey() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: APIKeyFormData) => apiKeysService.createApiKey(data),
    onSuccess: (newApiKey) => {
      // Optimistic update: Immediately update cache
      queryClient.setQueryData<APIKey[]>([API_KEYS_QUERY_KEY], (oldData) => {
        return oldData ? [...oldData, newApiKey] : [newApiKey];
      });
      toast.success("API key created successfully");
    },
    onError: (error: Error) => {
      toast.error(error.message || "Failed to create API key");
    },
  });
}
```

**Key Learning Points**:
1. **useQueryClient**: Access to cache management
2. **mutationFn**: Function that performs the mutation
3. **onSuccess**: Update cache immediately after success
4. **setQueryData**: Manually update cache with new data
5. **User Feedback**: Toast notifications for success/error

**Usage**:
```typescript
function CreateAPIKeyButton() {
  const createMutation = useCreateApiKey();

  const handleCreate = () => {
    createMutation.mutate({ name: "My API Key" });
  };

  return (
    <button
      onClick={handleCreate}
      disabled={createMutation.isPending}
    >
      {createMutation.isPending ? "Creating..." : "Create API Key"}
    </button>
  );
}
```

### Pattern 2: Update Mutation

```typescript
export function useUpdateApiKey() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<APIKeyFormData> }) =>
      apiKeysService.updateApiKey(id, data),
    onSuccess: (updatedApiKey) => {
      queryClient.setQueryData<APIKey[]>([API_KEYS_QUERY_KEY], (oldData) => {
        return oldData
          ? oldData.map((apiKey) =>
              apiKey.id === updatedApiKey.id ? updatedApiKey : apiKey
            )
          : [updatedApiKey];
      });
      toast.success("API key updated successfully");
    },
    onError: (error: Error) => {
      toast.error(error.message || "Failed to update API key");
    },
  });
}
```

**Key Learning Points**:
- **Mutation Args**: Pass both ID and data
- **Cache Update**: Map over array to replace updated item
- **Immutability**: Creates new array, doesn't mutate

### Pattern 3: Delete Mutation

```typescript
export function useDeleteApiKey() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (id: string) => apiKeysService.deleteApiKey(id),
    onSuccess: (_, deletedId) => {
      queryClient.setQueryData<APIKey[]>([API_KEYS_QUERY_KEY], (oldData) => {
        return oldData
          ? oldData.filter((apiKey) => apiKey.id !== deletedId)
          : [];
      });
      toast.success("API key deleted successfully");
    },
    onError: (error: Error) => {
      toast.error(error.message || "Failed to delete API key");
    },
  });
}
```

**Key Learning Points**:
- **Second Parameter**: `onSuccess: (data, variables)` - `variables` is the ID passed to `mutate()`
- **Filter**: Remove deleted item from cache

### Pattern 4: Mutation with Loading States

```typescript
export const useDeployRollupMutation = (options?: {
  onSuccess?: () => void;
  onError?: (error: Error) => void;
}) => {
  const router = useRouter();

  return useMutation({
    mutationFn: deployRollup,
    onMutate: () => {
      toast.loading("Deploying your rollup...", {
        id: "deploy-rollup",
      });
    },
    onSuccess: () => {
      toast.success("Rollup deployment initiated successfully!", {
        id: "deploy-rollup",
      });
      invalidateThanosStacks();
      router.push("/rollup");
      options?.onSuccess?.();
    },
    onError: (error: Error) => {
      toast.error(error.message || "Failed to deploy rollup", {
        id: "deploy-rollup",
      });
      options?.onError?.(error);
    },
  });
};
```

**Key Learning Points**:
1. **onMutate**: Runs before mutation starts - show loading toast
2. **Toast ID**: Same ID updates the same toast (loading → success/error)
3. **Callback Options**: Allow consumers to add custom logic
4. **Navigation**: Redirect after success
5. **Cache Invalidation**: `invalidateThanosStacks()` refetches data

### Pattern 5: Mutation with Cache Invalidation

```typescript
export const useDeleteRollupMutation = (options?: {
  onSuccess?: () => void;
  onError?: (error: Error) => void;
}) => {
  return useMutation({
    mutationFn: deleteRollup,
    onMutate: () => {
      toast.loading("Destroying rollup...", {
        id: "delete-rollup",
      });
    },
    onSuccess: (_data, id) => {
      toast.success("Rollup destruction initiated successfully!", {
        id: "delete-rollup",
      });
      invalidateThanosStacks();
      if (id) {
        queryClient.invalidateQueries({ queryKey: rollupKeys.thanosStack(id) });
        queryClient.invalidateQueries({
          queryKey: rollupKeys.thanosDeployments(id),
        });
      }
      options?.onSuccess?.();
    },
    onError: (error: Error) => {
      toast.error(error.message || "Failed to destroy rollup", {
        id: "delete-rollup",
      });
      options?.onError?.(error);
    },
  });
};
```

**Key Learning Points**:
- **invalidateQueries**: Mark queries as stale to trigger refetch
- **Hierarchical Invalidation**: Invalidate both parent and related queries
- **Granular Control**: Only invalidate affected data
