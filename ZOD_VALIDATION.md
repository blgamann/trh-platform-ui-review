# Zod Validation Pattern Guide

## What is Zod?

**Zod** is a TypeScript-first schema validation library. It allows you to:
- Define data schemas (shape and rules)
- Validate data at runtime
- Automatically infer TypeScript types from schemas
- Provide user-friendly error messages

### The Problem Zod Solves

```typescript
// ❌ Without Zod: No runtime validation
interface User {
  email: string;
  age: number;
}

function registerUser(data: User) {
  // TypeScript says it's a User, but at runtime...
  // - email might be invalid
  // - age might be negative
  // - fields might be missing
}

// ✅ With Zod: Runtime validation + Type safety
import { z } from 'zod';

const userSchema = z.object({
  email: z.string().email("Invalid email"),
  age: z.number().min(0, "Age must be positive"),
});

type User = z.infer<typeof userSchema>;

function registerUser(data: unknown) {
  const validData = userSchema.parse(data); // Throws if invalid
  // Now validData is guaranteed to be correct!
}
```

**Key Point**: TypeScript validates at **compile time**, Zod validates at **runtime**.

---

## Why Use Zod?

### 1. Runtime Type Safety

```typescript
// User input from forms is always `unknown` at runtime
const formData = await request.json(); // unknown

// Zod validates and transforms to typed data
const validData = userSchema.parse(formData); // User
```

### 2. Single Source of Truth

```typescript
// ✅ Schema defines both validation AND types
const userSchema = z.object({
  email: z.string().email(),
  age: z.number().min(18),
});

type User = z.infer<typeof userSchema>; // Type from schema!

// ❌ Without Zod: Duplicate definitions
interface User {
  email: string;
  age: number;
}

function validateUser(data: any): data is User {
  return typeof data.email === 'string' &&
         typeof data.age === 'number' &&
         data.age >= 18;
}
```

### 3. Better Error Messages

```typescript
const result = userSchema.safeParse({
  email: "invalid-email",
  age: -5,
});

if (!result.success) {
  console.log(result.error.issues);
  // [
  //   { path: ['email'], message: 'Invalid email' },
  //   { path: ['age'], message: 'Number must be >= 18' }
  // ]
}
```

### 4. Seamless React Hook Form Integration

```typescript
import { zodResolver } from '@hookform/resolvers/zod';

const form = useForm({
  resolver: zodResolver(userSchema), // Automatic validation!
});
```

---

## Basic Schema Patterns

### 1. Primitive Types

```typescript
import { z } from 'zod';

// String validations
const nameSchema = z.string();
const emailSchema = z.string().email();
const urlSchema = z.string().url();
const minLengthSchema = z.string().min(3, "Too short");
const maxLengthSchema = z.string().max(10, "Too long");
const regexSchema = z.string().regex(/^[A-Z]+$/, "Must be uppercase");

// Number validations
const ageSchema = z.number();
const positiveSchema = z.number().positive();
const integerSchema = z.number().int();
const rangeSchema = z.number().min(0).max(100);

// Boolean
const agreeSchema = z.boolean();

// Date
const dateSchema = z.date();
const futureDate = z.date().min(new Date(), "Must be in the future");
```

### 2. Object Schemas

```typescript
// Simple object
export const loginRequestSchema = z.object({
  email: z.string().email({ message: "Invalid email address" }),
  password: z.string().min(1, { message: "Password is required" }),
});

// Nested object
export const userSchema = z.object({
  email: z.string().email(),
  id: z.string(),
  role: z.enum(["Admin", "User"]),
});

export const loginResponseSchema = z.object({
  token: z.string(),
  user: userSchema, // Nested schema
});
```

### 3. Array Schemas

```typescript
// Array of strings
const tagsSchema = z.array(z.string());

// Array with length constraints
const seedPhraseSchema = z
  .array(z.string())
  .length(12, "Seed phrase must contain exactly 12 words");

// Array with min/max
const itemsSchema = z.array(z.string()).min(1).max(10);

// Array of objects
const usersSchema = z.array(
  z.object({
    name: z.string(),
    email: z.string().email(),
  })
);
```

