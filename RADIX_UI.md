# Radix UI Pattern Guide

## Introduction

**Radix UI** is a low-level, unstyled UI component library that provides accessible, customizable primitives for building high-quality design systems and web applications.

### Key Features
- **Unstyled**: No default styles, full design control
- **Accessible**: Built-in WAI-ARIA compliance
- **Composable**: Flexible component composition
- **Type-Safe**: Full TypeScript support
- **Framework Agnostic**: Works with React (used here), Vue, and more

### Components Used in This Project
This project uses 14 Radix UI primitives:

| Package | Component | Usage |
|---------|-----------|-------|
| `@radix-ui/react-dialog` | Dialog | Modals, confirmation dialogs |
| `@radix-ui/react-select` | Select | Dropdowns, filters |
| `@radix-ui/react-dropdown-menu` | DropdownMenu | Context menus, action menus |
| `@radix-ui/react-tabs` | Tabs | Navigation, content organization |
| `@radix-ui/react-tooltip` | Tooltip | Helper text on hover |
| `@radix-ui/react-switch` | Switch | Toggle controls |
| `@radix-ui/react-alert-dialog` | AlertDialog | Destructive confirmations |
| `@radix-ui/react-checkbox` | Checkbox | Form inputs |
| `@radix-ui/react-label` | Label | Form labels |
| `@radix-ui/react-avatar` | Avatar | User profile images |
| `@radix-ui/react-progress` | Progress | Loading indicators |
| `@radix-ui/react-separator` | Separator | Visual dividers |
| `@radix-ui/react-slot` | Slot | Polymorphic components |
| `@radix-ui/react-toggle` | Toggle | Binary state buttons |

---

## Why Radix UI?

### Problem: Building Accessible UI is Hard
Creating accessible, keyboard-navigable, screen-reader-friendly components from scratch requires:
- Deep ARIA knowledge
- Complex focus management
- Keyboard event handling
- Portal rendering for modals/tooltips
- State synchronization

### Solution: Radix UI Primitives
Radix handles all the complexity while giving you full styling control.

**Example: Dialog (Modal)**

<table>
<tr>
<th>Without Radix UI</th>
<th>With Radix UI</th>
</tr>
<tr>
<td>

```typescript
// 100+ lines of code
const Modal = () => {
  const [open, setOpen] = useState(false);
  const modalRef = useRef(null);

  // Handle escape key
  useEffect(() => {
    const handleEscape = (e) => {
      if (e.key === 'Escape') setOpen(false);
    };
    document.addEventListener('keydown', handleEscape);
    return () => document.removeEventListener('keydown', handleEscape);
  }, []);

  // Trap focus inside modal
  // Prevent body scroll
  // Handle outside click
  // Portal rendering
  // ... many more lines
};
```

</td>
<td>

```typescript
// 20 lines of code
import * as Dialog from '@radix-ui/react-dialog';

const Modal = () => (
  <Dialog.Root>
    <Dialog.Trigger>Open</Dialog.Trigger>
    <Dialog.Portal>
      <Dialog.Overlay />
      <Dialog.Content>
        <Dialog.Title>Title</Dialog.Title>
        <Dialog.Description>
          Description
        </Dialog.Description>
      </Dialog.Content>
    </Dialog.Portal>
  </Dialog.Root>
);
```

</td>
</tr>
</table>

