# Decisions and open questions

Status: `OPEN` → `PROPOSED` → `DECIDED (date, who)`. Refs point to [PRD](./PRD.md) and [architecture](../engineering/architecture.md) sections.

## Needed before week 1 (Nov 16)

| ID | Question | Lean | Ref |
|----|----------|------|-----|
| D-07 | **Repo stack.** The boilerplate is Next.js + Supabase + PostHog. Supabase and PostHog conflict with "nothing leaves the cell" and the < 1 h install | Keep the React/shadcn/Tailwind/React Query UI. **Drop Supabase and PostHog.** All logic in the Python API on plain Postgres | Arch §16 |
| D-03 | First pilot on Atlassian **Data Center or Cloud**? Live ACL check on Confluence DC is unproven | Spike both on days 1–2 | Arch §8 |
| D-22 | **Slack in v0?** (asker channel + source) | Yes if the pilot customer uses Slack. +~24 h, so re-plan | PRD FR-17 |
| D-23 | **BYO LLM key in v0?** (Claude / GPT) | Gateway interface in v0 with one provider (Anthropic or OpenAI). The rest in v1 | Arch §5 |
| D-02 | Who codes, at what rate? | — | PRD §10 |

## Architecture

| ID | Question | Lean | Ref |
|----|----------|------|-----|
| D-08 | Cell per customer, not shared multi-tenant | `PROPOSED`: yes | Arch §1 |
| D-10 | Monorepo (web + api + worker + gateway + control plane) | `PROPOSED`: yes, pnpm + uv | Arch §16 |
| D-11 | No Redis or vector DB in v0 | `PROPOSED`: yes | Arch §15 |
| D-12 | Policy language for IT policies | Plain code in v0, **Cedar** in v1 | Arch §9 |
| D-13 | MCP adoption order after Jira/Confluence | Linear → Atlassian Cloud MCP → Asana/Notion | Arch §8 |
| D-14 | Kanwal as an MCP server | v2, off by default | PRD FR-26 |
| D-15 | External LLM endpoints per profile | Never forced. `local` can hard-disable. Cloud: IT opt-in | Arch §5, §14 |
| D-16 | No pooled multi-tenant tier | `PROPOSED`: yes. A "lite cell" covers small budgets | Arch §14 |
| D-24 | Ladder thresholds and verifier method (grounding check) | Set from the week-1 100-question eval | Arch §4 |
| D-25 | Personal (per-PM) LLM keys allowed, or org keys only? | Org keys by default. Personal keys only if IT allows | Arch §5 |

## Hosting

| ID | Question | Lean | Ref |
|----|----------|------|-----|
| D-18 | First provider for customers without servers | Hetzner dedicated GPU (EU) | Arch §14 |
| D-19 | Who owns the hosted account? | The customer where possible (keeps us a vendor, not a processor) | Arch §14 |
| D-20 | No-GPU "lite cell" tier | Yes, with an EU-hosted open model, disclosed | Arch §14 |
| D-21 | Supabase's role | Internal tools only. Never customer data | Arch §14 |

## Billing

| ID | Question | Lean | Ref |
|----|----------|------|-----|
| D-26 | Payment gateway: **Stripe** (Billing + Tax; we handle VAT filing) vs **Paddle** (merchant of record, handles global VAT) | Paddle for speed while small. Stripe once we need enterprise invoicing flexibility | Arch §11 |
| D-27 | Pricing model: per seat, per feature add-on, plan tiers, metered, or a mix | **Per Owner seat + plan tiers + a few add-ons.** Askers free. Metering only for hosted model usage | Arch §11 |
| D-28 | Self-serve PM card checkout before org sales? | Yes (PM-led adoption), with later consolidation into an org subscription | Arch §11 |
| D-29 | Grace period and lapsed behavior | 14-day grace, then read-only paid features. Never delete data | Arch §11 |

## Product

| ID | Question | Lean | Ref |
|----|----------|------|-----|
| D-01 | Private chats kept 30 days, or not at all? | 30 days. Off the record is one click | PRD §7 |
| D-04 | Company registration (EU / US)? | — | — |
| D-05 | Build GPU: hourly or dedicated? | Hourly L4 until the pilot | — |
| D-06 | Which baseline do savings claims name? | Name both. Lead with "one larger open model, no routing" | — |
| D-09 | "Locked/unlocked" meaning | Plan tiers + add-ons + personas (Arch §10–11). Progressive per-user unlock: not planned | Arch §10 |
| D-17 | Privacy promise wording vs the "Privacy-First Positioning" page (Dia, Obsidian) | Confirm P1–P10 | Arch §12 |
| D-30 | Screens: the design artifact couldn't be read by Claude Code. Export its HTML into the repo so the review inbox matches it exactly | — | Arch §6 |
| — | `user_metadata.user_type` role checks in the boilerplate are user-editable, so they're unsafe for authorization ([auth-query.ts:56](../../src/lib/client/auth-query.ts)) | Replace when D-07 lands | Arch §9 |

## Week-1 spikes

1. Confluence DC live ACL check as another user (fallback: Jira-only cover mode, or a small add-on).
2. Model shortlist on 100 questions (EN/DE). Set ladder thresholds (D-24).
3. Egress-deny proof on Compose. Ollama private, no cloud models.
4. Policy engine skeleton + the first 10 access cases in CI.
5. Slack Socket Mode + identity mapping (if D-22 = yes).
