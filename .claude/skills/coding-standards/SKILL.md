---
name: coding-standards
description: "Canonical UI and backend coding standards for this repo (Next.js App Router, shadcn/Tailwind, React Query, RHF+Zod, server actions) plus Kanwal security/privacy rules. Use for any coding task: components, forms, tables, data fetching, actions, data access, error handling, styling, file placement."
---

# Coding Standards

Canonical coding standards for this repository (UI and backend). The rules live in `docs/engineering/coding-standards/`, which is the single source of truth. This file only points there.

## Before writing any code

1. Read `docs/engineering/coding-standards/project-standards.md`. These are Kanwal-specific security, privacy, and correctness rules, and they **win over everything below**.
2. Read `docs/engineering/coding-standards/standards.md` (the index, stack baseline, and verification checklist).
3. Read the topic files you are touching under `docs/engineering/coding-standards/rules/` and `docs/engineering/coding-standards/ui/`.
4. For boilerplate-specific APIs (Button, form wrappers, DataTable), check `docs/engineering/codebase/`.

## Priority when guidance conflicts

1. Explicit user instruction in the current chat
2. `CLAUDE.md` and `.cursor/rules/*.mdc`
3. `project-standards.md`
4. `standards.md` with `rules/` and `ui/`
5. `docs/engineering/codebase/*`
6. Generic external skills (`.agents/skills/*`)

## Before calling work done

Run the verification checklist in `standards.md` **and** the security checklist in `project-standards.md`. Never commit, push, or run migrations without explicit permission.
