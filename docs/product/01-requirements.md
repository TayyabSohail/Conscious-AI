# 01. Requirements (condensed)

IDs are stable. Use them in Linear tickets, PRs, and tests. Keywords: **MUST** means it blocks the release. **SHOULD** is cut only by explicit decision. **LATER** means after v0.

Release targets: **v0 = Dec 31, 2026, pilot-ready** (264 h). **v1 = +280 h.** **v2 = managed hosting.**

---

## 1. Roles

| Role | Who | Can do |
|------|-----|--------|
| **Owner** (v0: PM) | The person Kanwal covers for | Private chat, rules, sharing, approvals, recap, metrics |
| **Asker** (colleague) | Anyone signed in through company SSO | Ask questions; export or delete their own questions |
| **IT admin** | Customer IT | Connectors, models, retention ceilings, audit log, pause switch. **Cannot read the Owner's private chats** |
| **Auditor** (optional) | DPO or works council | Read-only audit log. Every read is itself logged |

> Design note: we model "Owner" as a generic role, not "PM". That lets other roles (eng leads, designers, account managers) arrive later as new *persona packs* without changing the core. See ARCH-6 in [system architecture](../engineering/architecture/01-system-architecture.md).

## 2. Functional requirements

### v0

| ID | Requirement | Pri | Done when | Metric |
|----|-------------|-----|-----------|--------|
| FR-01 | **Five-minute setup**: connect Jira and Confluence, pick a north star, set ground rules | MUST | A new Owner gets a first answer in < 5 min | Time from install to first answer |
| FR-02 | **Think it through**: a private chat that answers and drafts from the Owner's projects, with sources, streamed, and with an off-the-record mode | MUST | Answers stream with citations. Off the record writes nothing to disk | Drafts accepted; hours back |
| FR-03 | **Cover mode**: colleagues ask while the Owner is busy or away | MUST | A question gets the right result: answer, hold for approval, or pass to the Owner | Questions answered without the Owner; wait time |
| FR-04 | **Ground rules and sharing picker**: per project or space, choose *what* (tickets, pages, decisions, files), *who* (team, group, people, everyone), and *how* (just answer, check with me first, always me, off). Includes presets and "Preview as \<person\>" | MUST | Rules apply instantly, are logged, and can be undone | Share of questions covered; leaks = 0 |
| FR-05 | **Approval queue**: sensitive answers wait for the Owner's yes. The Owner can approve, edit, or decline | MUST | Nothing marked "check with me" is sent without approval | Drafts approved without edits |
| FR-06 | **Recap**: everything Kanwal said, what needs the Owner, and one-click fixes | MUST | 100% of sent answers appear in the recap | Wrong-answer rate |
| FR-07 | **Heads-up**: stuck tickets and clashing dates flagged from Jira by **plain rules** (no LLM) | SHOULD (first to cut) | Stuck tickets get flagged | Blockers flagged before standup |
| FR-08 | **North star and weekly report** | MUST | Events are logged and the weekly report shows the chosen goal, labeled "estimate" | The Owner's own number |
| FR-09 | **Data controls**: off the record, retention, export, delete, "forget this", and "what Kanwal knows" | MUST | Retention jobs run; export, delete, and pause all work | Weekly trust score |
| FR-10 | **IT console**: SSO, connectors, models, audit log, pause switch, and a "can X see Y?" test tool | MUST | IT can run the install without us | Days to IT approval; tokens and cost per answer |

### v1 and later

| ID | Requirement | When |
|----|-------------|------|
| FR-11 | Inbox triage, reply drafts, day plan (M365 and Google). Message bodies never enter the index | v1 |
| FR-12 | Team chat channel: Kanwal answers in Slack or Teams | v1 |
| FR-13 | More sources: SharePoint, OneDrive, Google Drive, **Linear**, Jira Service Management | v1 |
| FR-14 | German interface (i18n is built in from v0) | v1 |
| FR-15 | Offline installer | v1 |
| FR-16 | Managed hosting, one customer per server | v2 |

### Not building, on purpose
Auto-sent email, ticket edits without approval, per-person analytics or rankings, model fine-tuning, meeting transcription, voice, mobile app.

## 3. Platform requirements (from our "platform, not app" goal)

These are not in the Confluence doc. They come from our plan for a platform with locked and unlocked features and future personas. **Up for discussion.**

