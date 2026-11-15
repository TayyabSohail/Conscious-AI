# Forms Standards

Every form must use React Hook Form + Zod + shadcn `Form` component. No ad-hoc form
handling.

## Rules

- **Every form uses `react-hook-form`** for state and validation.
- **Every form uses Zod** for schema validation.
- **Every form uses the shadcn `Form` component** for consistent styling and error
  display.
- **Validation schemas live in `src/schema/`**, not inlined in the component.

## Example: Create Project Form

```ts
// src/schema/project.ts
import { z } from 'zod';

export const CreateProjectSchema = z.object({
  address: z.string().min(1, 'Address is required'),
  price: z.number().positive('Price must be positive'),
  bedrooms: z.number().int().min(0, 'Bedrooms must be 0 or more'),
  bathrooms: z.number().positive('At least 1 bathroom required'),
  description: z.string().optional(),
});

export type CreateProjectInput = z.infer<typeof CreateProjectSchema>;
```

```ts
// src/actions/create-project.ts
import { action } from 'next-safe-action/execute';
import { CreateProjectSchema } from '@/schema/project';

export const createProject = action(CreateProjectSchema, async (input) => {
  // input is typed and validated
  return await db.projects.create(input);
});
```

```tsx
// src/components/projects/create-project-form.tsx
'use client';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from '@/components/ui/form';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import { Button } from '@/components/ui/button';
import { CreateProjectSchema, CreateProjectInput } from '@/schema/project';
import { useAction } from 'next-safe-action/hooks';
import { createProject } from '@/actions/create-project';

export function CreateProjectForm() {
  const form = useForm<CreateProjectInput>({
    resolver: zodResolver(CreateProjectSchema),
    defaultValues: {
      address: '',
      price: 0,
      bedrooms: 1,
      bathrooms: 1,
      description: '',
    },
  });

  const { execute, isPending } = useAction(createProject, {
    onSuccess: () => {
      toast.success('Project created!');
      // redirect or reset form
    },
    onError: () => {
      toast.error('Failed to create project');
    },
  });

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit((data) => execute(data))} className="space-y-4 max-w-md">
        <FormField
          control={form.control}
          name="address"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Address</FormLabel>
              <FormControl>
                <Input placeholder="123 Main St" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="price"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Price (per month)</FormLabel>
              <FormControl>
                <Input type="number" placeholder="1500" {...field} onChange={(e) => field.onChange(Number(e.target.value))} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="description"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Description (optional)</FormLabel>
              <FormControl>
                <Textarea placeholder="Describe the project..." {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit" disabled={isPending} isLoading={isPending}>
          Create Project
        </Button>
      </form>
    </Form>
  );
}
```

## Form Pattern

The pattern is always:

1. **Schema** in `src/schema/`
2. **Action** in `src/actions/` using `action(Schema, ...)`
3. **Component** using `useForm(zodResolver(Schema))` + shadcn `Form`
4. **Error handling** via `FormMessage` (built into shadcn `Form`)

Never deviate from this pattern.

## Submission Handling

- **Use `useAction` hook** from `next-safe-action/hooks` to call server actions.
- **Handle loading state** with `isPending` or `isLoading`.
- **Show success/error toasts** using `sonner`.
- **Reset form on success** if appropriate (use `form.reset()`).

```tsx
// ✅ Good: complete form submission
const { execute, isPending } = useAction(createProject, {
  onSuccess: (result) => {
    toast.success('Project created!');
    form.reset();
    // or redirect: router.push(`/projects/${result.data?.id}`);
  },
  onError: (error) => {
    toast.error(error.serverError?.message || 'Failed to create');
  },
});
```