```typescript
export const accountAndAwsSchema = z.object({
  seedPhrase: z
    .array(z.string())
    .length(12, "Seed phrase must contain exactly 12 words"),
  // ... other fields
});
```

### 4. Optional and Nullable Fields

```typescript
// Optional: field can be undefined or the specified type
const schema = z.object({
  name: z.string(),
  nickname: z.string().optional(), // string | undefined
});

// Nullable: field can be null or the specified type
const schema2 = z.object({
  name: z.string(),
  deletedAt: z.date().nullable(), // Date | null
});

// Both optional and nullable
const schema3 = z.object({
  description: z.string().nullable().optional(), // string | null | undefined
});

// Default value
const schema4 = z.object({
  status: z.string().default("active"),
});
```

```typescript
export const networkAndChainSchema = z.object({
  network: z.string().min(1, "Network is required"),
  chainName: z.string().min(1, "Chain name is required"),
  // Optional fields
  l2BlockTime: z.string().optional(),
  batchSubmissionFreq: z.string().optional(),
  outputRootFreq: z.string().optional(),
  challengePeriod: z.string().optional(),
});
```

### 5. Enum Schemas

```typescript
// String literal union
const roleSchema = z.enum(["Admin", "User", "Guest"]);

// Equivalent TypeScript type: "Admin" | "User" | "Guest"
type Role = z.infer<typeof roleSchema>;

// Native enum (not recommended, use z.enum instead)
enum Status {
  Active = "active",
  Inactive = "inactive",
}
const statusSchema = z.nativeEnum(Status);
```

```typescript
export const rpcUrlSchema = z.object({
  id: z.string(),
  name: z.string().min(1, "Name is required"),
  rpcUrl: z.string().url("Must be a valid URL"),
  type: z.enum(["ExecutionLayer", "BeaconChain"]),
  network: z.enum(["Mainnet", "Testnet"]),
  createdAt: z.string(),
  updatedAt: z.string(),
});
```

---

## Advanced Validation

### 1. Custom Validation with `.refine()`

`.refine()` allows custom validation logic:

```typescript
const schema = z.object({
  password: z.string(),
  confirmPassword: z.string(),
}).refine(
  (data) => data.password === data.confirmPassword,
  {
    message: "Passwords don't match",
    path: ["confirmPassword"], // Which field to show error on
  }
);
```

```typescript
export const networkAndChainSchema = z
  .object({
    // ... fields
    l2BlockTime: z.string().optional(),
    outputRootFreq: z.string().optional(),
  })
  .refine(
    (data) => {
      // Complex validation: outputRootFreq must be multiple of l2BlockTime
      if (data.outputRootFreq && data.l2BlockTime) {
        const outputRootFreq = Number(data.outputRootFreq);
        const l2BlockTime = Number(data.l2BlockTime);
        if (isNaN(outputRootFreq) || isNaN(l2BlockTime)) return false;
        if (outputRootFreq <= 0 || l2BlockTime <= 0) return false;
        if (!Number.isInteger(outputRootFreq) || !Number.isInteger(l2BlockTime))
          return false;
        return outputRootFreq % l2BlockTime === 0; // Must be divisible
      }
      return true;
    },
    {
      message: "Output Root Frequency must be a multiple of L2 Block Time",
      path: ["outputRootFreq"], // Show error on this field
    }
  );
```

**Key Learning Points**:
- Validation function: `(data) => boolean`
- Return `true` if valid, `false` if invalid
- `path` determines where error is shown
- Can access multiple fields for cross-field validation

### 2. Multiple Refinements

You can chain multiple `.refine()` calls:

```typescript
const schema = z
  .object({
    l2BlockTime: z.string().optional(),
  })
  .refine(
    (data) => {
      if (data.l2BlockTime) {
        const num = Number(data.l2BlockTime);
        return !isNaN(num) && num > 0 && Number.isInteger(num);
      }
      return true;
    },
    {
      message: "L2 Block Time must be a positive integer",
      path: ["l2BlockTime"],
    }
  )
  .refine(
    (data) => {
      // Another validation...
    },
    {
      message: "Another validation rule",
      path: ["l2BlockTime"],
    }
  );
```

