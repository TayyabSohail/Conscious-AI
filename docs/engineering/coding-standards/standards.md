---
name: coding-standards
description: Canonical full-stack implementation standards for Next.js App Router projects using shadcn/ui, Tailwind, React Query v5, React Hook Form + Zod, and Supabase. Use for any frontend or backend coding task — components, forms, tables, data fetching, server actions, data access, error handling, styling, or file placement. Resolves conflicts between local docs and generic/external skills.
---

# Coding Standards

Apply these standards to every coding task in this repository, frontend and backend
alike. Detailed rules live one-per-topic under `rules/` and `ui/` in this skill
directory — read the relevant file(s) before writing code, not after.

## Priority order when guidance conflicts

1. Explicit user instruction in the current chat
2. `CLAUDE.md` (repo root)
3. This skill and its `rules/` + `ui/` docs
4. Project-specific business logic docs, if the repo has them
5. Generic external skills (`.agents/skills/*`, marketplace skills)

## Stack baseline (already installed — don't introduce alternatives)

- Next.js App Router, TypeScript strict mode, React 19
- `@tanstack/react-query` v5 for data fetching
- `@tanstack/react-table` v8 for all tabular data
- **shadcn/ui + Radix + Tailwind CSS for every UI surface**
- React Hook Form + Zod (`@hookform/resolvers`) for every form
- Supabase for auth/db, `next-safe-action` for server actions
- `sonner` for toasts, `lucide-react` / `react-icons` for icons
- pnpm as package manager
- Zustand for state management — **only where React Query isn't enough**, not as a
  default store for server data

## Rules, by topic

Read the file for whatever you're touching. Split into two folders: `rules/` for
general engineering and server-side rules, `ui/` for frontend/component rules.

### `rules/`

- [`typescript.md`](rules/typescript.md) — strict mode, no `any`, no
  unexplained `as`/`@ts-ignore`, Zod at every boundary
- [`file-placement.md`](rules/file-placement.md) — schemas/types/content
  data never inlined in components or actions, keep component/page files short, no
  premature abstraction
- [`code-organization-patterns.md`](rules/code-organization-patterns.md) —
  **NO types in components** (except Props), **NO helpers in components**, **NO static
  data inlined** — everything belongs in src/lib/, src/types/, src/constants/
- [`react-query.md`](rules/react-query.md) — query key centralization,
  invalidation, `useAction` vs `useMutation`
- [`async-concurrency.md`](rules/async-concurrency.md) — **ALWAYS use
  `Promise.all`** for independent operations, never `await` inside a `for` loop
- [`server-client-components.md`](rules/server-client-components.md) —
  default to Server Components
- [`supabase-and-actions.md`](rules/supabase-and-actions.md) — which Supabase client to
  use and what bypasses RLS, the action clients, migrations, layer boundaries
- [`domain-workflows.md`](rules/domain-workflows.md) — server-side permission checks,
  state machine validation, non-blocking side effects
- [`error-handling.md`](rules/error-handling.md) — check the Supabase `error`
  field, don't swallow failures into empty states, surface errors at the UI boundary
- [`config-and-routes.md`](rules/config-and-routes.md) — no hardcoded
  routes/thresholds/currency formatting

### `ui/`

- [`ui-and-styling.md`](ui/ui-and-styling.md) — match the existing app,
  shadcn + Tailwind are mandatory, responsiveness, semantic tokens, typography, a11y
- [`data-tables.md`](ui/data-tables.md) — always `DataTable`, never a
  hand-rolled `<table>`
- [`forms.md`](ui/forms.md) — React Hook Form + Zod + shadcn `Form`, always
- [`states.md`](ui/states.md) — loading/empty/error states

## Match the existing app

- **Use the app's existing colors, design, text styles, and UI patterns.** Don't invent
  your own unless explicitly asked.
- **Never use `<button>` directly** — always the shared `Button` component, reusing its
  existing variants rather than restyling it.
- **Reuse the existing `Skeleton`**; if a suitable one doesn't exist, create a reusable
  one rather than a one-off.
- **Don't introduce new libraries** when an existing dependency covers the need.
- **Keep it simple.** Don't overengineer or abstract ahead of a second real use case.
- **Follow DRY** — duplicated *logic* is as bad as duplicated components; extract it.

## Styling

