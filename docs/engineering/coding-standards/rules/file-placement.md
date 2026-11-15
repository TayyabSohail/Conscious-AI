# File Placement Standards

Organize code to keep files focused and prevent premature abstraction. The directory
structure mirrors responsibility boundaries.

## Directory structure

```
src/
├── app/               # Next.js App Router pages and layouts
├── actions/           # Server actions (next-safe-action)
├── components/        # React components (split by feature)
│   ├── ui/            # shadcn/ui + Radix primitives (auto-generated)
│   └── <feature>/     # Feature-specific components (e.g., projects/, task/)
├── config/            # Config & constants (env, theme, etc.)
├── constants/         # Static data: feature lists, query keys, nav items
├── contexts/          # React contexts (use sparingly)
├── controllers/       # Request handlers (for API routes if needed)
├── dtos/              # Data Transfer Objects (request/response shapes)
├── emails/            # Email templates
├── handlers/          # Event/webhook handlers
├── hooks/             # Custom React hooks
├── lib/               # Utility functions and helpers
├── schema/            # Zod schemas (not runtime data)
├── types/             # TypeScript type definitions
└── utils/             # Pure functions (math, formatting, parsing)
```

## Rules

### File naming and imports

- **Component files are kebab-case** — `user-profile.tsx`, never `UserProfile.tsx`.
- **Hook files are `use-*.ts`** — and use `.ts` unless the file actually renders JSX.
- **Always import via the `@/` alias** — `@/components/ui/button`, never a deep relative
  path like `../../../components/ui/button`.

### Choosing a component folder

`src/components/` is large, so pick the folder deliberately:

- **`ui/`** — shadcn/Radix primitives only. Nothing app-specific.
- **`common/`** — app-wide reusable pieces that aren't primitives. This is the canonical
  shared folder.
- **`shared/`** — deprecated. Don't add to it; use `common/`.
- **`<feature>/`** — anything tied to a single feature.
- **No new files at the root of `src/components/`** — they belong in a folder.

In a component tree this size, silent duplication is the default failure mode: the same
upload widget or combobox gets rewritten in several feature folders because nobody
searched first. **Search `src/components/` before adding one**, and if something close
exists, extend it or lift it to `common/` instead of forking a copy.

### Where non-component code goes

- **`src/lib/api/`** — data-access modules, one per domain.
- **`src/lib/server/`** — server-only helpers. **`src/lib/client/`** — browser-only.
- **`src/lib/supabase/`** — Supabase client factories (browser / server / admin). Use the
  right one; don't construct clients inline.
- **`src/lib/store/`** — Zustand stores.

Prefer an existing subdirectory over a new loose file at the root of `src/lib/`.

### Types and Schemas

- **Never inline type definitions** in component or action files, except for a
  component's own `Props` interface.
- **Put shared types in `src/types/`**: domain models, database types, API response
  types.
- **Put Zod schemas in `src/schema/`**: validation schemas for forms, API inputs, etc.
- **Put DTOs in `src/dtos/`**: request and response shapes for API endpoints.
- **Use `z.infer<typeof Schema>` to derive types** from Zod schemas where possible.

```ts
// ✅ Good: schema in src/schema/, types inferred
// src/schema/project.ts
export const CreateProjectSchema = z.object({
  address: z.string(),
  price: z.number().positive(),
});

export type CreateProjectInput = z.infer<typeof CreateProjectSchema>;

// src/actions/create-project.ts
import { CreateProjectSchema } from '@/schema/project';

export const createProjectAction = action(CreateProjectSchema, async (input) => {
  // input is typed correctly
});
```

```ts
// ❌ Bad: type inlined in action file
// src/actions/create-project.ts
interface CreateProjectInput {
  address: string;
  price: number;
}
export const createProjectAction = action(async (input: CreateProjectInput) => {
  // ...
});
```

### Static Content

- **Never declare static content inline** in components or pages.
- **Put feature lists, navigation items, step copy, etc. in `src/constants/`**.
- **Use named exports** so content is easy to import and discover.

```ts
// ✅ Good: constants in src/constants/
// src/constants/onboarding-steps.ts
export const ONBOARDING_STEPS = [
  { id: 'profile', title: 'Create Profile', description: 'Set up your account' },
  { id: 'verify', title: 'Verify Identity', description: 'Complete ID verification' },
];

// src/components/onboarding/onboarding-flow.tsx
import { ONBOARDING_STEPS } from '@/constants/onboarding-steps';
```

```ts
// ❌ Bad: content inlined in component
// src/components/onboarding/onboarding-flow.tsx
export function OnboardingFlow() {
  const steps = [
    { id: 'profile', title: 'Create Profile', description: '...' },
    // ...
  ];
}
```

### Component Files

- **Keep component files short** (~150–180 lines of JSX/logic). Split large pages into
  section components.
- **Store section components in `src/components/<feature>/`** where `<feature>` matches
  the domain (e.g., `projects/`, `task/`, `messaging/`).
- **No data fetching in leaf components**: pass data as props from the page (Server
  Component).

```
src/components/projects/
├── project-card.tsx       # Single project card (reusable)
├── project-grid.tsx       # Grid of projects
├── project-details.tsx    # Detailed view with actions
└── project-form.tsx       # Create/edit form
```

### Actions and Handlers

- **Server actions go in `src/actions/`**, one file per domain or feature.
- **Name server actions clearly**: `createProject`, `updateTask`, `cancelContract`.
- **Use `next-safe-action`** for type-safe, validated server actions.
- **Webhook/event handlers go in `src/handlers/`** if reused; inline in API route
  otherwise.

### No Premature Abstraction

- **Three similar lines is better than a premature abstraction.**
- **Don't create a helper/component for logic used only once.**
- **Don't extract functions across files unless you're certain they're reused or the
  function is complex.**

If you're unsure whether to extract, keep it inline. Abstractions should be driven by
reuse, not by fear of duplication.
