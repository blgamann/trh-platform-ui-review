# React Hook Form Pattern Guide

## What is React Hook Form?

**React Hook Form** is a performant, flexible, and extensible form library for React. It:
- Minimizes re-renders
- Reduces boilerplate code
- Provides built-in validation
- Integrates seamlessly with validation libraries (Zod, Yup, etc.)
- Supports complex form scenarios

### The Problem It Solves

```typescript
// ❌ Without React Hook Form: Manual state management
function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleEmailChange = (e) => {
    setEmail(e.target.value);
    // Validate email...
  };

  const handlePasswordChange = (e) => {
    setPassword(e.target.value);
    // Validate password...
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    setIsSubmitting(true);
    // Validate all fields...
    // Submit...
    setIsSubmitting(false);
  };

  // Lots of boilerplate! 😓
}

// ✅ With React Hook Form: Clean and declarative
function LoginForm() {
  const form = useForm({
    resolver: zodResolver(loginSchema),
  });

  const onSubmit = (data) => {
    // data is already validated!
    console.log(data);
  };

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      <input {...form.register('email')} />
      <input {...form.register('password')} />
      <button type="submit">Login</button>
    </form>
  );
}
```

---

## Why Use React Hook Form?

### 1. **Performance**

React Hook Form uses **uncontrolled components** and minimizes re-renders.

```typescript
// Traditional controlled input: Re-renders on every keystroke
const [value, setValue] = useState('');
<input value={value} onChange={e => setValue(e.target.value)} />

// React Hook Form: No re-renders until validation/submit
<input {...register('field')} />
```

**Benchmark**: React Hook Form is ~30-40% faster than Formik for large forms.

### 2. **Less Code**

```typescript
// Without RHF: ~50 lines
// With RHF: ~15 lines (70% reduction!)
```

### 3. **Built-in Validation**

```typescript
// Validation rules directly in register
<input {...register('email', {
  required: 'Email is required',
  pattern: {
    value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
    message: 'Invalid email',
  },
})} />
```

### 4. **Seamless Integration**

Works perfectly with:
- **Zod** - Schema validation (used in this project)
- **Yup** - Another schema validator
- **UI Libraries** - Radix UI, Material-UI, etc.

---

## Basic Setup

### 1. Initialize Form

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// Define schema
const authFormSchema = z.object({
  email: z.string().email("Invalid email address"),
  password: z.string().min(1, "Password is required"),
  rememberMe: z.boolean(),
});

type AuthFormData = z.infer<typeof authFormSchema>;

function AuthForm() {
  const form = useForm<AuthFormData>({
    resolver: zodResolver(authFormSchema),
    defaultValues: {
      email: "",
      password: "",
      rememberMe: false,
    },
  });

  const onSubmit = (data: AuthFormData) => {
    console.log(data); // Fully validated data!
  };

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* Form fields */}
    </form>
  );
}
```

**Key Components**:
- `useForm()` - Main hook to manage form
- `resolver` - Validation strategy (Zod in this case)
- `defaultValues` - Initial form values
- `form.handleSubmit()` - Wrapper for submit handler

### 2. Form Object Properties

```typescript
const form = useForm();

// Available properties:
form.register      // Register input fields
form.handleSubmit  // Handle form submission
form.formState     // Form state (errors, isDirty, isSubmitting, etc.)
form.watch         // Watch field values
form.setValue      // Set field value programmatically
form.trigger       // Trigger validation
form.reset         // Reset form
form.getValues     // Get current values
form.setError      // Set errors manually
```

---

## Form Registration Patterns

### Pattern 1: Basic Registration

The simplest way to connect an input to the form:

```typescript
<input {...register('email')} />
```

**What `register` returns**:
```typescript
{
  name: 'email',
  onChange: (e) => { /* update form state */ },
  onBlur: (e) => { /* trigger validation */ },
  ref: (el) => { /* register element */ }
}
```

The spread operator `{...register('email')}` applies all these props to the input.

```typescript
<Input
  id="email"
  type="email"
  placeholder="Enter your email or username"
  {...form.register("email")}
  disabled={isLoading}
  className={
    form.formState.errors.email ? "border-destructive" : ""
  }
/>
```

### Pattern 2: Registration with Validation

```typescript
<input
  {...register('age', {
    required: 'Age is required',
    min: { value: 18, message: 'Must be 18+' },
    max: { value: 100, message: 'Invalid age' },
  })}
/>
```

**Note**: In this project, we use **Zod** for validation instead of inline rules!

### Pattern 3: Controlled Components

For UI libraries that require controlled components:

```typescript
import { Controller } from 'react-hook-form';

<Controller
  name="country"
  control={form.control}
  render={({ field }) => (
    <Select
      value={field.value}
      onValueChange={field.onChange}
    >
      <SelectItem value="us">USA</SelectItem>
      <SelectItem value="uk">UK</SelectItem>
    </Select>
  )}
