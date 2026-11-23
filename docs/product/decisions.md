# Decisions and open questions

Status: `OPEN` → `PROPOSED` → `DECIDED (date, who)`. Refs point to [PRD](./PRD.md) and [architecture](../engineering/architecture.md) sections.

## Needed before week 1 (Nov 16)

| ID | Question | Lean | Ref |
|----|----------|------|-----|
| D-07 | **Repo stack.** The boilerplate is Next.js + Supabase + PostHog. Supabase and PostHog conflict with "nothing leaves the cell" | `DECIDED (2026-11-16, Tayyab)`: **Python 3.12 + FastAPI backend** on plain Postgres + pgvector. Keep the Next.js/shadcn/Tailwind/React Query UI. **Drop Supabase and PostHog** | Arch §16 |
| D-32 | **Deployment order** | `DECIDED (2026-11-16, Tayyab)`: **cloud first.** v0 = customer's cloud account (`private-cloud`) or a dedicated server we rent (`managed`). On-prem (`local`) in v1+. No cloud-only services in the core | Arch §14, PRD PLT-02 |
| D-03 | First pilot on Atlassian **Data Center or Cloud**? | `DECIDED`: **Data Center.** The live ACL check on DC is still a day 1–2 spike (D-36) | Arch §8 |
| D-39 | **Who is the customer?** | `DECIDED (2026-11-16, Tayyab)`: **the PM**, not the company. Open sub-question: PMs from one company share **one cell with a workspace per PM** (lean: shares sources, sign-in, IT and one GPU) or get **a cell each** (strongest isolation, but ~€214+/month per PM for a GPU) | Arch §1, PRD PLT-01 |
| D-33 | **Network path** from a cloud cell to the pilot's on-prem DC | Connector relay (outbound-only mTLS). VPN if their IT prefers. Confirm with their IT in week 1 | Arch §8.4 |
| D-36 | **Asker ACL check on DC**: service account + permission-check-for-user APIs, or per-user OAuth tokens? | Service account if the spike proves it (users connect nothing). Per-user OAuth as fallback, and always on Cloud | Arch §8.2 |
| D-34 | Does the pilot customer have a GPU server? | Unknown, ask them. Plan to rent a dedicated one | Arch §14 |
| D-35 | **Processor paperwork**: DPA, subprocessor list, TOMs, breach runbook | Required before any real pilot data if we host or have support access. Lawyer review | Arch §14 |
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
| D-24 | Ladder thresholds and verifier method (grounding check) | Set from the week-1 100-question eval. **Starting bar:** 0 unsupported claims sent, 0 access leaks, 100% citation accuracy on sent answers, ≥ 95% DONT_KNOW on unanswerable/judgment questions, ≥ 70% of answerable questions answered correctly, p95 < 10 s | Arch §4 |
| D-37 | When to move Ollama → vLLM | When > ~4–5 concurrent questions sustained, p95 target missed, or multi-GPU needed (likely v1) | Arch §15 |
| D-38 | Third-party model gateways (OpenRouter) | Dev only, synthetic data only. Blocked in production by registry + egress + CI | Arch §5 |
| D-25 | Personal (per-PM) LLM keys allowed, or org keys only? | Org keys by default. Personal keys only if IT allows | Arch §5 |

## Hosting

| ID | Question | Lean | Ref |
|----|----------|------|-----|
| D-18 | First provider for companies without servers | `PROPOSED`: **AWS** (EC2 GPU + Docker Compose, EU region), one AWS account per company. Hetzner dedicated GPU stays the low-cost alternative | Arch §14 |
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
| D-05 | Build GPU: hourly or dedicated? | `PROPOSED`: hourly for development and model tests, dedicated from the first pilot day | Arch §14 |
| D-06 | Which baseline do savings claims name? | Name both. Lead with "one larger open model, no routing" | — |
| D-09 | "Locked/unlocked" meaning | Plan tiers + add-ons + personas (Arch §10–11). Progressive per-user unlock: not planned | Arch §10 |
| D-17 | Privacy promise wording vs the "Privacy-First Positioning" page (Dia, Obsidian) | Confirm P1–P10 | Arch §12 |
| D-31 | **Suggested sources** on "I don't know" replies: links + verbatim excerpts only, or also model-written suggestions labeled "predicted"? | **Verbatim only.** Model-written suggestions are guesswork, which the no-guess rule forbids | Arch §4.1 |
| D-30 | Screens: the design artifact couldn't be read by Claude Code. Export its HTML into the repo so the review inbox matches it exactly | — | Arch §6 |
| — | `user_metadata.user_type` role checks in the boilerplate are user-editable, so they're unsafe for authorization ([auth-query.ts:56](../../src/lib/client/auth-query.ts)) | Replace with server-side roles from the Python API (D-07 now decided) | Arch §9 |

## Week-1 spikes

1. Confluence DC live ACL check as another user (fallback: Jira-only cover mode, or a small add-on).
2. Model shortlist on 100 questions (EN/DE). Set ladder thresholds (D-24).
3. Egress-deny proof on the cloud cell (security groups + host firewall). Ollama private, no cloud models.
6. Network path to the pilot's DC: connector relay prototype or VPN (D-33).
4. Policy engine skeleton + the first 10 access cases in CI.
5. Slack Socket Mode + identity mapping (if D-22 = yes).
