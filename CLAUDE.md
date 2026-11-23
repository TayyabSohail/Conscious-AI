# CLAUDE.md

Kanwal (working name): a private, self-hosted assistant for project managers. **Security and correct execution are the top priority, always.**

## Read first
- Product: `docs/product/PRD.md` (IDs `FR-`, `PLT-`, `NFR-`, `SEC-`) and `docs/product/decisions.md` (`D-` IDs).
- Architecture: `docs/engineering/architecture.md`.
- Coding standards: use the `coding-standards` skill (`.claude/skills/coding-standards/`). It points to `docs/engineering/coding-standards/`.

## Hard rules
- Fail closed: any error in an access or policy check means nothing is sent.
- Never guess: an unsupported or judgment question gets the fixed "I don't know, please contact <PM>" reply (code, not model text). Every asker reply carries the accuracy notice. Never phrase output as the PM.
- Never log question or answer text, document content, tokens, or secrets.
- No outbound calls, telemetry, or third-party scripts in product code.
- Never commit, push, create Linear tickets, or run DB migrations without explicit permission.
- Decided: Python (FastAPI) backend on plain Postgres, no Supabase or PostHog (D-07); cloud-first deployment, on-prem in v1+ (D-32). Open decisions (e.g. D-33 relay vs VPN, D-36 DC ACL check) are not settled. Don't build product features that assume an answer.
