# Supabase Clients and Server Actions

Picking the wrong Supabase client is a security decision, not a style one. Read this
before writing any data access or server action.

## Choosing a Supabase client

Four factories live in `src/lib/supabase/`. They are **not** interchangeable:

| Factory | Where | Key | RLS |
|---|---|---|---|
| `createSupabaseBrowserClient` (`client.ts`) | Browser | anon | **Enforced** |
| `createSupabaseServerClient` (`server.ts`) | Server | anon | **Enforced** — acts as the signed-in user |
| `createSupabaseAdminClient` (`admin.ts`) | Server only | **service key** | **BYPASSED** |
| `middleware.ts` | Middleware only | anon | Enforced — session refresh |

### Default to the RLS-enforced clients

- **Browser code** → `createSupabaseBrowserClient`.
- **Server code acting for a user** → `createSupabaseServerClient`. RLS scopes the query
  to that user, so the database enforces access even if application code forgets to.

### `createSupabaseAdminClient` bypasses row-level security

It uses `SUPABASE_SERVICE_KEY`, so **every query through it skips RLS entirely**. The
database will not stop a query that reads or writes another user's rows.

- **Never import it into client code.** The service key must not reach the browser.
- **Use it only when the operation genuinely needs to act outside a single user's
  permissions** — cron jobs, webhooks, admin tooling, cross-user reads.
- **When you do use it, you own authorization.** RLS is off, so the action must check
  identity and role itself before touching data.
- **Prefer the server client and let RLS do the work** whenever the operation is
  something the signed-in user should be able to do on their own behalf.

```ts
// ❌ Bad: admin client for a user's own write — RLS off, no ownership check
const supabase = await createSupabaseAdminClient();
await supabase.from('projects').update(input).eq('id', input.id);

// ✅ Good: server client — RLS scopes the update to rows the user may edit
const supabase = await createSupabaseServerClient();
await supabase.from('projects').update(input).eq('id', input.id);
```

## Server actions

**Default to a server action for mutations** unless told otherwise. Reach for an API
route only when something genuinely requires it — an external webhook, a cron target, a
non-browser consumer, or a response that isn't JSON.

Action clients are defined in `src/lib/server/safe-action.ts`. **Always build actions on
one of them** — don't export a bare `async` function from `src/actions/*` and do auth by
hand.

- **`safeActionClient`** — base client. Validation and error shaping, no auth. Use only
  for genuinely public actions.
- **`profileActionClient`** — resolves the auth user *and* their `profiles` row, and
  injects `{ supabase, authUser, profile }` into `ctx`. **Use this for any action taken
  by a signed-in user on their own behalf** — it carries the RLS-enforced server client.
- **`adminActionClient`** — auth check plus the **admin** client. Because it bypasses
  RLS, use it only for genuinely privileged operations, and check the caller's role
  inside the action.

```ts
// ✅ Good: user-scoped action — ctx carries the user, profile, and RLS-enforced client
export const updateProfile = profileActionClient
  .schema(UpdateProfileSchema)
  .action(async ({ parsedInput, ctx }) => {
    const { data, error } = await ctx.supabase
      .from('profiles')
      .update(parsedInput)
      .eq('id', ctx.authUser.user.id)
      .select()
      .single();

    if (error) throw error;
    return data;
  });
```

```ts
// ❌ Bad: bare function — no validation, no typed errors, hand-rolled auth,
// and the admin client silently disables RLS
export const updateProfile = async (input: UpdateProfileFields) => {
  const supabase = await createSupabaseAdminClient();
  const { data: { user } } = await supabase.auth.getUser();
  await supabase.from('profiles').update(input).eq('id', user?.id);
};
```

### Rules

- **Never trust a client-side role check.** Authorization belongs in the action or
  middleware, not in the component that renders the button.
- **Every action takes a Zod schema** via `.schema()` — never read raw untyped input.
  The schema lives in `src/schema/*`, not inline in the action or the component.
- **Check `error` on every Supabase call** and let it throw; see
  [`error-handling.md`](error-handling.md).

## Client-side authenticated queries

`authQuery` (`src/lib/client/auth-query.ts`) wraps browser-side reads: it resolves the
user, optionally enforces a `userType`, validates params with Zod, and hands the handler
`{ user, supabase, params }`.

- **Use `authQuery` for authenticated browser reads** rather than calling
  `createSupabaseBrowserClient` and repeating the auth check.
- **It runs in the browser**, so it is a convenience over RLS — never the only thing
  standing between a user and data they shouldn't see. The database policy is the real
  boundary.

## Layer boundaries

- **`src/lib/api/*`** — data-access modules, one per domain. Built on `authQuery` for
  browser-side reads.
- **`src/controllers/*`** — server-side orchestration over Supabase for a domain, used by
  routes and privileged flows.
- **`src/handlers/*`** — **client-side** wrappers that call API routes over `fetch`, for
  direct use or with React Query. Despite the name, these are not server handlers.
- **`src/actions/*`** — server actions, built on an action client.

Keep each layer in its lane: components call hooks or actions, never Supabase directly.

## Database changes — never run them automatically

Schema changes affect a shared database and are hard or impossible to reverse.

- **Never run `supabase db push`, `db reset`, or any migration command on your own.**
  Prepare the change, explain what it does, and **ask for explicit permission** before
  running it — every time. Prior approval for one migration is not approval for the next.
- **`db reset` destroys data.** Never run it against anything but a local database, and
  only when asked.
- **Write the migration file; don't hand-edit the database.** Schema changes belong in
  version control, not in ad-hoc SQL run against an environment.
- **Regenerate `src/types/supabase.ts` after a schema change** so the types match.
- **Flag destructive migrations explicitly** — dropping a column or table, narrowing a
  type, or adding a `NOT NULL` to an existing column can fail or lose data on a table
  that already has rows.

## Generated database types

- **`src/types/supabase.ts` is generated** from the database schema — **never hand-edit
  it.** Regenerate it when the schema changes; a manual edit is silently wrong the next
  time it's regenerated.
- **Type clients with it** — the factories are already generic over `Database`, so
  queries are typed end to end. Don't cast query results to work around a stale type;
  regenerate instead.