/>
```

### Pattern 4: Checkbox/Boolean Fields

```typescript
<Checkbox
  id="remember-me"
  {...form.register("rememberMe")}
  disabled={isLoading}
/>
<label htmlFor="remember-me">Remember me</label>
```

---

## Validation Integration

### With Zod (Recommended)

This project uses **Zod** for all validation.

```typescript
import { z } from 'zod';
import { zodResolver } from '@hookform/resolvers/zod';

// 1. Define Zod schema
const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

// 2. Infer TypeScript type
type FormData = z.infer<typeof schema>;

// 3. Use with React Hook Form
const form = useForm<FormData>({
  resolver: zodResolver(schema),
});
```

**Benefits**:
- Single source of truth
- Type safety
- Complex validation rules
- Great error messages

---

## Advanced Patterns

### Pattern 1: `useFormContext` for Nested Components

For multi-step or complex forms, avoid prop drilling with `useFormContext`.

```typescript
import { FormProvider, useForm } from 'react-hook-form';

function CreateRollupForm() {
  const form = useForm<CreateRollupFormData>({
    resolver: zodResolver(createRollupSchema),
    defaultValues: defaultFormData,
  });

  return (
    <FormProvider {...form}>
      <NetworkAndChainStep />
      <AccountAndAwsStep />
    </FormProvider>
  );
}
```

```typescript
import { useFormContext } from 'react-hook-form';

export function NetworkAndChainStep() {
  const {
    setValue,
    watch,
    trigger,
    formState: { errors },
  } = useFormContext<CreateRollupFormData>();

  // Now you have access to the form!
  const network = watch('networkAndChain.network');
}
```

**Key Learning Points**:
- `FormProvider` wraps the form and provides context
- `useFormContext()` accesses the form in any nested component
- No need to pass form props down manually
- Great for multi-step forms

### Pattern 2: `watch` - Reactive Form Values

Watch field values to trigger side effects:

```typescript
// Watch single field
const email = watch('email');

// Watch multiple fields
const [email, password] = watch(['email', 'password']);

// Watch all fields
const allValues = watch();

// Watch with callback (for side effects)
useEffect(() => {
  const subscription = watch((data, { name, type }) => {
    console.log(`${name} changed:`, data);
  });
  return () => subscription.unsubscribe();
}, [watch]);
```

```typescript
const formData = watch(); // Watch all fields
const advancedConfig = watch("networkAndChain.advancedConfig");
const selectedNetwork = watch("networkAndChain.network");

// Use watched values to control UI
{advancedConfig && (
  <Card>
    <CardHeader>Advanced Settings</CardHeader>
    {/* Advanced fields */}
  </Card>
)}
```

### Pattern 3: `setValue` - Programmatic Updates

Set values programmatically (not through user input):

```typescript
// Set single value
setValue('email', 'user@example.com');

// Set with validation
setValue('email', 'user@example.com', {
  shouldValidate: true,  // Trigger validation
  shouldDirty: true,     // Mark as dirty
  shouldTouch: true,     // Mark as touched
});
```

```typescript
const handleAdvancedConfigChange = (checked: boolean) => {
  setValue("networkAndChain.advancedConfig", checked);

  if (checked) {
    // Set default values when showing advanced fields
    setValue("networkAndChain.l2BlockTime", "2");
    setValue("networkAndChain.batchSubmissionFreq", "1440");
    setValue("networkAndChain.outputRootFreq", "240");
    setValue("networkAndChain.challengePeriod", "12");

    // Validate the new fields
    trigger([
      "networkAndChain.l2BlockTime",
      "networkAndChain.batchSubmissionFreq",
      "networkAndChain.outputRootFreq",
      "networkAndChain.challengePeriod",
    ]);
  } else {
    // Clear values when hiding advanced fields
    setValue("networkAndChain.l2BlockTime", undefined);
    setValue("networkAndChain.batchSubmissionFreq", undefined);
    setValue("networkAndChain.outputRootFreq", undefined);
    setValue("networkAndChain.challengePeriod", undefined);
  }
};
```

### Pattern 4: `trigger` - Manual Validation

Trigger validation programmatically:

```typescript
// Validate single field
await trigger('email');

// Validate multiple fields
await trigger(['email', 'password']);

// Validate all fields
await trigger();

