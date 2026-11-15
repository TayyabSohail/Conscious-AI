# Domain Workflows and Permissions

Patterns for modelling roles, permissions, state machines, and side effects. These are
shapes, not specifics — fill them in with your project's own roles and states.

## Roles and permissions

Most apps have several user roles with genuinely different capabilities. Respect the
boundaries between them, and enforce them where they can't be bypassed.

- **Always check permissions server-side.** A client-side role check controls what the UI
  renders, not what the user can do — anyone can call the endpoint directly.
- **Guard protected routes in middleware**, so an unauthenticated or wrong-role request
  never reaches the page.
- **Re-check the role inside the mutation.** Middleware protects the route; it doesn't
  protect a server action invoked from elsewhere.
- **Let the database enforce it too.** Row-level security is the backstop for the case
  where application code forgets — see [`supabase-and-actions.md`](supabase-and-actions.md).

```tsx
// ✅ Good: server-side check decides what to render
export default async function RecordPage({ params }: { params: { id: string } }) {
  const session = await getServerSession();
  const record = await fetchRecord(params.id);

  if (session?.user?.role === 'owner' && session.user.id === record.ownerId) {
    return <OwnerView record={record} />;
  }

  return <PublicView record={record} />;
}
```

```tsx
// ❌ Bad: client-side check only — the data was already sent to the browser
'use client';
export function RecordActions({ record }: { record: Record }) {
  const { user } = useSession();
  if (user?.role === 'owner') return <EditButton />;
  return <ViewOnlyMessage />;
}
```

## State machines

Entities that move through a lifecycle — an order, a request, an application, a
subscription — need their transitions modelled explicitly rather than inferred from
scattered boolean flags.

- **Define the canonical states and the legal transitions between them**, and write them
  down where the team can find them.
- **Never skip states.** Every change goes through a defined transition, so history stays
  reconstructible.
- **Validate the transition server-side, in the action** — not in the component that
  happens to render the button.
- **Reject invalid transitions loudly.** A no-op on an unexpected state hides bugs.
- **Timestamp transitions** (`created_at`, `updated_at`, and an expiry where relevant) so
  the lifecycle is auditable.

```ts
// ✅ Good: the transition is validated before anything is written
export const approveRequest = profileActionClient
  .schema(z.object({ requestId: z.string() }))
  .action(async ({ parsedInput, ctx }) => {
    const request = await fetchRequest(parsedInput.requestId);

    if (request.status !== 'pending') {
      throw new Error(`Cannot approve a request in "${request.status}" state`);
    }

    return updateRequest(request.id, { status: 'approved' });
  });
```

```ts
// ❌ Bad: writes blindly — an already-cancelled request silently becomes approved
export const approveRequest = async (id: string) =>
  updateRequest(id, { status: 'approved' });
```

## Side effects and notifications

Work that isn't part of the user's result — emails, push notifications, webhooks,
analytics — should not sit between the user and their response.

- **Don't block the response on a side effect.** Queue it, or dispatch it without
  awaiting, so a slow third-party API doesn't slow the mutation.
- **A failed notification must not fail the operation.** If the record was written, the
  request succeeded; a bounced email is a separate problem.
- **Dispatch through one module per channel** rather than calling the provider SDK inline
  from actions — one place to add retries, logging, or a queue later.
- **Make handlers idempotent.** Queues and webhooks retry; the same event will arrive
  twice eventually.

```ts
// ✅ Good: the write is what the user waits for
export const approveRequest = profileActionClient
  .schema(ApproveRequestSchema)
  .action(async ({ parsedInput, ctx }) => {
    const request = await updateRequest(parsedInput.requestId, { status: 'approved' });

    queueNotification({
      type: 'request_approved',
      userId: request.submittedBy,
      data: { requestId: request.id },
    });

    return request;
  });
```

## Documenting your domain

Rules that come from the business rather than the code — who may do what, which
transitions are legal, what an entity means — belong in prose your team can read, kept
close to the code and updated with it.

- **Point to it from your project's `CLAUDE.md`** so it's discoverable.
- **Reference it from the action** when a rule is non-obvious — this is one of the few
  cases where a comment earns its place, because the constraint isn't visible in the code.
