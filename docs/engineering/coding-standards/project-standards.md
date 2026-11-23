# Project standards (Kanwal-specific)

These rules come from the product's non-negotiables ([product plan](../../product/kanwal-v0-master-scope.md)). Where they conflict with [standards.md](./standards.md), **these win**.

## 1. Known conflicts with the general standards

| General standard says | Project position | Status |
|-----------------------|------------------|--------|
| Supabase for auth and DB, `supabase-and-actions.md` | The architecture proposes plain PostgreSQL + OIDC with the company IdP, and the business logic in a Python API | **D-07 decided (2026-11-16): Python API + plain Postgres, Supabase dropped.** The Supabase rules apply only to existing boilerplate code until it is removed. **Don't build new product features on Supabase-specific APIs** (RLS-as-authz, PostgREST from the browser, Supabase Auth) |
| PostHog is part of the stack | **Forbidden in product code.** Nothing may leave the customer's cell | Remove under D-07 |
| Boilerplate rule: role checks may use Supabase `user_metadata.user_type` (`.cursor/rules/project-conventions.mdc`) | **Unsafe for authorization.** `user_metadata` can be edited by the signed-in user (`auth.updateUser`), so any role read from it can be self-granted. Roles must come from a server-controlled source (IdP groups, `app_metadata`, or our own table) | **Open, needs your OK** to change the boilerplate rule |
| Server actions are the default mutation path | Fine for UI plumbing. **Access, sharing, and policy decisions never live in a server action**. They live in the policy engine | Applies now |

## 2. Security and correctness (MUST, every PR)

**Access and policy**
- [ ] Every access decision goes through the **policy engine**. No ad-hoc `if (user.role === …)` checks for content access.
- [ ] **Fail closed.** Errors, timeouts, `undefined`, or an unknown source answer all mean **deny**. Never use `catch { return true }` or a default-allow fallback.
- [ ] Only one function sends an answer, and it requires a `PolicyDecision` that only the policy engine can create. Never bypass it.
- [ ] **No-guess rule** (architecture §4.1): judgment questions never reach a model for an answer. Any unsupported claim withholds the whole answer. Model `unknown` is final. DONT_KNOW and accuracy-notice text come from i18n templates in code, never from the model.
- [ ] Refusal responses and errors **never reveal** that a restricted item exists. That covers the message, status code, and timing where feasible.
- [ ] Feature gating is checked **server-side** through the entitlement service. The UI "locked" state is cosmetic only.

**Untrusted content**
- [ ] Source content, question text, and model output are **untrusted data**. They never pick a code path, a tool, a URL, or a recipient.
- [ ] Model output renders as **plain text**. No `dangerouslySetInnerHTML`, no markdown images, no auto-linked URLs. Links only to verified source URLs.
- [ ] Every boundary is validated with a Zod schema (TS) or a Pydantic model (Python), including connector responses.

**Privacy**
- [ ] **Never log** question or answer text, document content, tokens, cookies, or secrets. Use the logger's redaction, and add a test when you add a new log field.
- [ ] No outbound network calls except to the configured customer tools. No third-party scripts, analytics, CDNs, or remote fonts at runtime.
- [ ] The metrics `events` table never gets a text column.
- [ ] Every new feature, connector, or pack declares a `privacy:` block in its manifest ([product plan, section 7](../../product/kanwal-v0-master-scope.md)). New egress or a new stored data type needs a privacy reviewer.
- [ ] Every new stored data type has a **retention rule** and is covered by export and delete.

**Secrets**
- [ ] Secrets come only from the env or the secrets store, validated in `src/env.ts` (or the Python settings). Never use `NEXT_PUBLIC_*` for anything sensitive.
- [ ] Connector tokens are read-only scopes, encrypted at rest, and never returned to the browser.

## 3. Testing rules

- [ ] Policy engine changes ship with tests for **allow, deny, and error** paths. The target is near-100% branch coverage.
- [ ] Any change to policy, the orchestrator, connectors, or answer rendering must keep the **access (50), injection (30), and quality (100)** eval suites green in CI.
- [ ] Bug fixes in security paths add a regression test first.
- [ ] Migrations are reviewed for data loss and must be reversible. They are never run automatically (same as the general standards).

## 4. Product-driven UI rules

- [ ] **No hardcoded user-facing copy.** All strings go through i18n keys (German ships in v1).
- [ ] **WCAG 2.2 AA:** fully keyboard-operable, visible focus, targets ≥ 24 px, contrast checked.
- [ ] Every answer shows its **sources** and the **AI label**.
- [ ] Long operations (> 10 s) show progress and let the user move on. Answers stream.
- [ ] Destructive actions say exactly what happens ("This deletes 30 days of chat history. It can't be undone."). No jokes in error, permission, delete, or security copy.

## 5. Backend (Python) standards

**D-07 decided: Python API.** To be written: this section will define the layout, typing (strict mypy/pyright), Pydantic boundaries, FastAPI dependency-injected auth, the DB access layer, migrations (Alembic), tests (pytest + property-based tests for policy), and tooling (uv, ruff). The security rules above apply in either case.

## 6. Review

PRs touching `policy`, `orchestrator`, `connectors`, answer rendering, auth, or retention need a second reviewer, plus the security checklist above copied into the PR description.
