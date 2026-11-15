# Code Organization Patterns

Enforce strict separation of concerns: types in types/, helpers in lib/, constants in
constants/. Components contain ONLY JSX and component logic.

## Rules

### Keep it simple, and DRY

- **Don't overengineer.** Solve the problem in front of you; don't add configuration,
  layers, or generalization for a second use case that doesn't exist yet.
- **Follow DRY for logic, not just components** — the same validation, formatting, or
  calculation written twice belongs in `src/lib/` or `src/utils/` once.
- **Reuse before you create.** Search for an existing helper, hook, or component before
  writing a new one; extend what's there rather than forking a near-copy.
- **Don't introduce a new library** when an existing dependency already covers the need.

### Meaningful names

- **Name files, functions, variables, hooks, types, schemas, and components for what
  they are or do.** A reader should not need to open a file to learn what it holds.
- **Avoid vague names** — `data`, `item`, `handler`, `utils2`, `temp` — where a specific
  one is available.

### No types in component files

- **Every type definition must live in `src/types/` or `src/schema/`**.
- **Exception**: A component's own `Props` interface is allowed inline.
- **Never nest types in components** — they're harder to reuse and break tree-shaking.

```ts
// ✅ Good: type in src/types/
// src/types/project.ts
export interface Project {
  id: string;
  address: string;
  price: number;
  ownerId: string;
}

// src/components/projects/project-card.tsx
import { Project } from '@/types/project';

interface Props {
  project: Project;
  onSelect?: (id: string) => void;
}

export function ProjectCard({ project, onSelect }: Props) {
  // component code
}
```

```ts
// ❌ Bad: type defined in component file
// src/components/projects/project-card.tsx
interface Project {
  id: string;
  address: string;
  price: number;
}

export function ProjectCard({ project }: { project: Project }) {
  // ...
}
```

### No helper functions in component files

- **Every utility function must live in `src/lib/` or `src/utils/`**.
- **Check if a similar function exists first** before creating a new one.
- **Reuse existing helpers** — don't duplicate logic across component files.
- **Extract even if used only once** if the function has independent logic or formatting
  responsibility.

```ts
// ✅ Good: helper in src/lib/
// src/lib/formatting.ts
export function formatCurrency(cents: number, currency = 'USD'): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency,
  }).format(cents / 100);
}

export function formatDate(date: Date): string {
  return new Intl.DateTimeFormat('en-US', {
    month: 'short',
    day: 'numeric',
    year: 'numeric',
  }).format(date);
}

// src/components/projects/project-card.tsx
import { formatCurrency, formatDate } from '@/lib/formatting';
import { Project } from '@/types/project';

export function ProjectCard({ project }: { project: Project }) {
  return (
    <div>
      <h3>{project.address}</h3>
      <p>{formatCurrency(project.price)}</p>
      <span>{formatDate(new Date(project.createdAt))}</span>
    </div>
  );
}
```

```ts
// ❌ Bad: helper logic in component file
// src/components/projects/project-card.tsx
export function ProjectCard({ project }: { project: Project }) {
  const formatCurrency = (cents: number) => {
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD',
    }).format(cents / 100);
  };

  const formatDate = (date: Date) => {
    return new Intl.DateTimeFormat('en-US', {
      month: 'short',
      day: 'numeric',
      year: 'numeric',
    }).format(date);
  };

  return (
    <div>
      <h3>{project.address}</h3>
      <p>{formatCurrency(project.price)}</p>
      <span>{formatDate(new Date(project.createdAt))}</span>
    </div>
  );
}
```

### Where to put helpers

- **Formatting (currency, dates, phone, etc.)**: `src/lib/formatting.ts`
- **Calculations and math**: `src/lib/calculations.ts` or domain-specific file
- **String manipulation (slugs, case conversion, etc.)**: `src/lib/strings.ts`
- **Array operations (dedup, group, sort)**: `src/lib/arrays.ts`
- **Object/data transformation**: `src/lib/transform.ts` or domain-specific file
- **Supabase/database queries**: `src/lib/supabase.ts` or `src/lib/<entity>.ts`
- **API calls**: `src/lib/api.ts` or domain-specific file
- **Validation helpers**: `src/lib/validation.ts`

If a helper file doesn't exist yet, **create it**. If one exists, **reuse it**.

```
src/lib/
├── formatting.ts       # currency, date, phone, etc.
├── calculations.ts     # math, metrics, aggregations
├── strings.ts          # slugs, case conversion, truncate
├── arrays.ts           # dedup, group, flatten, etc.
├── transform.ts        # object/data transformation
├── validation.ts       # custom validators (beyond Zod)
├── supabase.ts         # Supabase client & queries
├── api.ts              # external API calls
├── auth.ts             # auth helpers
└── <entity>.ts         # entity-specific helpers (projects.ts, tasks.ts, etc.)
```

### Static content in constants

- **Feature lists, nav items, step copy, config values** go in `src/constants/`.
- **Never inline static data** in components or pages.

