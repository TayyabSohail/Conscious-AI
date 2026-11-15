# Config and Routes Standards

No hardcoded values in component or action code. Extract routes, thresholds, currency
formatting, and other config to centralized files.

## Rules

### Environment Variables

- **All runtime config goes in `src/env.ts`** (using `@t3-oss/env-nextjs`).
- **Access via `env` object, not `process.env`**.
- **Validate at startup** — misconfigured env is caught immediately, not at runtime.

```ts
// ✅ Good: env.ts with validation
// src/env.ts
import { createEnv } from '@t3-oss/env-nextjs';
import { z } from 'zod';

export const env = createEnv({
  server: {
    DATABASE_URL: z.string().url(),
    STRIPE_SECRET_KEY: z.string(),
  },
  client: {
    NEXT_PUBLIC_ANALYTICS_TOKEN: z.string(),
  },
  runtimeEnv: {
    DATABASE_URL: process.env.DATABASE_URL,
    STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY,
    NEXT_PUBLIC_ANALYTICS_TOKEN: process.env.NEXT_PUBLIC_ANALYTICS_TOKEN,
  },
});

// src/actions/create-project.ts
import { env } from '@/env';

export async function createProject(data: any) {
  // env is validated; safe to use
  const stripe = new Stripe(env.STRIPE_SECRET_KEY);
}
```

```ts
// ❌ Bad: direct process.env access
export async function createProject(data: any) {
  const key = process.env.STRIPE_SECRET_KEY; // not validated, may be undefined
  const stripe = new Stripe(key!);
}
```

### Routes

- **Don't hardcode route paths in components.**
- **Define routes in `src/constants/routes.ts` or use URL building utilities.**

```ts
// ✅ Good: centralized routes
// src/constants/routes.ts
export const ROUTES = {
  HOME: '/',
  PROJECTS: '/projects',
  PROJECT_DETAIL: (id: string) => `/projects/${id}`,
  TASK: (projectId: string) => `/task/${projectId}`,
  ADMIN_DASHBOARD: '/admin/dashboard',
} as const;

// src/components/project-card.tsx
import { ROUTES } from '@/constants/routes';
import Link from 'next/link';

export function ProjectCard({ project }: { project: Project }) {
  return (
    <Link href={ROUTES.PROJECT_DETAIL(project.id)}>
      {project.address}
    </Link>
  );
}
```

```tsx
// ❌ Bad: hardcoded routes
export function ProjectCard({ project }: { project: Project }) {
  return (
    <Link href={`/projects/${project.id}`}>
      {project.address}
    </Link>
  );
}
```

### Configuration Constants

- **Thresholds, limits, and business rules go in `src/constants/`** or `src/config/`.
- **Currency formatting, number formatting, dates go in `src/lib/formatting`** or similar
  utilities.

```ts
// ✅ Good: config extracted
// src/config/task.ts
export const TASK_CONFIG = {
  MIN_TASK_INCREMENT: 100, // cents
  MAX_CONCURRENT_TASKS: 5,
  TASK_TIMEOUT_HOURS: 24,
} as const;

// src/lib/formatting.ts
export function formatCurrency(cents: number, currency = 'USD'): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency,
  }).format(cents / 100);
}

// src/components/task-form.tsx
import { TASK_CONFIG } from '@/config/task';
import { formatCurrency } from '@/lib/formatting';

export function TaskForm({ projectId }: { projectId: string }) {
  const minTask = TASK_CONFIG.MIN_TASK_INCREMENT;
  return (
    <input
      type="number"
      placeholder={formatCurrency(minTask)}
      min={minTask}
    />
  );
}
```

```tsx
// ❌ Bad: hardcoded values
export function TaskForm({ projectId }: { projectId: string }) {
  return (
    <input
      type="number"
      placeholder="$1.00"
      min={100} // magic number
    />
  );
}
```