// Returns boolean (true if valid)
const isValid = await trigger('email');
if (isValid) {
  // Proceed
}
```

```typescript
const handleL2BlockTimeChange = async (
  e: React.ChangeEvent<HTMLInputElement>
) => {
  setValue("networkAndChain.l2BlockTime", e.target.value);

  // Validate both l2BlockTime AND outputRootFreq
  // (because outputRootFreq must be multiple of l2BlockTime)
  await trigger([
    "networkAndChain.l2BlockTime",
    "networkAndChain.outputRootFreq",
  ]);
};
```

**Why validate both fields?**
The Zod schema has a cross-field validation rule:
```typescript
.refine(
  (data) => {
    // outputRootFreq must be multiple of l2BlockTime
    return data.outputRootFreq % data.l2BlockTime === 0;
  }
)
```

When `l2BlockTime` changes, `outputRootFreq` validity might change too!

### Pattern 5: `formState` - Form Status

Track form state:

```typescript
const {
  errors,           // Validation errors
  isDirty,          // Has form been modified?
  isSubmitting,     // Is form currently submitting?
  isSubmitted,      // Has form been submitted?
  isValid,          // Are all fields valid?
  touchedFields,    // Which fields have been touched?
  dirtyFields,      // Which fields have been modified?
} = formState;
```

```typescript
<Button
  type="submit"
  className="w-full"
  disabled={isLoading || isLoggingIn || form.formState.isSubmitting}
>
  {isLoading || isLoggingIn || form.formState.isSubmitting
    ? "Signing in..."
    : "Sign In"}
</Button>
```

---

## Multi-Step Forms

This project has an excellent multi-step form example for rollup creation.

### Architecture

```
CreateRollupStepper (Container)
  ├── FormProvider (shares form across steps)
  ├── Step 1: NetworkAndChainStep
  ├── Step 2: AccountAndAwsStep
  ├── Step 3: DaoCandidateStep
  └── Step 4: ReviewAndDeployStep
```

### Implementation

```typescript
export function useCreateRollup() {
  const { state, updateFormData, updateCurrentStep } = useRollupCreationContext();

  // Single form for all steps
  const form = useForm<CreateRollupFormData>({
    resolver: zodResolver(createRollupSchema),
    defaultValues: state.formData || defaultFormData,
  });

  // Watch for changes and save to context
  useEffect(() => {
    const subscription = form.watch((data) => {
      if (data) {
        updateFormData(data as CreateRollupFormData);
      }
    });
    return () => subscription.unsubscribe();
  }, [form, updateFormData]);

  const goToNextStep = async () => {
    // Validate only current step's fields
    let isValid = false;

    switch (currentStep) {
      case 1: // Network & Chain
        isValid = await form.trigger([
          "networkAndChain.network",
          "networkAndChain.chainName",
          "networkAndChain.l1RpcUrl",
          "networkAndChain.l1BeaconUrl",
        ]);
        break;
      case 2: // Account & AWS
        isValid = await form.trigger([
          "accountAndAws.seedPhrase",
          "accountAndAws.adminAccount",
          // ... all account/AWS fields
        ]);
        break;
      // ... other steps
    }

    if (isValid) {
      updateCurrentStep(currentStep + 1);
    }
  };

  return {
    form,
    currentStep,
    goToNextStep,
    goToPreviousStep,
  };
}
```

**Key Patterns**:

1. **Single Form Instance**
   - One `useForm()` for entire multi-step form
   - Shared via `FormProvider`

2. **Partial Validation**
   - Each step validates only its own fields
   - Use `trigger([...specificFields])`

3. **State Persistence**
   - Form data saved to Context on change
   - Survives navigation between steps

4. **Conditional Validation**
   ```typescript
   // Validate advanced fields only if enabled
   isValid = await form.trigger([
     ...baseFields,
     ...(advancedConfig ? advancedFields : []),
   ]);
   ```

---

## Error Handling

### Pattern 1: Display Field Errors

```typescript
<Input {...register('email')} />
{errors.email && (
  <p className="text-red-500">{errors.email.message}</p>
)}
```

```typescript
<Input
  id="email"
  {...form.register("email")}
  className={
    form.formState.errors.email ? "border-destructive" : ""
  }
/>
{form.formState.errors.email && (
  <p className="text-sm text-destructive">
    {form.formState.errors.email.message}
  </p>
)}
```

### Pattern 2: Nested Field Errors

For nested objects:

```typescript
// Schema
const schema = z.object({
  networkAndChain: z.object({
    network: z.string(),
    chainName: z.string(),
  }),
});

// Access errors
{errors.networkAndChain?.network && (
  <p>{errors.networkAndChain.network.message}</p>
)}
```

```typescript
{errors.networkAndChain?.network && (
  <p className="text-xs text-red-500 mt-1">
    {errors.networkAndChain.network.message}
  </p>
)}
```

### Pattern 3: Manual Error Setting

```typescript
// Set error manually
form.setError('email', {
  type: 'manual',
  message: 'This email is already taken',
});

// Set root/form-level error
form.setError('root', {
  message: 'Server error occurred',
});
```

```typescript
const deployMutation = useDeployRollupMutation({
  onError: (error) => {
    form.setError("root", {
      message: error.message || "Failed to deploy rollup",
    });
  },
});
```

### Pattern 4: Error Styling

```typescript
// Conditional className based on error
<Input
  {...register('email')}
  className={errors.email ? 'border-red-500' : ''}
/>

// Or with utility function
<Input
  {...register('email')}
  className={cn(
    'base-styles',
    errors.email && 'border-red-500'
  )}
/>
```