- **Tailwind CSS only.** Don't introduce other styling approaches.
- **No manual CSS** — the sole exception is where a template genuinely requires it, such
  as email templates in `src/emails/`, which need inline styles for mail-client support.

## Responsiveness

- **Everything must be responsive.** Every screen and component works on mobile, tablet,
  and desktop — this is a requirement, not a follow-up task.
- **Verify all three breakpoints before calling work done**, not just the width you
  happened to develop at.

## Naming and imports

- **Use meaningful names** for files, functions, variables, hooks, types, schemas, and
  components — the name should say what the thing is or does.
- **Component files are kebab-case** — `user-profile.tsx`, never `UserProfile.tsx`.
- **Hook files are `use-*.ts`** — `use-search-filters.ts`.
- **Hooks are `.ts` unless they render JSX** — only use `.tsx` when the file actually
  returns markup.
- **Always import via the `@/` alias** — `@/components/ui/button`, never a deep relative
  path like `../../../components/ui/button`.

## Where components go

`src/components/` is large. Pick the folder deliberately before writing the file:

- **`ui/`** — shadcn/Radix primitives only (`button`, `card`, `dialog`). Don't put
  app-specific components here.
- **`common/`** — app-wide reusable pieces that aren't primitives (`loader`, `footer`,
  `empty-card`). **This is the canonical shared folder.**
- **`shared/`** — deprecated. Don't add to it; use `common/`.
- **Feature folders** — anything tied to a single feature lives in that feature's folder.
- **No new files at the root of `src/components/`** — they belong in a folder.

### Reuse before you create

In a component tree this size, silent duplicates are the default failure mode — the same
upload widget or combobox gets rewritten in three feature folders because nobody looked
first.

- **Search `src/components/` for an existing component before adding one.**
- If something close exists, extend it or lift it to `common/` — don't fork a copy into
  your feature folder.

## Where non-component code goes

- **`src/lib/api/`** — data-access modules, one per domain.
- **`src/lib/server/`** — server-only helpers (route wrappers, safe-action client).
- **`src/lib/client/`** — browser-only helpers.
- **`src/lib/supabase/`** — Supabase client factories (browser / server / admin). Pick
  the right one; don't construct clients inline.
- **`src/lib/store/`** — Zustand stores.
- **`src/controllers/`** — orchestration over `lib/api` + Supabase for a domain.
- **`src/handlers/`** — domain handlers invoked by actions/routes.
- **`src/dtos/`** — request/response shapes.
- **`src/schema/`** — Zod schemas. **`src/types/`** — shared TS types.
- **`src/constants/`** — static data, enums, query keys. **`src/config/`** — routes,
  thresholds, currency.

Prefer an existing subdirectory over a new loose file at the root of `src/lib/`.

## Async

- **`Promise.all`** when independent operations can run together and any failure should
  fail the whole batch.
- **`Promise.allSettled`** when independent operations should each continue even if one
  fails — don't let one rejected call discard the successful results.
- **Never `await` inside a `for` loop.** Use one of the above.
- **Avoid unnecessary sequential requests** — audit for calls that don't depend on each
  other and parallelize them.

## Git

- **Never commit automatically.** Always ask for permission before creating a commit,
  even when the work is finished and verified.
- **Never run database migrations automatically** — `supabase db push`, `db reset`, or
  any schema command. Prepare the change, explain it, and ask every time. Approval for
  one migration is not approval for the next.

## Error handling

- **Let `lib/api` and controllers throw** — don't swallow a Supabase error and return
  `null`, which turns a failure into a silent empty state.
- **Check the Supabase `error` field on every call** — a query that errored still
  returns a shaped object; ignoring `error` reads as "no results".
- **Surface errors at the UI boundary**: `toast.error` for mutations, `FormMessage` for
  field-level validation, an explicit error state for queries.
- **Never show a raw error string to a user** — log the detail, display a clear message.

## Verification before calling work done

