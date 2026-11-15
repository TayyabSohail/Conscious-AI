# Server and Client Components Standards

Use Next.js App Router Server Components by default. Only use Client Components when
you need interactivity or browser APIs.

## Rules

### Default to Server Components

- **Fetch data in Server Components**.
- **Pass data as props to Client Components**.
- **Server Components are cheaper**: they don't ship JavaScript to the browser.

```tsx
// ✅ Good: Server Component fetching, Client Component rendering
// src/app/projects/page.tsx (Server Component by default)
import { fetchProjects } from '@/lib/projects';
import { ProjectsGrid } from '@/components/projects/projects-grid';

export default async function ProjectsPage() {
  const projects = await fetchProjects();
  return <ProjectsGrid projects={projects} />;
}

// src/components/projects/projects-grid.tsx
'use client'; // only needed for interactivity
import { useState } from 'react';

export function ProjectsGrid({ projects }: { projects: Project[] }) {
  const [sortBy, setSortBy] = useState('price');
  // ...
}
```

### When to use `'use client'`

- **State** (useState, useReducer)
- **Effects** (useEffect, useLayoutEffect)
- **Event handlers** (onClick, onChange, etc.)
- **Browser APIs** (localStorage, sessionStorage, window.*, etc.)
- **Hooks** (useContext, useQuery, useMutation, etc.)

### When NOT to use `'use client'`

- **Fetching data**: do it in a Server Component instead.
- **Accessing secrets** (database credentials, API keys): they must stay on the server.
- **Rendering lists of items**: if possible, do it in a Server Component.

## Example: Projects Page

```tsx
// src/app/projects/page.tsx (Server Component)
import { fetchProjects } from '@/lib/projects';
import { ProjectsPage } from '@/components/projects/projects-page';

export default async function Page() {
  const projects = await fetchProjects();
  return <ProjectsPage projects={projects} />;
}

// src/components/projects/projects-page.tsx (Client Component)
'use client';
import { useState } from 'react';
import { ProjectCard } from './project-card';

interface Props {
  projects: Project[];
}

export function ProjectsPage({ projects }: Props) {
  const [sortBy, setSortBy] = useState('price');

  const sorted = [...projects].sort((a, b) => {
    if (sortBy === 'price') return a.price - b.price;
    if (sortBy === 'date') return new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime();
    return 0;
  });

  return (
    <div>
      <select value={sortBy} onChange={(e) => setSortBy(e.target.value)}>
        <option value="price">Price</option>
        <option value="date">Date</option>
      </select>
      <div className="grid gap-4">
        {sorted.map(project => <ProjectCard key={project.id} project={project} />)}
      </div>
    </div>
  );
}
```
