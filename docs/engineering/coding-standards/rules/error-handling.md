# Error Handling Standards

Failures must be visible. The default failure mode to avoid is a swallowed error that
renders as an empty state — the user sees "no results" when the truth is "the request
failed."

## Rules

### Check the Supabase `error` field on every call

A Supabase query that errored still returns a shaped object with `data: null`. Reading
`data` without checking `error` silently turns a failure into an empty result.

```ts
// ✅ Good: error checked and surfaced
const { data, error } = await supabase.from('projects').select('*');

if (error) {
  throw new Error(`Failed to fetch projects: ${error.message}`);
}

return data;
```

```ts
// ❌ Bad: error ignored — a failed request looks like zero rows
const { data } = await supabase.from('projects').select('*');
return data ?? [];
```

### Let data-access code throw

- **Don't catch an error just to return `null` or `[]`** from `src/lib/api/` or a
  controller — that discards the reason and hides the failure from every caller.
- **Catch at the boundary that can act on it**: a server action that maps it to a
  user-facing message, or a route that maps it to a status code.
- **Add context when rethrowing** — say which operation failed, not just what the driver
  reported.

### Surface errors at the UI boundary

- **Mutations** → `toast.error()` with a clear, human message.
- **Form validation** → `FormMessage` on the offending field.
- **Queries** → an explicit error state with a retry, never a silent empty state.
- **Distinguish empty from failed.** "No results yet" and "we couldn't load this" are
  different messages and must not share a render path.

### Never show a raw error to a user

- **Log the detail** (stack, driver message, IDs) for debugging.
- **Display a clear, plain message** — no stack traces, SQL, or internal identifiers.
- **Don't leak sensitive information** in an error string: no table names, no row
  contents, no auth details.

```tsx
// ✅ Good: detail logged, message shown
onError: (error) => {
  console.error('Failed to create project', error);
  toast.error('Could not create the project. Please try again.');
}
```