| ID | Requirement | Pri |
|----|-------------|-----|
| PLT-01 | **Entitlements**: every feature is a module behind a feature key. A key unlocks through a signed license that is verified offline. Enforcement happens in the backend, never only in the UI | MUST (v0 basic) |
| PLT-02 | **Connector framework**: every integration implements one interface (auth, incremental sync, deletes, live access check, rate limits). Read-only by default. Write actions are a separate, approval-gated capability | MUST |
| PLT-03 | **Persona packs**: an Owner role type bundles its default rules, metrics, heads-up rules, and UI copy. PM is the first pack | LATER (design for it in v0) |
| PLT-04 | **One codebase, three deployments**: the same signed artifact runs on-prem, in the customer's cloud, or managed by us | MUST |
| PLT-05 | **Model gateway**: all model calls go through one internal OpenAI-compatible interface, so the runtime and the model are both swappable by a setting | MUST |
| PLT-06 | **MCP connectivity**: any PM tool with an MCP server (remote or local) can be connected by IT without new code. Calls go through an MCP gateway and tool broker, always as the asker | MUST (design in v0, ship in v1) |
| PLT-07 | **Integration access control**: per server, per tool, and per project/team scope, by role and group. Tools are default-deny and classified (read / write / destructive). Writes need Owner approval. Definitions are hash-pinned | MUST |
| PLT-08 | **Deployment profiles**: `local` (on-prem) and cloud (`private-cloud`, `managed`) from the same signed build. A profile sets security ceilings | MUST (`local` v0, cloud v1/v2) |
| PLT-09 | **Layered feature restriction**: build → license → profile → IT → role → Owner → user, where each layer only narrows. Kill switches per feature, connector, server, and tool | MUST (layer interfaces in v0) |
| PLT-10 | **Customization**: settings, rules and policies, templates, packs (persona and connector), workflows, and custom extensions via customer MCP servers. No customization can widen access | LATER (levels 1–2 in v0) |
| PLT-11 | **Privacy-first by architecture**: promises P1–P9 each backed by a mechanism and a proof. A data inventory as code drives "What Kanwal knows", retention, export and delete coverage, and the privacy policy | **MUST MUST** (v0) |

Design: [03 Integrations and MCP](../engineering/architecture/03-integrations-and-mcp.md) · [04 Deployment](../engineering/architecture/04-deployment.md) · [05 Feature control](../engineering/architecture/05-feature-control-and-customization.md) · [06 Privacy-first](../engineering/architecture/06-privacy-architecture.md)

## 4. Non-functional requirements

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-01 | Latency | Clicks < 0.1 s. First token < 2 s with 10 concurrent askers (load test). Anything > 10 s shows progress |
| NFR-02 | Answer quality | 100 real questions (EN and DE) meet the bar set in week 1 |
| NFR-03 | Install | Someone who didn't build it installs it in < 1 h from the guide |
| NFR-04 | Recovery | A backup restores cleanly. RTO ≤ 4 h. Restore is tested before every release |
| NFR-05 | Accessibility | WCAG 2.2 AA, fully keyboard-driven, targets ≥ 24 px |
| NFR-06 | Browsers | Two latest versions of Chrome, Edge, Firefox, Safari |
| NFR-07 | i18n | All copy externalized from day one |
| NFR-08 | Sizing (pilot) | ≤ 10 Owners on 1× 24 GB GPU, 8 cores, 64 GB RAM, 500 GB NVMe |
| NFR-09 | Efficiency | Tokens per answer and share of answers with no model call are logged. Target ≥ 50% fewer tokens than the single-model baseline at equal quality |
| NFR-10 | Updates | Signed releases, installed when the customer chooses. No auto-update |

## 5. Security requirements (detail in [security architecture](../engineering/architecture/02-security-architecture.md))

| ID | Requirement |
|----|-------------|
| SEC-01 | Three gates on every question: **IT policy → Owner sharing → live source ACL as the asker**. Fail closed |
| SEC-02 | Access is checked before retrieval **and again before sending**, in plain code, outside the model |
| SEC-03 | Retrieved content is data, never instructions. It cannot trigger any action |
| SEC-04 | Answers render as plain text: no auto-loaded images, no outside links except citations to sources |
| SEC-05 | A refusal never reveals that a restricted item exists |
| SEC-06 | No local passwords. OIDC with the company IdP only |
| SEC-07 | Model server is on a private network only, behind app auth. Telemetry is off (`OLLAMA_NO_CLOUD=1`, `VLLM_NO_USAGE_STATS=1`) |
| SEC-08 | Egress is denied by default except to the customer's own tools. A shipped script proves it |
| SEC-09 | Read-only, least-privilege connector tokens, stored in a secrets store, rotated |
| SEC-10 | Rate limits per asker, a cap on answer length, a bounded queue |
| SEC-11 | Supply chain: pinned and checksummed model weights (safetensors only), SBOM, dependency scanning, signed releases |
| SEC-12 | Audit log is append-only and tamper-evident. Reading it is logged. Admins cannot read private chats |
| SEC-13 | TLS in transit, encryption at rest, encrypted backups |

**Release gates (Dec 31):** 50-case access test with zero leaks, 30 poisoned docs with zero actions and zero leaks, load test, restore test, install test, accessibility test, quality test.

## 6. Product rules

Retention defaults, sharing logic, metric counting, and voice live in [02-product-rules.md](./02-product-rules.md). The technical design that implements these requirements is in [engineering/architecture](../engineering/architecture/).