```typescript
export const networkAndChainSchema = z
  .object({ /* ... */ })
  .refine(/* validation 1 */)
  .refine(/* validation 2 */)
  .refine(/* validation 3 */)
  .refine(/* validation 4 */);
```

### 3. Transform Data

`.transform()` modifies validated data:

```typescript
// Convert string to number
const ageSchema = z.string().transform((val) => parseInt(val, 10));

// Trim whitespace
const nameSchema = z.string().transform((val) => val.trim());

// Complex transformation
const userSchema = z.object({
  email: z.string().email().transform((val) => val.toLowerCase()),
  createdAt: z.string().transform((val) => new Date(val)),
});
```

### 4. Conditional Schemas

```typescript
// Different validation based on a field
const formSchema = z.object({
  type: z.enum(["email", "phone"]),
  contact: z.string(),
}).refine(
  (data) => {
    if (data.type === "email") {
      return z.string().email().safeParse(data.contact).success;
    }
    if (data.type === "phone") {
      return /^\d{10}$/.test(data.contact);
    }
    return true;
  },
  {
    message: "Invalid contact format",
    path: ["contact"],
  }
);
```

---

## Integration with React Hook Form

This is one of the most powerful patterns in the project.

### Basic Integration

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

type LoginForm = z.infer<typeof loginSchema>;

function LoginForm() {
  const form = useForm<LoginForm>({
    resolver: zodResolver(loginSchema), // Zod handles all validation!
    defaultValues: {
      email: '',
      password: '',
    },
  });

  const onSubmit = (data: LoginForm) => {
    // data is fully validated and typed!
    console.log(data);
  };

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      <input {...form.register('email')} />
      {form.formState.errors.email && (
        <p>{form.formState.errors.email.message}</p>
      )}

      <input {...form.register('password')} type="password" />
      {form.formState.errors.password && (
        <p>{form.formState.errors.password.message}</p>
      )}

      <button type="submit">Login</button>
    </form>
  );
}
```

**Key Benefits**:
- Automatic validation on submit
- Error messages from Zod schema
- Type-safe form data
- No manual validation code

### Real Example: Multi-Step Form

```typescript
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { createRollupSchema, CreateRollupFormData } from "../schemas/create-rollup";

export function useCreateRollup() {
  const form = useForm<CreateRollupFormData>({
    resolver: zodResolver(createRollupSchema),
    defaultValues: state.formData || defaultFormData,
  });

  // Validate specific step fields
  const goToNextStep = async () => {
    let isValid = false;

    switch (currentStep) {
      case 1: // Network & Chain step
        isValid = await form.trigger([
          "networkAndChain.network",
          "networkAndChain.chainName",
          "networkAndChain.l1RpcUrl",
          "networkAndChain.l1BeaconUrl",
        ]);
        break;
      case 2: // Account & AWS step
        isValid = await form.trigger([
          "accountAndAws.seedPhrase",
          "accountAndAws.adminAccount",
          // ... all fields
        ]);
        break;
      // ...
    }

    if (isValid) {
      updateCurrentStep(currentStep + 1);
    }
  };
}
```

**Key Learning Points**:
- `form.trigger()` validates specific fields
- Can validate subset of schema for multi-step forms
- Zod schema handles all complex validation rules

---

## Schema Composition

### 1. Reusing Schemas

```typescript
// Base schemas
const addressSchema = z.object({
  street: z.string(),
  city: z.string(),
  zipCode: z.string(),
});

const contactSchema = z.object({
  email: z.string().email(),
  phone: z.string(),
});

