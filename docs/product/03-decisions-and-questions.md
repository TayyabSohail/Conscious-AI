# 03. Decisions and open questions

Status values: `OPEN` → `PROPOSED` → `DECIDED (date, who)`.

## Architecture decisions to brainstorm

### D-07. The current repo stack vs the requirements: `OPEN`, needs a decision before week 1
The repo is a **Next.js + Supabase + PostHog** boilerplate. Three parts of it conflict with the requirements:

| Boilerplate piece | Conflict | Proposal |
|-------------------|----------|----------|
| **Supabase (cloud)** | Data must stay in the customer's cell. Self-hosted Supabase is ~10 containers (Kong, GoTrue, PostgREST, Realtime, Storage…), which hurts "install in < 1 h". Also, PostgREST exposes the DB to the browser, so security would hinge on RLS being perfect everywhere | **Drop it.** Use plain PostgreSQL + pgvector, accessed only by the API |
| **Supabase Auth** | The requirement is OIDC with the company IdP and no local passwords | OIDC directly in the API |
| **PostHog** | Third-party telemetry breaks "nothing leaves the cell" | **Remove it.** Metrics go to our own `events` schema |
| **Next.js server** | The doc says React SPA + Python API. Two backends (Node and Python) means two places where auth logic could drift | Option A: keep Next.js as a **static export** (SPA only, no server code). Option B: switch to Vite + React. Either way, **all logic lives in the Python API** |

Recommendation: keep the UI toolchain (React, shadcn, Tailwind, React Query, Zod, RHF), drop Supabase and PostHog, and put all business and security logic in Python.

### D-08. Cell-per-customer deployment model: `PROPOSED`
See ARCH-0 in [system architecture](../engineering/architecture/01-system-architecture.md). This is the biggest choice. Do we agree that "platform scale" means **many isolated cells plus a fleet control plane (v2)**, not one shared multi-tenant SaaS?

### D-09. What "locked/unlocked features" means: `OPEN`
Which one (or which mix) did you mean?
- (a) **Plan tiers**: the customer buys Basic or Pro, and the license unlocks modules. ARCH-4 in the system architecture covers this.
- (b) **Progressive unlock per user**: features open as the user builds trust, for example cover mode only after the Owner has reviewed N answers.
- (c) **Persona-based**: PM features vs eng-lead features.

ARCH-4 handles (a) and (c). (b) would be a small "readiness" layer on top.

### D-10. Monorepo with Python and TS: `PROPOSED`
One repo, `apps/web` + `apps/api` + `packages/connectors`. pnpm for the web side, uv for Python.

### D-11. No Redis or vector DB in v0: `PROPOSED`
Postgres handles the queue and vectors. Add more only when a metric demands it (ARCH-8 in the system architecture).

### D-12. Policy language for IT policies: `PROPOSED`
Owner sharing rules stay a fixed, simple model. For customer-authored IT policies (v1), **evaluate Cedar**: it's embeddable, works offline, is default-deny, and policies can be analyzed. The alternative is OPA/Rego (more flexible, but harder to analyze, and usually run as a sidecar). In v0 we build policies in plain code behind an interface that Cedar can fill later.

### D-13. MCP adoption order: `OPEN`
Which PM tools go first after Jira/Confluence? Proposal: Linear (official MCP), then Atlassian Cloud remote MCP, then Asana and Notion. Each needs an integrations spike to check OAuth-as-user, scopes, and tool stability.

### D-14. Kanwal as an MCP server (Ask Kanwal from Claude Desktop, IDEs): `OPEN`
Proposal: v2, behind `surface.mcp_server`, off by default, through the same three gates.

### D-15. Allow customer-approved external model endpoints in cloud profiles? `OPEN`
Example: Azure OpenAI or Bedrock in the customer's region. Proposal: allowed only in `private-cloud`/`managed`, off by default, enabled per endpoint by IT, and shown on "data leaves your network to". **Never allowed in `local`.**

### D-16. No pooled multi-tenant tier: `PROPOSED`
Even small customers get a dedicated cell. The cheap option is a "lite cell" with no GPU, using an in-region hosted open model. Pooling would break privacy promise P1.

### D-17. Privacy promises wording: `OPEN`
Confirm the user-facing promises P1–P9 ([06 Privacy](../engineering/architecture/06-privacy-architecture.md)) against the "Examples of Privacy-First Positioning" page (Dia, Obsidian). Each must keep a mechanism and a proof.

## Open questions from the source doc

| ID | Question | Why it matters | Our lean |
|----|----------|----------------|----------|
| D-01 | Private chats: kept 30 days by default, or not at all? | Privacy posture vs recap usefulness | 30 days, with off the record one click away |
| D-02 | Who writes the code, at what rate? | Sets the budget | — |
| D-03 | First pilot on Atlassian **Data Center or Cloud**? | Live ACL check on Confluence DC is unproven. That decides the week-1 spike | Spike both on day 1–2 |
| D-04 | Company registration (EU or US)? | A US entity brings in EU–US transfer rules | — |
| D-05 | GPU for the build: hourly or dedicated? | Cost vs convenience | Hourly L4 until the pilot |
| D-06 | Which baseline do savings claims name? | Honest marketing | Name both. Lead with "one larger open model, no routing" |

## Week-1 spikes (de-risk the architecture first)

1. **Confluence DC live ACL check** as another user. If it fails, the fallback is a small Confluence add-on, or Jira-only cover mode.
2. **Model shortlist eval**: 100 questions, EN and DE, quality bar set before cost.
3. **Egress-deny proof** on Compose. Ollama confirmed private with no cloud models.
4. **Policy engine skeleton** plus the first 10 access test cases. They run in CI from day one.
