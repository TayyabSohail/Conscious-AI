# CLAUDE.md

Kanwal (working name): an AI assistant for project managers, hosted by us with one private space per company. **Security and correct execution are the top priority, always.**

## Read first
- Product, scope (v0 and v1), architecture, security, and cost: `docs/product/kanwal-v0-master-scope.md`. This is the single source of truth.
- Coding standards: use the `coding-standards` skill (`.claude/skills/coding-standards/`). It points to `docs/engineering/coding-standards/`.

## Hard rules
- Fail closed: any error in an access or policy check means nothing is sent.
- Never guess: an unsupported or judgment question gets the fixed "I don't know, please contact <PM>" reply (code, not model text). Every asker reply carries the accuracy notice. Never phrase output as the PM.
- Never log question or answer text, document content, tokens, or secrets.
- Outbound calls go only to the approved list: Jira Cloud, Confluence Cloud, Google and Microsoft sign-in, Amazon Bedrock (through the private endpoint), and the invitation mail sender. No telemetry or third-party scripts in product code.
- Send to AI only the question and the passages that passed the access checks. Projects marked "keep local" send nothing.
- Sources are read-only. Kanwal writes only replies to the person who asked, and invitation emails.
- Never commit, push, create Linear tickets, or run DB migrations without explicit permission.
- Stack: Next.js web app, Python (FastAPI) backend, PostgreSQL with pgvector on AWS. No Supabase or PostHog for company data. v0 integrates Jira Cloud and Confluence Cloud only; everything else is v1.
