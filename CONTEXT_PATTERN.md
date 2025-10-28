# Context Pattern

This document explains how to leverage the React Context API through `RollupCreationContext` and `AuthProvider`.

## What is React Context?

React Context is a way to provide data throughout the component tree. It allows you to share values between components without prop drilling (passing props through multiple levels).

### Core Components
- **Context Creation**: `createContext()`
- **Provider**: Component that provides Context values
- **Consumer/Hook**: Method to consume Context values (`useContext`)

---

## Context Implementation Pattern

Core structure of the Context pattern used in this project:

```typescript
// 1. Define Context Type
interface MyContextType {
  // state and functions
}

// 2. Create Context
const MyContext = createContext<MyContextType | undefined>(undefined);

// 3. Implement Provider Component
export function MyProvider({ children }: { children: ReactNode }) {
  // state management logic
  return (
    <MyContext.Provider value={/* ... */}>
      {children}
    </MyContext.Provider>
  );
}

// 4. Provide Custom Hook
export function useMyContext() {
  const context = useContext(MyContext);
  if (context === undefined) {
    throw new Error("useMyContext must be used within a MyProvider");
  }
  return context;
}
```

---

## RollupCreationContext Analysis

### Purpose
Manages multi-step form state for the Rollup creation process.

### 1. State Structure Definition

```typescript
interface RollupCreationState {
  formData: CreateRollupFormData | null;  // Form data
  currentStep: number;                     // Current step
  hasUnsavedChanges: boolean;              // Unsaved changes flag
}
```

### 2. Context Type Definition

```typescript
interface RollupCreationContextType {
  state: RollupCreationState;
  updateFormData: (data: CreateRollupFormData) => void;
  updateCurrentStep: (step: number) => void;
  setHasUnsavedChanges: (hasChanges: boolean) => void;
  resetState: () => void;
}
```

### 3. Default Value Setup

```typescript
const defaultFormData: CreateRollupFormData = {
  networkAndChain: {
    network: "",
    chainName: "",
    l1RpcUrl: "",
    // ... other fields
  },
  accountAndAws: {
    seedPhrase: Array(12).fill(""),
    adminAccount: "",
    // ... other fields
  },
  daoCandidate: undefined,
};

const initialState: RollupCreationState = {
  formData: null,
  currentStep: 1,
  hasUnsavedChanges: false,
};
```

### 4. Provider Implementation

```typescript
export function RollupCreationProvider({ children }: { children: ReactNode }) {
  const [state, setState] = useState<RollupCreationState>(initialState);

  const updateFormData = (data: CreateRollupFormData) => {
    setState(prev => ({
      ...prev,
      formData: data,
      hasUnsavedChanges: true,  // Automatically set unsaved flag
    }));
  };

  const updateCurrentStep = (step: number) => {
    setState(prev => ({
      ...prev,
      currentStep: step,
    }));
  };

  const setHasUnsavedChanges = (hasChanges: boolean) => {
    setState(prev => ({
      ...prev,
      hasUnsavedChanges: hasChanges,
    }));
  };

  const resetState = () => {
    setState(initialState);
  };

  return (
    <RollupCreationContext.Provider
      value={{
        state,
        updateFormData,
        updateCurrentStep,
        setHasUnsavedChanges,
        resetState,
      }}
    >
      {children}
    </RollupCreationContext.Provider>
  );
}
```

### 5. Custom Hook

```typescript
export function useRollupCreationContext() {
  const context = useContext(RollupCreationContext);
  if (context === undefined) {
    throw new Error("useRollupCreationContext must be used within a RollupCreationProvider");
  }
  return context;
}
```

### Usage Example

```typescript
function MyComponent() {
  const { state, updateFormData, updateCurrentStep } = useRollupCreationContext();

  const handleNext = () => {
    updateCurrentStep(state.currentStep + 1);
  };

  const handleFormChange = (newData: CreateRollupFormData) => {
    updateFormData(newData);
  };

  return (
    <div>
      <p>Current Step: {state.currentStep}</p>
      {state.hasUnsavedChanges && <p>You have unsaved changes</p>}
    </div>
  );
}
```

---

## AuthProvider Analysis

### Purpose
Manages authentication state for the entire application.

### 1. Context Type Definition

```typescript
interface AuthContextType {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (credentials: { email: string; password: string }) => void;
  logout: () => void;
  isLoggingIn: boolean;
}
```

### 2. Context Creation

```typescript
const AuthContext = createContext<AuthContextType | undefined>(undefined);
```

### 3. Provider Implementation (Using Custom Hook)

```typescript
const AuthProviderContent: React.FC<AuthProviderProps> = ({ children }) => {
  const auth = useAuth();  // Delegates actual logic to separate Hook

  return <AuthContext.Provider value={auth}>{children}</AuthContext.Provider>;
};
```

### 4. Suspense Integration

```typescript
export const AuthProvider: React.FC<AuthProviderProps> = ({ children }) => {
  return (
    <Suspense
      fallback={
        <div className="flex items-center justify-center min-h-screen">
          <div className="animate-spin rounded-full h-32 w-32 border-b-2 border-gray-900"></div>
        </div>
      }
    >
      <AuthProviderContent>{children}</AuthProviderContent>
    </Suspense>
  );
};
```

### 5. Custom Hook

```typescript
export const useAuthContext = () => {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error("useAuthContext must be used within an AuthProvider");
  }
  return context;
};
```

### Usage Example

```typescript
function UserProfile() {
  const { user, logout, isAuthenticated } = useAuthContext();

  if (!isAuthenticated) {
    return <div>Please log in</div>;
  }

  return (
    <div>
      <h1>Welcome, {user?.name}</h1>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```
