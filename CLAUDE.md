# CLAUDE.md

Kanwal (working name): a private, self-hosted assistant for project managers. **Security and correct execution are the top priority, always.**

## Read first
- Product (what and why): `docs/product/`. Requirements have IDs (`FR-`, `SEC-`, …) and decisions have `D-` IDs.
- Engineering (how): `docs/engineering/`, covering architecture, coding standards, and codebase guides.
- Coding standards: use the `coding-standards` skill (`.claude/skills/coding-standards/`). It points to `docs/engineering/coding-standards/`.

## Hard rules
- Fail closed: any error in an access or policy check means nothing is sent.
- Never log question or answer text, document content, tokens, or secrets.
- No outbound calls, telemetry, or third-party scripts in product code.
- Never commit, push, create Linear tickets, or run DB migrations without explicit permission.
- Open decisions (e.g. D-07: Supabase vs plain Postgres + Python API) are not settled. Don't build product features that assume an answer.
