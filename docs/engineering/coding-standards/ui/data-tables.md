# Data Tables Standards

Always use `DataTable` from `src/components/ui/data-table/*`. Never hand-roll a
`<table>` element.

## Rules

- **Every table must use `DataTable`** from `@tanstack/react-table` v8.
- **Columns are defined outside the component** in a `columns.tsx` file.
- **Sorting, filtering, pagination are handled by `DataTable`**.
- **No custom `<table>` implementations** — reuse `DataTable`.

## Example: Projects Table

```tsx
// src/components/projects/projects-columns.tsx
import { ColumnDef } from '@tanstack/react-table';
import { Project } from '@/types/project';

export const projectsColumns: ColumnDef<Project>[] = [
  {
    accessorKey: 'address',
    header: 'Address',
  },
  {
    accessorKey: 'price',
    header: 'Price',
    cell: ({ row }) => formatCurrency(row.original.price),
  },
  {
    accessorKey: 'createdAt',
    header: 'Listed',
    cell: ({ row }) => format(new Date(row.original.createdAt), 'MMM d, yyyy'),
  },
  {
    id: 'actions',
    cell: ({ row }) => (
      <DropdownMenu>
        <DropdownMenuTrigger asChild>
          <Button variant="ghost" size="sm">
            <MoreHorizontal className="h-4 w-4" />
          </Button>
        </DropdownMenuTrigger>
        <DropdownMenuContent>
          <DropdownMenuItem asChild>
            <Link href={`/projects/${row.original.id}`}>View</Link>
          </DropdownMenuItem>
          <DropdownMenuItem>Edit</DropdownMenuItem>
          <DropdownMenuItem>Delete</DropdownMenuItem>
        </DropdownMenuContent>
      </DropdownMenu>
    ),
  },
];

// src/components/projects/projects-table.tsx
'use client';
import { DataTable } from '@/components/ui/data-table';
import { projectsColumns } from './projects-columns';

interface Props {
  projects: Project[];
  isLoading?: boolean;
}

export function ProjectsTable({ projects, isLoading }: Props) {
  return (
    <DataTable
      columns={projectsColumns}
      data={projects}
      isLoading={isLoading}
    />
  );
}
```

## DataTable Features

The existing `DataTable` component should support:

- **Sorting**: click column headers
- **Pagination**: next/previous, page size selection
- **Row selection**: checkboxes (optional)
- **Filtering**: basic text search or column-specific filters
- **Loading state**: skeleton rows while data loads
- **Empty state**: message when no data

Refer to `src/components/ui/data-table/` for the implementation. If a feature is
missing, add it to the existing `DataTable` component rather than creating a new
component.