### Code Organization
- [ ] **NO types in component files** — all types live in `src/types/*` or `src/schema/*`
      (only exception: component's own `Props` interface)
- [ ] **NO helper functions in component files** — formatting, calculations, validation
      helpers live in `src/lib/*` (reuse existing files if they exist)
- [ ] **NO static data inlined** — feature lists, step copy, nav items, config live in
      `src/constants/*`
- [ ] DTOs live in `src/dtos/*` for request/response shapes
- [ ] Component/page files are short (~150–180 lines) — split large pages into section
      components

### Naming & Placement
- [ ] Component files kebab-case; hook files `use-*.ts` (`.tsx` only if it renders JSX)
- [ ] All imports use the `@/` alias — no deep relative paths
- [ ] New component is in `ui/` (primitives), `common/` (app-wide), or a feature folder —
      never `shared/`, never the root of `src/components/`
- [ ] **Searched `src/components/` for an existing component before creating a new one**
      — extend or lift to `common/` instead of forking a copy

### Error Handling
- [ ] Supabase `error` field checked on every call — never treated as an empty result
- [ ] Errors propagate out of `lib/api`/controllers rather than being swallowed as `null`
- [ ] User-facing failures surface via `toast.error` / `FormMessage` / an error state,
      with no raw error strings shown to the user

### Async & Performance
- [ ] **ALWAYS `Promise.all` for independent operations** — never `await` in a `for`
      loop (critical: can cause 10x slowdowns)
- [ ] `Promise.allSettled` where independent operations must survive one failing
- [ ] Identify all independent async calls and parallelize them

### Data & Forms
- [ ] Tables use `DataTable`; forms use React Hook Form + Zod + shadcn `Form`
- [ ] Query keys come from `src/constants/query-keys.ts`; affected keys invalidated
      after writes

### UI & States
- [ ] Every UI surface uses shadcn primitives and Tailwind CSS — no ad-hoc HTML/CSS
- [ ] **Responsive on mobile, tablet, and desktop** — all three actually checked
- [ ] Matches the app's existing colors, typography, and UI patterns — nothing invented
- [ ] **No raw `<button>`** — shared `Button` with an existing variant
- [ ] Loading → `Skeleton`/`isLoading`; empty → explicit empty state; error →
      `toast.error`/`FormMessage`
- [ ] Skeletons reuse an existing one, or add a reusable one if none fits
- [ ] Only semantic Tailwind tokens, spacing on 4px grid
- [ ] No manual CSS (email templates excepted)

### Comments
- [ ] **NO long explanatory comments** — code should be self-explanatory through clear
      naming; don't narrate what the next line does or why a change was made
- [ ] A comment is only justified when it states a constraint the code itself can't
      show (e.g. a workaround for a third-party bug, a non-obvious business rule)
- [ ] No commented-out code, no "TODO" left behind without a ticket reference, no
      changelog-style comments ("updated to fix X")

### Security & Standards
- [ ] Role-based permissions checked **server-side** (in middleware and server actions)
      before rendering sensitive UI — never trust a client-side role check
- [ ] **Correct Supabase client** — RLS-enforced (`server`/`browser`) by default; the
      admin client only where the operation truly needs to bypass RLS, with an explicit
      authorization check in the action
- [ ] Mutations are server actions by default — API routes only where genuinely required
- [ ] Server actions built on an action client — not bare exported async functions
- [ ] Every action validates input via `.schema()`, with the Zod schema in `src/schema/*`
- [ ] Generated Supabase types regenerated, not hand-edited
- [ ] No hardcoded routes/thresholds/currency — use `src/config/*` and `src/constants/*`
- [ ] No `any`; no unexplained `@ts-ignore`/`@ts-expect-error`/`as` cast
- [ ] No new dependency added where an existing one would do
- [ ] Names are meaningful across files, functions, variables, hooks, types, schemas
- [ ] `pnpm typecheck` and `pnpm lint` pass — all errors fixed, not suppressed

### Final review pass
Before reporting the work done, re-read the implementation and confirm:
- [ ] Responsiveness verified at all three breakpoints
- [ ] Reusability — nothing duplicated that should have been extracted or reused
- [ ] Types, loading/empty/error states, and lint all clean
- [ ] Code is as simple as the problem allows — no speculative abstraction
- [ ] **No commit created without explicit permission**
- [ ] **No migration or `supabase db push` run without explicit permission**

## Non-goals

- Don't replace the local `Button`/`Form` API with generic shadcn examples that skip
  `isLoading`/`icon`/`iconLeft`.
- Don't force React Query v3/v4 style patterns.
- Don't invent UI components that don't exist in this repo — e.g. don't assume a
  membership/feature-gate component exists unless you actually find one under
  `src/components/*`.
- Don't create new role/permission abstractions without first checking how the project
  already models roles.
