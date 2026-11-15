# Coding standards

Two layers:

1. **[standards.md](./standards.md)** plus [rules/](./rules/) and [ui/](./ui/): the team's general full-stack standards, imported verbatim from [TayyabSohail/Coding-Standards](https://github.com/TayyabSohail/Coding-Standards). Don't edit these files here. Change them upstream and re-sync (see below).
2. **[project-standards.md](./project-standards.md)**: rules specific to Kanwal. Security, privacy, and correctness rules **override** the general standards wherever they conflict.

## Priority when guidance conflicts

1. Explicit user instruction in the current chat
2. [`CLAUDE.md`](../../../CLAUDE.md) and `.cursor/rules/*.mdc`
3. **[project-standards.md](./project-standards.md)** (a security rule here can never be overridden by anything below)
4. [standards.md](./standards.md) with `rules/` and `ui/`
5. [Codebase guides](../codebase/README.md) (boilerplate-specific APIs: `Button`, forms, tables)
6. Generic external skills (`.agents/skills/*`)

## Where the agents read this

| Tool | Entry point | Points to |
|------|-------------|-----------|
| Claude Code | [`.claude/skills/coding-standards/SKILL.md`](../../../.claude/skills/coding-standards/SKILL.md) | This folder |
| Cursor | [`.cursor/skills/coding-standards/SKILL.md`](../../../.cursor/skills/coding-standards/SKILL.md) | This folder |

Those skill files are thin pointers. **This folder is the only copy of the rules**, so they can't drift apart.

## Upstream sync

| Field | Value |
|-------|-------|
| Source | `github.com/TayyabSohail/Coding-Standards`, `skills/coding-standards/` |
| Imported commit | `03b6019f8f363c0da87a0365badf2d0b5c5a7846` |
| Imported on | 2026-11-15 |
| Mapping | `SKILL.md` → `standards.md`; `rules/`, `ui/` copied as-is |

To re-sync: clone the upstream repo, copy `skills/coding-standards/{rules,ui}` over this folder, copy `SKILL.md` to `standards.md`, then update the commit hash above. Review the diff against `project-standards.md` for new conflicts.
