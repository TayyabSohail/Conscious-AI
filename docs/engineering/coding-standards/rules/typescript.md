# TypeScript Standards

All code must use TypeScript strict mode (`strict: true` in `tsconfig.json`).

## Rules

- **No `any`**: Avoid `any` entirely. If you must bypass type checking, use a specific
  type like `unknown` and narrow it.
- **No unexplained casts**: Never use `as Type` or `@ts-ignore` without a comment
  explaining why. If the reason is unclear, refactor to avoid the cast.
- **Zod at every boundary**: All external input (API responses, form submissions, webhook
  payloads, environment variables) must be validated with a Zod schema before use.
- **Infer from usage**: Let TypeScript infer types when it's obvious. Explicit types are
  only needed when:
  - The type is complex or non-obvious
  - The type is shared across multiple files
  - The type is part of a public API

## Examples

```ts
// ✅ Good: Zod validation at API boundary
import { z } from 'zod';

const ProjectSchema = z.object({
  id: z.string().uuid(),
  address: z.string().min(1),
  price: z.number().positive(),
});

type Project = z.infer<typeof ProjectSchema>;

async function getProject(id: string) {
  const res = await fetch(`/api/projects/${id}`);
  const data = await res.json();
  return ProjectSchema.parse(data); // throws if invalid
}

// ✅ Good: Inferred types
const projects = [{ id: '1', address: 'Main St', price: 1000 }];
const first = projects[0]; // type is inferred correctly
```

```ts
// ❌ Bad: any type
async function getProject(id: string) {
  const res = await fetch(`/api/projects/${id}`);
  const data: any = await res.json(); // unsafe!
  return data;
}

// ❌ Bad: unexplained cast
const project = JSON.parse(response) as Project; // why is this safe?

// ❌ Bad: no Zod validation
const project: Project = { id: '1', address: 'Main St', price: -100 }; // invalid data not caught
```