// Compose into larger schema
const userSchema = z.object({
  name: z.string(),
  address: addressSchema,    // Reuse
  contact: contactSchema,    // Reuse
});
```

```typescript
// Individual step schemas
export const networkAndChainSchema = z.object({ /* ... */ });
export const accountAndAwsSchema = z.object({ /* ... */ });
export const daoCandidateSchema = z.object({ /* ... */ }).optional();

// Combined schema for entire form
export const createRollupSchema = z.object({
  networkAndChain: networkAndChainSchema,
  accountAndAws: accountAndAwsSchema,
  daoCandidate: daoCandidateSchema,
});
```

**Benefits**:
- Modular, reusable schemas
- Each step has its own schema
- Easy to validate individual sections

### 2. Extending Schemas

```typescript
// Base schema
const baseUserSchema = z.object({
  name: z.string(),
  email: z.string().email(),
});

// Extend with additional fields
const adminUserSchema = baseUserSchema.extend({
  role: z.literal("admin"),
  permissions: z.array(z.string()),
});

// Partial schema (all fields optional)
const updateUserSchema = baseUserSchema.partial();

// Pick specific fields
const userNameOnlySchema = baseUserSchema.pick({ name: true });

// Omit specific fields
const userWithoutEmailSchema = baseUserSchema.omit({ email: true });
```

```typescript
// Full schema
export const awsCredentialSchema = z.object({
  id: z.string(),
  name: z.string().min(1, "Name is required"),
  accessKeyId: z.string().min(1, "Access Key ID is required"),
  secretAccessKey: z.string().min(1, "Secret Access Key is required"),
  createdAt: z.string(),
  updatedAt: z.string().optional(),
});

// Create schema (omit id, timestamps)
export const createAwsCredentialSchema = z.object({
  name: z.string().min(1, "Name is required"),
  accessKeyId: z.string().min(1, "Access Key ID is required"),
  secretAccessKey: z.string().min(1, "Secret Access Key is required"),
});

// Update schema (all fields optional)
export const updateAwsCredentialSchema = z.object({
  name: z.string().min(1, "Name is required").optional(),
  accessKeyId: z.string().min(1, "Access Key ID is required").optional(),
  secretAccessKey: z
    .string()
    .min(1, "Secret Access Key is required")
    .optional(),
});
```

---

## Type Inference

One of Zod's killer features: **automatic TypeScript type generation**.

### Basic Type Inference

```typescript
const userSchema = z.object({
  name: z.string(),
  age: z.number(),
  email: z.string().email(),
});

// Infer TypeScript type from schema
type User = z.infer<typeof userSchema>;

// Equivalent to:
// type User = {
//   name: string;
//   age: number;
//   email: string;
// }
```

### Complex Type Inference

```typescript
// Define schemas
export const awsCredentialSchema = z.object({
  id: z.string(),
  name: z.string().min(1, "Name is required"),
  accessKeyId: z.string().min(1, "Access Key ID is required"),
  secretAccessKey: z.string().min(1, "Secret Access Key is required"),
  createdAt: z.string(),
  updatedAt: z.string().optional(),
});

export const createAwsCredentialSchema = z.object({
  name: z.string().min(1, "Name is required"),
  accessKeyId: z.string().min(1, "Access Key ID is required"),
  secretAccessKey: z.string().min(1, "Secret Access Key is required"),
});

// Infer all types
export type AWSCredential = z.infer<typeof awsCredentialSchema>;
export type CreateAWSCredentialRequest = z.infer<typeof createAwsCredentialSchema>;
export type UpdateAWSCredentialRequest = z.infer<typeof updateAwsCredentialSchema>;
```

**Benefits**:
- Single source of truth (schema)
- Types automatically stay in sync with validation
- No duplicate type definitions

### Input vs Output Types

```typescript
const schema = z.object({
  name: z.string(),
  age: z.string().transform((val) => parseInt(val, 10)),
});

// Input type (before transformation)
type Input = z.input<typeof schema>;
// { name: string, age: string }

// Output type (after transformation)
type Output = z.output<typeof schema>;
// { name: string, age: number }

// Default z.infer gives output type
type Default = z.infer<typeof schema>;
// { name: string, age: number }
```