Radix automatically handles:
- ESC key to close
- Focus trap (can't tab outside)
- Body scroll lock
- Outside click detection
- ARIA attributes
- Portal rendering

---

## Installation and Setup

### Package Installation
```bash
npm install @radix-ui/react-dialog
npm install @radix-ui/react-select
npm install @radix-ui/react-dropdown-menu
# ... and 11 more packages
```

### Project Setup
This project follows the **shadcn/ui pattern**: wrap Radix primitives with styled components in `/src/components/ui/`.

```
src/
└── components/
    └── ui/
        ├── dialog.tsx         # Styled Dialog wrapper
        ├── select.tsx         # Styled Select wrapper
        ├── dropdown-menu.tsx  # Styled DropdownMenu wrapper
        └── ...
```

**Benefits**:
1. Centralized styling with Tailwind CSS
2. Consistent design system
3. Easy to customize globally
4. Type-safe component API

---

## Core Concepts

### 1. Composition Pattern
Radix components are built from smaller, composable parts.

```typescript
// Dialog is composed of multiple parts
<Dialog.Root>              {/* State container */}
  <Dialog.Trigger />       {/* Opens the dialog */}
  <Dialog.Portal>          {/* Renders outside DOM hierarchy */}
    <Dialog.Overlay />     {/* Background overlay */}
    <Dialog.Content>       {/* Main content */}
      <Dialog.Title />     {/* Accessible title */}
      <Dialog.Description /> {/* Accessible description */}
      <Dialog.Close />     {/* Closes the dialog */}
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

**Why Composition?**
- **Flexibility**: Use only the parts you need
- **Customization**: Style each part independently
- **Semantics**: Clear relationship between parts

### 2. Controlled vs Uncontrolled
Radix components support both patterns.

```typescript
// Uncontrolled (internal state)
<Dialog.Root>
  <Dialog.Trigger>Open</Dialog.Trigger>
  {/* Radix manages open/close state */}
</Dialog.Root>

// Controlled (external state)
const [open, setOpen] = useState(false);

<Dialog.Root open={open} onOpenChange={setOpen}>
  <Dialog.Trigger>Open</Dialog.Trigger>
  {/* You control open/close state */}
</Dialog.Root>
```

**When to Use Each:**
- **Uncontrolled**: Simple use cases, no external logic needed
- **Controlled**: Need to sync with external state, trigger side effects

### 3. Primitives vs Styled Components
Radix provides **primitives** (unstyled), this project wraps them in **styled components**.

```typescript
// Radix Primitive (unstyled)
import * as Dialog from "@radix-ui/react-dialog";

<Dialog.Trigger>Open</Dialog.Trigger>

// Styled Component (this project)
import { Dialog, DialogTrigger } from "@/components/ui/dialog";

<DialogTrigger>Open</DialogTrigger>
```

### 4. Data Attributes for Styling
Radix uses `data-*` attributes for state-based styling.

```typescript
<Dialog.Overlay
  data-state="open"    // or "closed"
/>

// Style based on state
<Dialog.Overlay
  className="data-[state=open]:animate-in data-[state=closed]:animate-out"
/>
```

**Common Data Attributes:**
- `data-state`: `"open"`, `"closed"`, `"checked"`, `"unchecked"`
- `data-disabled`: Boolean
- `data-highlighted`: Boolean (dropdown items)
- `data-side`: `"top"`, `"bottom"`, `"left"`, `"right"` (tooltips, popovers)

### 5. Portal Rendering
Components like Dialog, Dropdown, Tooltip render outside the DOM hierarchy to avoid z-index issues.

```typescript
<Dialog.Portal>
  {/* Renders at document.body by default */}
  <Dialog.Content />
</Dialog.Portal>

// Custom portal container
<Dialog.Portal container={customElement}>
  <Dialog.Content />
</Dialog.Portal>
```

---

## Component Patterns

### Pattern 1: Dialog (Modal)

#### Basic Structure
```typescript
import * as DialogPrimitive from "@radix-ui/react-dialog";

<DialogPrimitive.Root>
  <DialogPrimitive.Trigger>Open Dialog</DialogPrimitive.Trigger>
  <DialogPrimitive.Portal>
    <DialogPrimitive.Overlay />
    <DialogPrimitive.Content>
      <DialogPrimitive.Title>Title</DialogPrimitive.Title>
      <DialogPrimitive.Description>Description</DialogPrimitive.Description>
      <DialogPrimitive.Close>Close</DialogPrimitive.Close>
    </DialogPrimitive.Content>
  </DialogPrimitive.Portal>
</DialogPrimitive.Root>
```

#### Styled Wrapper (Project Pattern)
```typescript
// src/components/ui/dialog.tsx
"use client"

import * as React from "react"
import * as DialogPrimitive from "@radix-ui/react-dialog"
import { XIcon } from "lucide-react"
import { cn } from "@/lib/utils"

function Dialog({
  ...props
}: React.ComponentProps<typeof DialogPrimitive.Root>) {
  return <DialogPrimitive.Root data-slot="dialog" {...props} />
}

function DialogTrigger({
  ...props
}: React.ComponentProps<typeof DialogPrimitive.Trigger>) {
  return <DialogPrimitive.Trigger data-slot="dialog-trigger" {...props} />
}

function DialogContent({
  className,
  children,
  showCloseButton = true,
  ...props
}: React.ComponentProps<typeof DialogPrimitive.Content> & {
  showCloseButton?: boolean
}) {
  return (
    <DialogPrimitive.Portal>
      <DialogPrimitive.Overlay
        className={cn(
          "fixed inset-0 z-50 bg-black/50",
          "data-[state=open]:animate-in data-[state=closed]:animate-out",
          "data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0"
        )}
      />
      <DialogPrimitive.Content
        className={cn(
          "fixed top-[50%] left-[50%] z-50",
          "translate-x-[-50%] translate-y-[-50%]",
          "w-full max-w-lg p-6",
          "bg-background rounded-lg border shadow-lg",
          "data-[state=open]:animate-in data-[state=closed]:animate-out",
          "data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0",
          "data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95",
          className
        )}
        {...props}
      >
        {children}
        {showCloseButton && (
          <DialogPrimitive.Close className="absolute top-4 right-4">
            <XIcon />
          </DialogPrimitive.Close>
        )}
      </DialogPrimitive.Content>
    </DialogPrimitive.Portal>
  )
}

export { Dialog, DialogTrigger, DialogContent }
```

**Key Features:**
- Overlay with fade animation
- Content with zoom animation
- Optional close button
- Centered positioning
- Accessible title/description

---

### Pattern 2: Select (Dropdown)

#### Basic Structure
```typescript
import * as SelectPrimitive from "@radix-ui/react-select";

<SelectPrimitive.Root>
  <SelectPrimitive.Trigger>
    <SelectPrimitive.Value placeholder="Select..." />
    <SelectPrimitive.Icon />
  </SelectPrimitive.Trigger>
  <SelectPrimitive.Portal>
    <SelectPrimitive.Content>
      <SelectPrimitive.Viewport>
        <SelectPrimitive.Item value="1">
          <SelectPrimitive.ItemText>Option 1</SelectPrimitive.ItemText>
          <SelectPrimitive.ItemIndicator />
        </SelectPrimitive.Item>
      </SelectPrimitive.Viewport>
    </SelectPrimitive.Content>
  </SelectPrimitive.Portal>
</SelectPrimitive.Root>
```

#### Styled Wrapper (Project Pattern)
```typescript
// src/components/ui/select.tsx
"use client";

import * as SelectPrimitive from "@radix-ui/react-select";
import { CheckIcon, ChevronDownIcon } from "lucide-react";
import { cn } from "@/lib/utils";

function SelectTrigger({
  className,
  size = "default",
  children,
  ...props
}: React.ComponentProps<typeof SelectPrimitive.Trigger> & {
  size?: "sm" | "default";
}) {
  return (
    <SelectPrimitive.Trigger
      className={cn(
        "flex items-center justify-between gap-2",
        "rounded-md border bg-transparent px-3 py-2",
        "text-sm shadow-xs transition-[color,box-shadow]",
        "focus-visible:border-ring focus-visible:ring-[3px]",
        "disabled:cursor-not-allowed disabled:opacity-50",
        "data-[size=default]:h-9 data-[size=sm]:h-8",
        className
      )}
      {...props}
    >
      {children}
      <SelectPrimitive.Icon asChild>
        <ChevronDownIcon className="size-4 opacity-50" />
      </SelectPrimitive.Icon>
    </SelectPrimitive.Trigger>
  );
}

function SelectItem({
  className,
  children,
  ...props
}: React.ComponentProps<typeof SelectPrimitive.Item>) {
  return (
    <SelectPrimitive.Item
      className={cn(
        "relative flex items-center gap-2",
        "rounded-sm py-1.5 pr-8 pl-2",
        "cursor-default select-none",
        "focus:bg-accent focus:text-accent-foreground",
        "data-[disabled]:pointer-events-none data-[disabled]:opacity-50",
        className
      )}
      {...props}
    >
      <span className="absolute right-2 flex size-3.5 items-center justify-center">
        <SelectPrimitive.ItemIndicator>
          <CheckIcon className="size-4" />
        </SelectPrimitive.ItemIndicator>
      </span>
      <SelectPrimitive.ItemText>{children}</SelectPrimitive.ItemText>
    </SelectPrimitive.Item>
  );
}

export { Select, SelectTrigger, SelectItem }
```

**Key Features:**
- Keyboard navigation (arrow keys)
- Type-ahead search
- Scroll buttons for long lists
- Selected item indicator (checkmark)
- Custom trigger sizes

---

### Pattern 3: Dropdown Menu

#### Basic Structure
```typescript
import * as DropdownMenuPrimitive from "@radix-ui/react-dropdown-menu";

<DropdownMenuPrimitive.Root>
  <DropdownMenuPrimitive.Trigger>Menu</DropdownMenuPrimitive.Trigger>
  <DropdownMenuPrimitive.Portal>
    <DropdownMenuPrimitive.Content>
      <DropdownMenuPrimitive.Item>Item 1</DropdownMenuPrimitive.Item>
      <DropdownMenuPrimitive.Item>Item 2</DropdownMenuPrimitive.Item>
      <DropdownMenuPrimitive.Separator />
      <DropdownMenuPrimitive.Item>Item 3</DropdownMenuPrimitive.Item>
    </DropdownMenuPrimitive.Content>
  </DropdownMenuPrimitive.Portal>
</DropdownMenuPrimitive.Root>
```

#### Advanced Features
```typescript
<DropdownMenu>
  <DropdownMenuTrigger>Actions</DropdownMenuTrigger>
  <DropdownMenuContent>
    {/* Regular item */}
    <DropdownMenuItem>Edit</DropdownMenuItem>

    {/* Destructive item */}
    <DropdownMenuItem variant="destructive">
      Delete
    </DropdownMenuItem>

    {/* Checkbox item */}
    <DropdownMenuCheckboxItem checked={showPanel}>
      Show Panel
    </DropdownMenuCheckboxItem>

    {/* Radio group */}
    <DropdownMenuRadioGroup value={theme}>
      <DropdownMenuRadioItem value="light">Light</DropdownMenuRadioItem>
      <DropdownMenuRadioItem value="dark">Dark</DropdownMenuRadioItem>
    </DropdownMenuRadioGroup>

    {/* Sub-menu */}
    <DropdownMenuSub>
      <DropdownMenuSubTrigger>More</DropdownMenuSubTrigger>
      <DropdownMenuSubContent>
        <DropdownMenuItem>Sub Item 1</DropdownMenuItem>
      </DropdownMenuSubContent>
    </DropdownMenuSub>
  </DropdownMenuContent>
</DropdownMenu>
```

**Key Features:**
- Keyboard navigation
- Checkbox and radio items
- Nested submenus
- Keyboard shortcuts display
- Destructive variant for dangerous actions

---

### Pattern 4: Tabs

#### Basic Structure
```typescript
import * as TabsPrimitive from "@radix-ui/react-tabs";

<TabsPrimitive.Root defaultValue="tab1">
  <TabsPrimitive.List>
    <TabsPrimitive.Trigger value="tab1">Tab 1</TabsPrimitive.Trigger>
    <TabsPrimitive.Trigger value="tab2">Tab 2</TabsPrimitive.Trigger>
  </TabsPrimitive.List>
  <TabsPrimitive.Content value="tab1">
    Content 1
  </TabsPrimitive.Content>
  <TabsPrimitive.Content value="tab2">
    Content 2
  </TabsPrimitive.Content>
</TabsPrimitive.Root>
```

#### Styled Wrapper
```typescript
// src/components/ui/tabs.tsx
function TabsTrigger({
  className,
  ...props
}: React.ComponentProps<typeof TabsPrimitive.Trigger>) {
  return (
    <TabsPrimitive.Trigger
      className={cn(
        "inline-flex items-center justify-center gap-1.5",
        "rounded-md px-2 py-1",
        "text-sm font-medium whitespace-nowrap",
        "focus-visible:ring-[3px]",
        "data-[state=active]:bg-background",
        "data-[state=active]:shadow-sm",
        "disabled:pointer-events-none disabled:opacity-50",
        className
      )}
      {...props}
    />
  );
}
```

**Key Features:**
- Automatic ARIA attributes
- Keyboard navigation (arrow keys)
- Active state styling
- Only renders active content (performance)

---

### Pattern 5: Tooltip

#### Basic Structure
```typescript
import * as TooltipPrimitive from "@radix-ui/react-tooltip";

<TooltipPrimitive.Provider>
  <TooltipPrimitive.Root>
    <TooltipPrimitive.Trigger>Hover me</TooltipPrimitive.Trigger>
    <TooltipPrimitive.Portal>
      <TooltipPrimitive.Content>
        Tooltip content
        <TooltipPrimitive.Arrow />
      </TooltipPrimitive.Content>
    </TooltipPrimitive.Portal>
  </TooltipPrimitive.Root>
</TooltipPrimitive.Provider>
```

#### Styled Wrapper
```typescript
// src/components/ui/tooltip.tsx
function TooltipProvider({
  delayDuration = 0,
  ...props
}: React.ComponentProps<typeof TooltipPrimitive.Provider>) {
  return (
    <TooltipPrimitive.Provider delayDuration={delayDuration} {...props} />
  );
}

function TooltipContent({
  className,
  sideOffset = 0,
  children,
  ...props
}: React.ComponentProps<typeof TooltipPrimitive.Content>) {
  return (
    <TooltipPrimitive.Portal>
      <TooltipPrimitive.Content
        sideOffset={sideOffset}
        className={cn(
          "z-50 rounded-md px-3 py-1.5",
          "bg-primary text-primary-foreground text-xs",
          "animate-in fade-in-0 zoom-in-95",
          "data-[state=closed]:animate-out",
          className
        )}
        {...props}
      >
        {children}
        <TooltipPrimitive.Arrow className="fill-primary" />
      </TooltipPrimitive.Content>
    </TooltipPrimitive.Portal>
  );
}
```

**Key Features:**
- Provider for shared configuration
- Customizable delay duration
- Arrow pointing to trigger
- Side/alignment positioning
- Animation on show/hide

---

### Pattern 6: Switch (Toggle)

#### Basic Structure
```typescript
import * as SwitchPrimitive from "@radix-ui/react-switch";

<SwitchPrimitive.Root>
  <SwitchPrimitive.Thumb />
</SwitchPrimitive.Root>
```

#### Styled Wrapper
```typescript
// src/components/ui/switch.tsx
function Switch({
  className,
  ...props
}: React.ComponentProps<typeof SwitchPrimitive.Root>) {
  return (
    <SwitchPrimitive.Root
      className={cn(
        "inline-flex h-[1.15rem] w-8 shrink-0 items-center",
        "rounded-full border border-transparent shadow-xs",
        "data-[state=checked]:bg-primary",
        "data-[state=unchecked]:bg-input",
        "focus-visible:ring-[3px]",
        "disabled:cursor-not-allowed disabled:opacity-50",
        className
      )}
      {...props}
    >
      <SwitchPrimitive.Thumb
        className={cn(
          "block size-4 rounded-full bg-background",
          "transition-transform",
          "data-[state=checked]:translate-x-[calc(100%-2px)]",
          "data-[state=unchecked]:translate-x-0"
        )}
      />
    </SwitchPrimitive.Root>
  );
}
```

**Key Features:**
- Checkbox semantics (accessible)
- Smooth thumb animation
- Controlled/uncontrolled modes
- Disabled state

---

## Real-World Examples

### Example 1: Filters with Select
```typescript
// src/features/rollup/components/RollupFilters.tsx
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";

export function RollupFilters({
  statusFilter,
  setStatusFilter,
}: RollupFiltersProps) {
  return (
    <Select value={statusFilter} onValueChange={setStatusFilter}>
      <SelectTrigger className="w-full sm:w-48">
        <SelectValue placeholder="All Status" />
      </SelectTrigger>
      <SelectContent>
        <SelectItem value="all">All Status</SelectItem>
        <SelectItem value="DEPLOYED">Deployed</SelectItem>
        <SelectItem value="PENDING">Pending</SelectItem>
        <SelectItem value="FAILED">Failed</SelectItem>
      </SelectContent>
    </Select>
  );
}
```

**Use Case**: Filter rollups by status
**Features Used**:
- Controlled select (value + onValueChange)
- Placeholder text
- Custom trigger width

---

### Example 2: Tabs for Navigation
```typescript
// src/features/rollup/components/detail/RollupDetailTabs.tsx
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";

export function RollupDetailTabs({
  stack,
  currentTab = "overview",
}: RollupDetailTabsProps) {
  const router = useRouter();

  const handleTabChange = (value: string) => {
    const pathname = window.location.pathname;
    const searchParams = new URLSearchParams(window.location.search);
    searchParams.set("tab", value);
    router.push(`${pathname}?${searchParams.toString()}`);
  };

  return (
    <Tabs value={currentTab} onValueChange={handleTabChange}>
      <TabsList className="grid w-full grid-cols-5">
        <TabsTrigger value="overview">Overview</TabsTrigger>
        <TabsTrigger value="deployments">Deployment History</TabsTrigger>
        <TabsTrigger value="components">Integrations</TabsTrigger>
        <TabsTrigger value="metadata">Metadata</TabsTrigger>
        <TabsTrigger value="settings">Settings</TabsTrigger>
      </TabsList>

      <TabsContent value="overview">
        <OverviewTab stack={stack} />
      </TabsContent>

      <TabsContent value="deployments">
        <DeploymentsTab stack={stack} />
      </TabsContent>

      {/* ... more tabs */}
    </Tabs>
  );
}
```

**Use Case**: Navigate between rollup detail sections
**Features Used**:
- Controlled tabs (synced with URL)
- Custom tab change handler
- Grid layout for equal-width tabs
- URL-based navigation

---

### Example 3: Log Viewer Dialog
```typescript
// src/features/rollup/components/detail/LogDialog.tsx
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
import { Switch } from "@/components/ui/switch";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import { Label } from "@/components/ui/label";

export function LogDialog({
  open,
  onOpenChange,
  deployment,
  stackId,
}: LogDialogProps) {
  const [isRealtimeEnabled, setIsRealtimeEnabled] = React.useState(true);
  const [lineLimit, setLineLimit] = React.useState<number>(200);

  const { data: logs = [], refetch } = useThanosDeploymentLogsQuery(
    stackId,
    deployment?.id,
    {
      limit: lineLimit,
      refetchIntervalMs: isRealtimeEnabled ? 5000 : false,
    }
  );

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="max-w-4xl max-h-[80vh]">
        <DialogHeader>
          <DialogTitle>Deployment logs</DialogTitle>
          <DialogDescription>
            Showing recent logs for {deployment?.step}
          </DialogDescription>
        </DialogHeader>

        {/* Controls */}
        <div className="flex items-center gap-4">
          {/* Realtime Toggle */}
          <div className="flex items-center gap-2">
            <Label htmlFor="realtime-switch">Realtime</Label>
            <Switch
              id="realtime-switch"
              checked={isRealtimeEnabled}
              onCheckedChange={setIsRealtimeEnabled}
            />
          </div>

          {/* Line Limit Dropdown */}
          <div className="flex items-center gap-2">
            <Label htmlFor="line-limit">Lines</Label>
            <Select
              value={lineLimit.toString()}
              onValueChange={(value) => setLineLimit(parseInt(value))}
            >
              <SelectTrigger id="line-limit" className="w-32">
                <SelectValue />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="10">10 lines</SelectItem>
                <SelectItem value="50">50 lines</SelectItem>
                <SelectItem value="100">100 lines</SelectItem>
                <SelectItem value="200">200 lines</SelectItem>
                <SelectItem value="500">500 lines</SelectItem>
              </SelectContent>
            </Select>
          </div>
        </div>

        {/* Log Content */}
        <div className="rounded-md bg-slate-950 text-slate-100 overflow-y-auto">
          <pre className="text-xs font-mono p-3">
            {logs.map((log) => parseLogMessage(log)).join("")}
          </pre>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

**Use Case**: View deployment logs with realtime updates
**Components Used**:
- Dialog: Modal container
- Switch: Toggle realtime updates
- Select: Choose log line limit
- Label: Accessible form labels

**Key Features**:
- Controlled dialog (open/onOpenChange)
- Realtime toggle affects query refetch interval
- Line limit changes refetch query with new limit
- All components work together seamlessly

---

### Example 4: Button with Slot
```typescript
// src/components/ui/button.tsx
import { Slot } from "@radix-ui/react-slot";

function Button({
  asChild = false,
  ...props
}: React.ComponentProps<"button"> & {
  asChild?: boolean;
}) {
  const Comp = asChild ? Slot : "button";
  return <Comp {...props} />;
}

// Usage: Render as Link instead of button
import Link from "next/link";

<Button asChild>
  <Link href="/rollup/create">Create Rollup</Link>
</Button>

// Result: <a> element with button styling
```

**Use Case**: Polymorphic button (button, link, etc.)
**Key Feature**: `asChild` prop merges props with child element

---

## Styling with Tailwind CSS

### Data Attribute Styling
Radix components use `data-*` attributes for state. Tailwind supports this via `data-[]` modifier.

```typescript
<Dialog.Overlay
  className={cn(
    // Default styles
    "fixed inset-0 bg-black/50",

    // State-based styles
    "data-[state=open]:animate-in",
    "data-[state=closed]:animate-out",
    "data-[state=open]:fade-in-0",
    "data-[state=closed]:fade-out-0"
  )}
/>
```

### Common State Patterns
```typescript
// Open/Closed
data-[state=open]:opacity-100
data-[state=closed]:opacity-0

// Checked/Unchecked
data-[state=checked]:bg-primary
data-[state=unchecked]:bg-gray-200

// Disabled
data-[disabled]:opacity-50
data-[disabled]:pointer-events-none

// Highlighted (dropdown items)
data-[highlighted]:bg-accent

// Side (tooltips, popovers)
data-[side=top]:slide-in-from-bottom-2
data-[side=bottom]:slide-in-from-top-2
```

### Animation Classes
```typescript
// Fade in/out
"animate-in fade-in-0"
"animate-out fade-out-0"

// Zoom in/out
"zoom-in-95"  // scale from 95%
"zoom-out-95" // scale to 95%

// Slide in
"slide-in-from-top-2"    // from top
"slide-in-from-bottom-2" // from bottom
"slide-in-from-left-2"   // from left
"slide-in-from-right-2"  // from right
```

### cn() Utility
Merge Tailwind classes with conditional logic.

```typescript
// src/lib/utils.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

// Usage
<div className={cn(
  "base-class",
  isActive && "active-class",
  className // allow override
)} />
```

---

## Accessibility Features

Radix UI components are built with accessibility in mind. Here's what they handle automatically:

### 1. ARIA Attributes
Radix adds correct ARIA attributes without you needing to remember them.

```typescript
// You write:
<Dialog.Root>
  <Dialog.Trigger>Open</Dialog.Trigger>
  <Dialog.Content>
    <Dialog.Title>Title</Dialog.Title>
  </Dialog.Content>
</Dialog.Root>

// Radix renders:
<button
  aria-haspopup="dialog"
  aria-expanded="false"
  aria-controls="radix-123"
>
  Open
</button>

<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="radix-124"
  id="radix-123"
>
  <h2 id="radix-124">Title</h2>
</div>
```

### 2. Keyboard Navigation
All components support keyboard navigation.

| Component | Keyboard Support |
|-----------|------------------|
| Dialog | ESC to close, Tab to cycle focus |
| Select | Arrow keys to navigate, Enter to select, Type to search |
| DropdownMenu | Arrow keys to navigate, Enter to select, ESC to close |
| Tabs | Arrow keys to switch tabs |
| Tooltip | Focus trigger to show, Blur to hide |

### 3. Focus Management
- **Focus Trap**: Modals trap focus inside (can't tab outside)
- **Focus Return**: Focus returns to trigger after closing
- **Focus Visible**: Only show outline when navigating with keyboard

```typescript
<Dialog.Content>
  {/* Focus automatically set to first focusable element */}
  <input autoFocus />
</Dialog.Content>
```

### 4. Screen Reader Support
- Proper role attributes (`role="dialog"`, `role="menu"`, etc.)
- Accessible names via `aria-label` or `aria-labelledby`
- Accessible descriptions via `aria-describedby`
- Status announcements via `aria-live` regions

### 5. Disabled State
Disabled elements are properly handled.

```typescript
<DropdownMenuItem disabled>
  {/* Automatically adds:
      - aria-disabled="true"
      - data-disabled
      - pointer-events: none
  */}
  Disabled Item
</DropdownMenuItem>
```

---

## Best Practices

### 1. Always Use Controlled Components for Complex Logic
```typescript
✅ Good: Controlled
const [open, setOpen] = useState(false);

const handleDelete = async () => {
  await deleteItem();
  setOpen(false); // Close after delete
  router.push('/list'); // Navigate away
};

<Dialog open={open} onOpenChange={setOpen}>
  {/* ... */}
</Dialog>

❌ Bad: Uncontrolled (can't control close timing)
<Dialog>
  <DialogTrigger>Delete</DialogTrigger>
  {/* Can't close programmatically */}
</Dialog>
```

### 2. Provide Accessible Names
```typescript
✅ Good: Has Title and Description
<Dialog.Root>
  <Dialog.Content>
    <Dialog.Title>Delete Rollup</Dialog.Title>
    <Dialog.Description>
      Are you sure? This action cannot be undone.
    </Dialog.Description>
  </Dialog.Content>
</Dialog.Root>

❌ Bad: Missing Title (screen reader can't announce)
<Dialog.Root>
  <Dialog.Content>
    <p>Are you sure?</p>
  </Dialog.Content>
</Dialog.Root>
```

### 3. Use Portal for Overlays
```typescript
✅ Good: Portal avoids z-index issues
<Dialog.Portal>
  <Dialog.Overlay />
  <Dialog.Content />
</Dialog.Portal>

❌ Bad: No portal, may have z-index conflicts
<Dialog.Overlay />
<Dialog.Content />
```

### 4. Handle Loading and Error States
```typescript
✅ Good: Show loading state
<Select disabled={isLoading}>
  <SelectTrigger>
    <SelectValue placeholder={isLoading ? "Loading..." : "Select"} />
  </SelectTrigger>
</Select>

❌ Bad: No feedback
<Select>
  <SelectTrigger>
    <SelectValue />
  </SelectTrigger>
</Select>
```

### 5. Wrap Radix Primitives in Styled Components
```typescript
✅ Good: Centralized styling
// src/components/ui/button.tsx
export function Button({ variant, size, ...props }) {
  return <button className={buttonVariants({ variant, size })} {...props} />;
}

// Usage
import { Button } from "@/components/ui/button";
<Button variant="destructive">Delete</Button>

❌ Bad: Inline styling everywhere
<button className="bg-red-500 text-white px-4 py-2 rounded">Delete</button>
```

### 6. Use TypeScript for Type Safety
```typescript
✅ Good: Type-safe props
import { Dialog, DialogProps } from "@/components/ui/dialog";

interface DeleteDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  itemName: string;
}

❌ Bad: Loose types
function DeleteDialog({ open, onOpenChange, itemName }: any) {
  // ...
}
```

---

## Common Patterns in This Project

### Pattern 1: Styled Component Wrappers
All Radix primitives are wrapped in `src/components/ui/` with:
- Tailwind CSS styling
- Consistent design system
- Additional props (e.g., `size`, `variant`)
- TypeScript types

```typescript
// src/components/ui/button.tsx
import { Slot } from "@radix-ui/react-slot";
import { cva, type VariantProps } from "class-variance-authority";

const buttonVariants = cva(
  "base-classes",
  {
    variants: {
      variant: {
        default: "...",
        destructive: "...",
        outline: "...",
      },
      size: {
        default: "h-9 px-4",
        sm: "h-8 px-3",
        lg: "h-10 px-6",
      },
    },
  }
);

export function Button({
  variant,
  size,
  asChild,
  ...props
}: React.ComponentProps<"button"> & VariantProps<typeof buttonVariants> & {
  asChild?: boolean;
}) {
  const Comp = asChild ? Slot : "button";
  return <Comp className={cn(buttonVariants({ variant, size }))} {...props} />;
}
```

### Pattern 2: Controlled Dialogs
Dialogs are always controlled for programmatic control.

```typescript
const [open, setOpen] = useState(false);

<Dialog open={open} onOpenChange={setOpen}>
  <DialogTrigger>Open</DialogTrigger>
  <DialogContent>
    <DialogTitle>Title</DialogTitle>
    <Button onClick={() => {
      // Do something
      setOpen(false); // Close programmatically
    }}>
      Confirm
    </Button>
  </DialogContent>
</Dialog>
```

### Pattern 3: Tabs with URL Sync
Tabs sync with URL query parameters for shareable links.

```typescript
const router = useRouter();
const searchParams = useSearchParams();
const currentTab = searchParams.get("tab") || "overview";

<Tabs
  value={currentTab}
  onValueChange={(value) => {
    const params = new URLSearchParams(searchParams);
    params.set("tab", value);
    router.push(`?${params.toString()}`);
  }}
>
  {/* ... */}
</Tabs>
```

### Pattern 4: Select with Controlled State
Selects are controlled for filtering and state management.

```typescript
const [statusFilter, setStatusFilter] = useState("all");

<Select value={statusFilter} onValueChange={setStatusFilter}>
  <SelectTrigger>
    <SelectValue placeholder="All Status" />
  </SelectTrigger>
  <SelectContent>
    <SelectItem value="all">All Status</SelectItem>
    <SelectItem value="active">Active</SelectItem>
  </SelectContent>
</Select>
```

### Pattern 5: Switch with React Query Integration
Switch controls query refetch behavior.

```typescript
const [isRealtimeEnabled, setIsRealtimeEnabled] = useState(true);

const { data } = useQuery({
  queryKey: ["logs"],
  queryFn: fetchLogs,
  refetchInterval: isRealtimeEnabled ? 5000 : false,
});

<Switch
  checked={isRealtimeEnabled}
  onCheckedChange={setIsRealtimeEnabled}
/>
```

---

## Summary

### Key Takeaways
1. **Radix UI = Accessible Primitives**: Unstyled, composable, accessible components
2. **Composition Pattern**: Build complex UIs from small, focused parts
3. **Data Attributes**: Use `data-[state=open]` for state-based styling
4. **Portal Rendering**: Avoid z-index issues with Dialog, Dropdown, Tooltip
5. **Controlled vs Uncontrolled**: Use controlled for complex logic
6. **Styling with Tailwind**: `data-[]` modifiers for state-based styles
7. **Accessibility Built-in**: ARIA, keyboard nav, focus management, screen reader support

### Components Used in This Project
| Component | Use Cases |
|-----------|-----------|
| Dialog | Log viewer, confirmation modals |
| Select | Filters (status, type, network) |
| DropdownMenu | Action menus, context menus |
| Tabs | Rollup detail sections (Overview, Deployments, etc.) |
| Tooltip | Helper text on hover |
| Switch | Realtime logging toggle |
| Checkbox | Form inputs |
| Label | Form labels |
| Button (with Slot) | Polymorphic buttons |

### Architectural Pattern
```
Radix Primitive → Styled Wrapper → Feature Component
     ↓                  ↓                   ↓
  @radix-ui/     src/components/ui/    src/features/
  react-dialog      dialog.tsx        rollup/components/
```

### Why This Pattern Works
- **Separation of Concerns**: Logic (Radix) + Styling (Tailwind) + Business Logic (Features)
- **Reusability**: Styled components used across features
- **Consistency**: Single source of truth for design
- **Maintainability**: Easy to update styles globally
- **Accessibility**: Handled by Radix, guaranteed across app

---

**Related Documentation**:
- [React Hook Form Guide](./react-hook-form-guide.md) - Form components use Radix Label, Checkbox, Select
- [Project Structure Guide](./project-structure-guide.md) - `/src/components/ui/` pattern
- [Next.js App Router Guide](./nextjs-app-router-guide.md) - Dialog, Tabs used in pages

---

## Conclusion

Radix UI provides production-ready, accessible primitives that handle the complex parts of UI development (keyboard nav, focus management, ARIA) while giving you complete control over styling. By wrapping Radix primitives in styled components (shadcn/ui pattern), this project achieves:

- **Accessibility**: WAI-ARIA compliant, keyboard navigable, screen reader friendly
- **Consistency**: Centralized design system in `/src/components/ui/`
- **Flexibility**: Composable primitives for any use case
- **Developer Experience**: Type-safe, well-documented, easy to use

Understanding Radix UI patterns is essential for building modern, accessible React applications.
