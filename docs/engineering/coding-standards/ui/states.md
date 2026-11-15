# Component States Standards

Every component must handle loading, empty, and error states explicitly. Never assume
data is always present.

## Rules

### Loading State

- **Use `Skeleton` component** from shadcn/ui while data is loading.
- **Reuse an existing skeleton** if one already matches the layout you're building. If
  none fits, create a reusable skeleton component rather than inlining a one-off — a
  skeleton should mirror the shape of the content it replaces.
- **Show multiple skeleton placeholders** if rendering a list (match expected item
  count).
- **Pass `isLoading` prop** to components that fetch data.

```tsx
// ✅ Good: explicit loading state
'use client';
import { useQuery } from '@tanstack/react-query';
import { queryKeys } from '@/constants/query-keys';
import { Skeleton } from '@/components/ui/skeleton';
import { ProjectCard } from './project-card';

export function ProjectsGrid() {
  const { data: projects, isLoading } = useQuery({
    queryKey: queryKeys.projects.list({}),
    queryFn: fetchProjects,
  });

  if (isLoading) {
    return (
      <div className="grid gap-4 md:grid-cols-3">
        {Array.from({ length: 6 }).map((_, i) => (
          <Skeleton key={i} className="h-64 w-full" />
        ))}
      </div>
    );
  }

  return (
    <div className="grid gap-4 md:grid-cols-3">
      {projects?.map(project => <ProjectCard key={project.id} project={project} />)}
    </div>
  );
}
```

### Empty State

- **Render an explicit empty state** when there's no data.
- **Provide actionable message**: "No projects yet. Create one to get started."
- **Include a link or button** to create/search.

```tsx
// ✅ Good: explicit empty state
export function ProjectsGrid({ projects }: { projects: Project[] }) {
  if (!projects || projects.length === 0) {
    return (
      <div className="flex flex-col items-center justify-center gap-4 py-12">
        <Building2 className="h-12 w-12 text-muted-foreground" />
        <div className="text-center">
          <h3 className="text-lg font-semibold">No projects found</h3>
          <p className="text-sm text-muted-foreground">
            Try adjusting your filters or create a new project.
          </p>
        </div>
        <Button asChild>
          <Link href="/projects/new">Create Project</Link>
        </Button>
      </div>
    );
  }

  return (
    <div className="grid gap-4 md:grid-cols-3">
      {projects.map(project => <ProjectCard key={project.id} project={project} />)}
    </div>
  );
}
```

### Error State

- **Show an error message** using `toast.error()` or inline error UI.
- **Provide a retry option** if the error is transient.
- **Log the error** for debugging (don't expose sensitive info).

```tsx
// ✅ Good: error handling with retry
'use client';
import { useQuery } from '@tanstack/react-query';
import { toast } from 'sonner';

export function ProjectsGrid() {
  const { data: projects, error, refetch } = useQuery({
    queryKey: queryKeys.projects.list({}),
    queryFn: fetchProjects,
    retry: 1,
  });

  if (error) {
    return (
      <div className="flex flex-col items-center justify-center gap-4 py-12">
        <AlertCircle className="h-12 w-12 text-destructive" />
        <div className="text-center">
          <h3 className="text-lg font-semibold">Failed to load projects</h3>
          <p className="text-sm text-muted-foreground">
            {error.message || 'Something went wrong. Please try again.'}
          </p>
        </div>
        <Button onClick={() => refetch()}>Retry</Button>
      </div>
    );
  }

  return (
    <div className="grid gap-4 md:grid-cols-3">
      {projects?.map(project => <ProjectCard key={project.id} project={project} />)}
    </div>
  );
}
```

### Server Component Errors

For Server Components that fetch data, use error boundaries or explicit error fallback:

```tsx
// src/app/projects/page.tsx (Server Component)
import { fetchProjects } from '@/lib/projects';

export default async function ProjectsPage() {
  try {
    const projects = await fetchProjects();
    return <ProjectsGrid projects={projects} />;
  } catch (error) {
    return (
      <div className="flex items-center justify-center py-12">
        <p className="text-destructive">Failed to load projects. Please try again.</p>
      </div>
    );
  }
}
```

## State Machine

```
Loading → Success (render data)
Loading → Error (show error + retry)

Data loaded → Empty state (no items)
Data loaded → Populated (render items)
Data loaded → Error (retry or dismiss)
```

Always implement all three states before calling work done.
