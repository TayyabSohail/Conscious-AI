# React Query Standards

Use `@tanstack/react-query` v5 for all data fetching. Centralize query keys and
invalidation logic.

## Rules

### Query Keys

- **Centralize all query keys in `src/constants/query-keys.ts`**.
- **Use a factory pattern** to avoid hard-coding keys across files.
- **Include all query parameters** in the key so that different parameter combinations
  create different cache entries.

```ts
// ✅ Good: centralized query keys
// src/constants/query-keys.ts
export const queryKeys = {
  projects: {
    all: ['projects'],
    list: (filters: ProjectFilters) => [...queryKeys.projects.all, 'list', filters],
    detail: (id: string) => [...queryKeys.projects.all, 'detail', id],
  },
  tasks: {
    all: ['tasks'],
    projectTasks: (projectId: string) => [...queryKeys.tasks.all, 'project', projectId],
  },
};

// src/hooks/use-projects.ts
import { queryKeys } from '@/constants/query-keys';
import { useQuery } from '@tanstack/react-query';

export function useProjects(filters: ProjectFilters) {
  return useQuery({
    queryKey: queryKeys.projects.list(filters),
    queryFn: () => fetchProjects(filters),
  });
}
```

```ts
// ❌ Bad: hard-coded keys scattered throughout
// src/hooks/use-projects.ts
return useQuery({
  queryKey: ['projects', JSON.stringify(filters)],
  queryFn: () => fetchProjects(filters),
});

// src/hooks/use-tasks.ts
queryClient.invalidateQueries({ queryKey: ['projects'] }); // different string!
```

### Mutations and Invalidation

- **Use `useMutation` for writes** (create, update, delete).
- **Invalidate related query keys after mutations** to refetch stale data.
- **Avoid raw `useCallback` for mutations**: let React Query handle caching and
  lifecycle.

```ts
// ✅ Good: mutation with invalidation
// src/hooks/use-create-project.ts
import { useAction } from 'next-safe-action/hooks';
import { queryKeys } from '@/constants/query-keys';
import { useQueryClient } from '@tanstack/react-query';

export function useCreateProject() {
  const queryClient = useQueryClient();

  return useAction(createProjectAction, {
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.projects.all });
    },
  });
}
```

### `useAction` vs `useMutation`

- **Use `useAction` (from `next-safe-action/hooks`)** for server actions — it's
  type-safe and handles validation automatically.
- **Use `useMutation` only for client-side data mutations** that don't have a server
  action.

## Fetching on the Server

- **Fetch data in Server Components** whenever possible. Pass data as props to Client
  Components.
- **Use `unstable_cache` or fetch in a Server Action** if the data must be dynamic and
  tied to user input.

```tsx
// ✅ Good: fetch in Server Component
// src/app/projects/page.tsx
import { fetchProjects } from '@/lib/projects';

export default async function ProjectsPage() {
  const projects = await fetchProjects();
  return <ProjectsGrid projects={projects} />;
}

// src/components/projects/projects-grid.tsx
'use client';
export function ProjectsGrid({ projects }: { projects: Project[] }) {
  // render projects
}
```

```tsx
// ❌ Bad: fetch in Client Component
// src/components/projects/projects-grid.tsx
'use client';
export function ProjectsGrid() {
  const { data: projects } = useQuery({
    queryKey: ['projects'],
    queryFn: async () => {
      const res = await fetch('/api/projects');
      return res.json();
    },
  });
  // ...
}
```