```ts
// ✅ Good: constants in src/constants/
// src/constants/onboarding-steps.ts
export const ONBOARDING_STEPS = [
  { id: 'profile', title: 'Create Profile', description: 'Set up your account' },
  { id: 'verify', title: 'Verify Identity', description: 'Complete ID verification' },
  { id: 'preferences', title: 'Set Preferences', description: 'Personalize your experience' },
] as const;

export const TASK_RULES = {
  MIN_INCREMENT: 100, // cents
  MAX_TASKS_PER_PROJECT: 10,
  TASK_EXPIRY_DAYS: 7,
} as const;

// src/app/onboarding/page.tsx
import { ONBOARDING_STEPS } from '@/constants/onboarding-steps';

export default function OnboardingPage() {
  return (
    <div>
      {ONBOARDING_STEPS.map(step => (
        <Step key={step.id} title={step.title} description={step.description} />
      ))}
    </div>
  );
}
```

```tsx
// ❌ Bad: static data inlined
export default function OnboardingPage() {
  const steps = [
    { id: 'profile', title: 'Create Profile', description: 'Set up your account' },
    { id: 'verify', title: 'Verify Identity', description: 'Complete ID verification' },
  ];

  return (
    <div>
      {steps.map(step => (
        <Step key={step.id} title={step.title} description={step.description} />
      ))}
    </div>
  );
}
```

### Component file checklist

Before calling a component file done:

- [ ] **No types defined** (except component's own `Props`)
- [ ] **No helper functions** (even small ones)
- [ ] **No static data inlined** (use `src/constants/`)
- [ ] **No business logic** (use server actions or `src/lib/`)
- [ ] **Only JSX and component state** (useState, useContext, hooks)
- [ ] **Imports are from**: `@/types/`, `@/components/`, `@/constants/`, `@/lib/`, `@/hooks/`

If a component file violates any of these, refactor immediately.

## Example: Before and After

**Before (messy)**:
```tsx
// src/components/projects/project-card.tsx
interface Project {
  id: string;
  address: string;
  price: number;
  createdAt: string;
}

const PRICE_FORMAT_OPTIONS = {
  style: 'currency',
  currency: 'USD',
} as const;

const formatCurrency = (cents: number) => {
  return new Intl.NumberFormat('en-US', PRICE_FORMAT_OPTIONS).format(cents / 100);
};

const formatDate = (date: Date) => {
  return new Intl.DateTimeFormat('en-US', { month: 'short', day: 'numeric' }).format(date);
};

const getStatusLabel = (status: string) => {
  const labels: Record<string, string> = {
    available: 'Available',
    rented: 'Rented',
    pending: 'Pending',
  };
  return labels[status] || 'Unknown';
};

export function ProjectCard({ project, onSelect }: { project: Project; onSelect?: (id: string) => void }) {
  return (
    <div onClick={() => onSelect?.(project.id)}>
      <h3>{project.address}</h3>
      <p>{formatCurrency(project.price)}</p>
      <span>{formatDate(new Date(project.createdAt))}</span>
    </div>
  );
}
```

**After (organized)**:
```ts
// src/types/project.ts
export interface Project {
  id: string;
  address: string;
  price: number;
  createdAt: string;
}

// src/lib/formatting.ts
export function formatCurrency(cents: number, currency = 'USD'): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency,
  }).format(cents / 100);
}

export function formatDate(date: Date): string {
  return new Intl.DateTimeFormat('en-US', {
    month: 'short',
    day: 'numeric',
    year: 'numeric',
  }).format(date);
}

// src/lib/status.ts
export const STATUS_LABELS = {
  available: 'Available',
  rented: 'Rented',
  pending: 'Pending',
} as const;

// src/components/projects/project-card.tsx
import { Project } from '@/types/project';
import { formatCurrency, formatDate } from '@/lib/formatting';

interface Props {
  project: Project;
  onSelect?: (id: string) => void;
}

export function ProjectCard({ project, onSelect }: Props) {
  return (
    <div onClick={() => onSelect?.(project.id)}>
      <h3>{project.address}</h3>
      <p>{formatCurrency(project.price)}</p>
      <span>{formatDate(new Date(project.createdAt))}</span>
    </div>
  );
}
```

Notice: component file went from 40+ lines to 12 lines. It's now **focused, reusable, and testable**.

### No long explanatory comments

- **Code explains itself through naming** — clear function/variable names beat comments.
- **Don't narrate**: no comments saying what the next line does, why a change was made,
  or where code came from. That noise belongs in the PR description, not the file.
- **A comment is justified only** for constraints the code can't show: third-party
  workarounds, non-obvious business rules, deliberate deviations from a standard.
- **Never leave**: commented-out code, changelog-style comments ("updated to fix X"),
  or TODOs without a ticket reference.

```ts
// ❌ Bad
// Loop through the projects and filter out rented ones, then map to cards
const cards = projects.filter(l => l.status !== 'rented').map(toCard);

// ✅ Good (no comment needed — the code says it)
const availableProjectCards = projects
  .filter(isAvailable)
  .map(toCard);

// ✅ Acceptable comment (constraint the code can't show)
// The SDK requires the token to be set before any instance is created (see vendor/#1234)
sdk.accessToken = ANALYTICS_TOKEN;
```
