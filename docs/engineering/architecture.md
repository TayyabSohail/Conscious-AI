# Kanwal: platform architecture

Status: DRAFT · Requirements: [PRD](../product/PRD.md) · Open items: [decisions](../product/decisions.md) · Code rules: [coding standards](./coding-standards/README.md)

Diagrams are plain text, so they render everywhere.

**Contents**
1. Big picture · 2. Inside a cell · 3. Question flow · 4. Answer ladder · 5. LLM providers (BYO key) · 6. PM review loop · 7. Context and data model · 8. Integrations and MCP · 9. Identity and access control · 10. Feature control and customization · 11. Billing and licensing · 12. Privacy · 13. Security · 14. Deployment and hosting · 15. Scaling · 16. Stack and repo

**Glossary:** **Cell** = one customer's private, complete installation. **Owner** = the PM Kanwal covers for. **Asker** = the colleague asking. **Tier** = a step on the answer ladder. **Gate** = an access check. **Control plane** = Bullah Labs' central services (billing, licenses, releases). **It never holds customer content.**

---

## 1. Big picture

Two halves, deliberately separated:

```
 ┌──────────────── BULLAH LABS CONTROL PLANE (ours, central) ────────────────┐
 │  Website & signup · Billing (payment gateway) · License signer ·          │
 │  Release registry (signed images) · Fleet manager (managed cells only)    │
 │  Holds: accounts, plans, seats, invoices, cell IDs, versions, health.     │
 │  NEVER holds: questions, answers, documents, tokens, chats.               │
 └───────────────┬──────────────────────┬──────────────────────┬─────────────┘
     signed license + releases   signed license + releases   deploy/upgrade + license
                 ▼                      ▼                      ▼
 ┌─ Cell: Customer A ──────┐ ┌─ Cell: Customer B ──────┐ ┌─ Cell: Customer C ───────┐
 │ app · DB A · model · key│ │ app · DB B · model · key│ │ app · DB C · model · key │
 │ on THEIR servers        │ │ in THEIR AWS account    │ │ dedicated server WE rent │
 │ (local)                 │ │ (private-cloud)         │ │ for C only (managed)     │
 └─────────────────────────┘ └─────────────────────────┘ └──────────────────────────┘
            ✗ no network path, no shared DB, no shared key between cells
```

- **Isolation level: the whole stack per customer.** Not a shared DB with a `tenant_id`, not a schema per tenant. That holds the privacy promise without asterisks, and the blast radius is one customer.
- **Scale out** = more cells, provisioned automatically. **Scale up** = more replicas or GPUs inside a cell.
- **Same signed images everywhere.** Docker is the packaging; the hosting options are in §14.

## 2. Inside a cell

```
  Asker (Slack / Web)        Owner (Web)        IT (Web)
          │                       │                 │
          ▼                       ▼                 ▼
 ┌──────────────────────── CELL · private network · egress deny-by-default ────────────────────────┐
 │  [Proxy] TLS · WAF rules · rate limit · security headers                                       │
 │     │                                                                                          │
 │  [Channel adapters]  Web SPA ·  Slack (Socket Mode, no inbound port) · Teams (v1)             │
 │     │                                                                                          │
 │  [Core API · Python · stateless · N replicas]                                                  │
 │     ├─ Identity      OIDC login, Slack→IdP user mapping, sessions                              │
 │     ├─ Capabilities  feature control: license ∩ profile ∩ IT ∩ role ∩ Owner (§10, §11)         │
 │     ├─ Policy engine the 3 gates, fail closed, the ONLY place access is decided (§9)           │
 │     ├─ Orchestrator  question state machine + answer ladder (§3, §4)                          │
 │     ├─ Review        answered / approval / unanswered / flagged, verdicts, verified answers (§6)│
 │     ├─ Tool broker   authorizes every connector/MCP call (§8)                                  │
 │     └─ Metrics       text-free events, weekly report                                           │
 │        │              │                    │                       │                           │
 │  [LLM gateway]   [Postgres + pgvector]  [MCP gateway]          [Secrets vault]                 │
 │   local models ·  app · index · chat ·   per-user tokens ·       connector tokens,             │
 │   BYO-key APIs ·  review · audit ·       sandboxed local MCP,    LLM API keys, DB keys         │
 │   budgets         events (encrypted)     egress allow-list       (envelope-encrypted)          │
 │        │                                     │                                                 │
 │  [Model server: Ollama→vLLM, GPU]   [Worker(s)] sync · retention · recap · heads-up · license │
 └────────┼─────────────────────────────────────┼─────────────────────────────────────────────────┘
          │ (only if IT enabled a BYO provider)  │
          ▼                                      ▼
   Anthropic / OpenAI API               Jira · Confluence · Slack · Linear · MCP servers
```

| Component | Decides access? | Notes |
|-----------|-----------------|-------|
| Channel adapters | No | Normalize Slack/web messages into one `Question`. Post replies back to the same thread |
| Policy engine | **Yes, the only one** | Plain typed code. Every allow needs a `PolicyDecision` object only it can create |
| Orchestrator | No | Drives the state machine. Asks policy at every step |
| LLM gateway | No | Picks the provider per tier and project. Enforces budgets, timeouts, and output caps. Meters tokens |
| Model / external LLM | **Never** | Sees only pre-approved text. Has no tools in v0 |
| Tool broker + MCP gateway | No (enforces policy) | Runs calls **as the asker**. Holds tokens. Sandboxes servers |
| Worker | No | Postgres-backed queue (`SKIP LOCKED`). No Redis in v0 |

## 3. Question flow (end to end)

Example: Jonas asks in Slack, *"When is Release 3.4 due, and is anything blocking it?"* Sara (the PM) is in a meeting.

```
 1  Slack event ──► Channel adapter: verify Slack signature/socket · dedupe · map Slack user → IdP user
                    (unmapped or deactivated user → refuse, go to Unanswered)
 2  Create Question{asker, owner, channel, thread_ref}   state = RECEIVED   · rate-limit · size cap
 3  Capabilities: is cover mode on for Sara's plan/seat? license, profile, IT, role ── no → polite refusal
 4  GATE 3 · IT policy: Kanwal not paused · source allowed · asker's group allowed
 5  GATE 2 · Sara's sharing rule for this project ──► off / always_me ─────────────► UNANSWERED (tell Jonas)
 6  Load thread context (previous turns in this thread, within retention)
 7  ANSWER LADDER (§4): Tier 0 → Tier 1 → Tier 2. Each tier:
       a. retrieve candidates (index or connector, filtered to the allowed scope)
       b. GATE 1 · live ACL: can JONAS open each source? (his own token) · error ⇒ STOP
       c. produce an answer (script or model, sources passed as quoted data)
       d. checks: grounded? cites ⊆ permitted? within length? confidence ≥ threshold?
       e. pass → go to 8 · fail → next tier · all fail → UNANSWERED
 8  Pre-send re-check: ACL again (permissions may have changed) · render as plain text
 9  Sara's mode = check_with_me ─► NEEDS_APPROVAL (Sara approves / edits / declines)
10  SEND to Jonas in the same Slack thread: answer + sources + "From Kanwal on Sara's behalf"
    + [👍 Correct] [This is wrong] buttons                                     state = ANSWERED
11  Persist: Q&A + context (encrypted, retention) · audit entry (hash-chained) · event (no text)
12  Sara's review inbox updates in real time (§6). Asker flag → FLAGGED
```

**Question state machine** (every transition is audited):

```
RECEIVED ─► GATED ─► ROUTING ─► DRAFTED ─► CHECKED ─┬─► SENT ─────────────► ANSWERED ─┬─► REVIEWED_CORRECT
   │          │                    │                ├─► NEEDS_APPROVAL ─┬─► SENT       └─► FLAGGED ─► REVIEWED_INCORRECT
   │          │                    │                │                   └─► DECLINED                     │ (correction
   └──────────┴───── any error ────┴───────────────►└─► UNANSWERED ─► PM_REPLIED ─► SENT                 ▼  optionally sent)
                                                                                                    CORRECTED
```

## 4. Answer ladder

Cheapest correct path first. Every tier must pass **the same checks**. Escalation is automatic.

```
            ┌───────────────────────────────────────────────────────────────────────────┐
 Question ─►│ Tier 0  SCRIPT / LOOKUP          0 LLM tokens (except the ~400-token classifier)│
            │  · intent rules: due date, owner, status, count, "where is the doc"        │
            │  · structured query AS ASKER via connector                                 │
            │  · verified answers: a PM-approved answer whose sources are unchanged      │
            │    and still accessible to this asker                                      │
            ├───────────────────────────── fails / no match ─────────────────────────────┤
            │ Tier 1  BASIC MODEL (free: open-weight model in the cell, e.g. 8–9B)       │
            │  · RAG over permitted chunks only                                          │
            ├───────────────────────────── fails checks / complex ───────────────────────┤
            │ Tier 2  HIGH-PERFORMANCE MODEL                                             │
            │  · larger local model, OR the customer's own Claude / GPT key (if allowed),│
            │    OR our hosted model (managed)                                           │
            ├───────────────────────────── fails checks ─────────────────────────────────┤
            │ Tier 3  THE PM                                                             │
            │  · goes to Unanswered with the best draft attached (if any)                │
            └───────────────────────────────────────────────────────────────────────────┘
```

| Check (every tier) | Rule | Fail means |
|--------------------|------|------------|
| Access | Every cited source passed Gate 1 for this asker | Drop the source, retry the tier once, else escalate |
| Grounding | Every factual sentence maps to a cited passage (verifier pass) | Escalate |
| Citation integrity | Cited IDs ⊆ provided IDs, ≥ 1 citation | Escalate |
| Confidence | Score ≥ per-tier threshold (set in week 1 from the 100-question eval) | Escalate |
| Policy on content | Project `local-only` ⇒ Tier 2 can't use external providers | Use local Tier 2 or escalate to the PM |
| Budget | Provider spend cap not exceeded | Next allowed provider, or the PM |
| Output | Length cap, plain text, no URLs except verified sources | Truncate or escalate |

Routing is configurable per project: which tiers are enabled, the thresholds, and the providers per tier.

## 5. LLM providers (local, BYO key, hosted)

```
 Orchestrator ──► LLM gateway ──┬─► Local runtime (Ollama/vLLM) ── open models     [default, always available]
   (tier, project,              ├─► BYO: Anthropic API (Claude) ─ customer's key   [IT opt-in]
    sensitivity)                ├─► BYO: OpenAI API (GPT) ─────── customer's key   [IT opt-in]
                                └─► Bullah-hosted open models (managed cells) ──── [contract opt-in]
```

**Provider registry** (per cell, managed by IT):

| Field | Purpose |
|-------|---------|
| `provider`, `model`, `endpoint`, `region` | What and where. The endpoint is added to the egress allow-list automatically |
| `api_key_ref` | Pointer into the vault. **The key itself is never in the DB, logs, browser, or model prompts** |
| `scope` | Org-wide, or per project. Optionally a PM's own key **if IT allows personal keys** |
| `allowed_tiers` | e.g. only Tier 2 |
| `data_terms` | Recorded provider terms: no training on API data, retention, ZDR agreement yes/no. Shown on "What Kanwal knows" |
| `budget` | Monthly cap + per-request token cap. When exhausted, fall back per routing |
| `status` | Health. On failure: fall back, never fail open |

Rules:
- **Default is local only.** External providers are off until IT enables them. The `local` profile can forbid them outright.
- **Per-project `local-only` flag.** Sensitive projects never leave the cell, whatever the routing says.
- **Disclosure:** answers produced by an external provider carry "AI: \<provider\>" in the PM's view and in the audit. "What Kanwal knows" lists *data leaves your network to: \<provider\>*.
- **Key handling:** keys are written once through the IT console → vault (envelope-encrypted). Only the LLM gateway can use them, in memory. They're rotatable, and revocation is immediate. A key validation call happens at save time.
- **Token metering per provider** goes into `events`. That powers cost-per-answer in the IT console and usage-based billing if we need it (§11).
- **Free API tiers that train on inputs are not allowed** as providers. "Free" in our ladder means open-weight models running in the cell.

## 6. PM review loop

The PM sees everything Kanwal did on their behalf, and corrects it.

```
                       ┌──────────────── PM REVIEW INBOX ────────────────┐
  question states ───► │  ANSWERED      what Kanwal said · tier · model  │──► [Correct] [Incorrect]
                       │                sources · asker · channel        │
                       │  NEEDS APPROVAL drafts waiting ("check with me")│──► [Approve] [Edit] [Decline]
                       │  UNANSWERED    passed to me / failed / no tier  │──► [Reply] → posted to asker's thread
                       │                passed checks · wait timer       │        as "Sara replied"
                       │  FLAGGED       asker said wrong · check flagged │──► [Correct it] → optionally
                       │                · I marked incorrect             │        re-send to asker
                       └─────────────────────────────────────────────────┘
                                 │ every action → audit + event + metrics
                                 ▼
          [Correct] ──► optionally "Promote to verified answer" ──► reusable by Tier 0
          [Incorrect] ─► correction text required ──► stored · wrong-answer rate · tier/threshold tuning
```

| Item | Rule |
|------|------|
| Views | Each question appears in exactly one view, derived from its state (§3). Filters: project, channel, asker, tier, date |
| Real time | Server-sent events push changes. Counts are shown as badges |
| Unanswered SLA | If the PM hasn't replied within N minutes (setting), the asker gets "Sara will get back to you". No re-ping spam |
| PM reply | Posted as *"Sara replied (via Kanwal)"* in the original thread. Stored like any answer. Can be promoted to a verified answer |
| Verified answers | Stored with the source IDs and source versions they relied on. **Auto-invalidated** when a source changes or is deleted. Reused only after Gate 1 for the new asker |
| Flags | From an asker (Slack button or reaction, web button), from automatic checks (low grounding after send, source changed), or from the PM |
| Corrections | Sent to the asker only with the PM's click. Kept for the retention period. Never used to train a model, only as verified answers and for tuning thresholds |
| Bulk | "Mark all correct" for a filtered set, with undo |

## 7. Context and data model

Everything lives in **the cell's own Postgres** (§1). Schemas separate concerns and DB roles.

```
 app      users(idp_sub, slack_id, email, status) · owners · projects · sharing_rules · presets
          connections(user, connector, token_ref) · providers(…api_key_ref) · settings · capabilities_cache
          license(current signed blob, seats, features, expires)
 index    sources · documents(source_id, version, acl_hint, synced_at) · chunks(embedding vector) · sync_cursors
 chat     conversations(owner) · messages            ← encrypted with per-Owner key; off-the-record = never written
 qa       questions(asker, owner, channel, thread_ref, state, project) · answers(tier, provider, model,
          text_enc, source_ids, versions, confidence) · thread_context · approvals · flags
          reviews(verdict, correction_enc, by) · verified_answers(text_enc, source_ids, source_versions, valid)
 audit    entries(seq, ts, actor, action, object_ref, decision, prev_hash, hash)   ← append-only, hash chain
 events   (ts, type, project, route/tier, provider, latency, tokens_in/out, source_count)  ← NO text columns
```

- **Context saved:** the thread context (follow-ups), Q&A, approvals, reviews, corrections, and verified answers. **Not saved:** off-the-record chats, model-server prompts, and secrets in any log.
- **Encryption:** disk/volume + DB at rest with the per-cell key. **Text columns (`*_enc`) with a per-Owner data key** wrapped by the cell key, so DB admins see ciphertext. TLS inside the cell too.
- **Retention:** worker jobs read the retention config generated from feature manifests (§12). Deletes cascade to chunks, embeddings, caches, and verified answers. Backups expire on a stated window.
- **Source mirroring:** index rows carry `source_id`, `version`, and `synced_at`. Deletes and restrictions in the source drop the row at the next sync.
- **DB roles:** `api` (no DDL, no audit update/delete) · `worker` · `retention` (delete only per policy) · `audit_reader` (reads logged) · `migrator` (deploy time only).

## 8. Integrations and MCP

### 8.1 Three kinds of integration

| Kind | What | Examples | Direction |
|------|------|----------|-----------|
| **Channel** | Where questions arrive and replies go | Web, **Slack** (Socket Mode), Teams (v1) | In/out, rate-limited |
| **Source** | Where answers come from | Jira, Confluence, **Slack channels**, Linear, Drive, any PM tool via MCP | Read-only |
| **Action** | Change something outside | Comment, create ticket | v0: none. Later: Owner approval for each exact call |

### 8.2 Adapters behind one interface

```
 Features (cover, recap, heads-up, chat) ──► Canonical PM model ──► Connector interface
   Project · WorkItem · Doc · Message · Person · Milestone · Decision      ├─ Native adapter  (our code: sync + index + live ACL)
                                                                           │    Jira/Confluence DC & Cloud, Slack
                                                                           ├─ MCP adapter     (any MCP server, live query, via MCP gateway)
                                                                           │    Linear, Atlassian Cloud MCP, Asana, Notion, GitHub …
                                                                           └─ Hybrid          (native sync for index + MCP for live calls)
```

```python
class Connector(Protocol):                       # sketch
    id: str; adapter: Literal["native", "mcp", "hybrid"]
    capabilities: set[Capability]                # read.search, read.item, read.acl_live, sync.incremental,
                                                 # sync.deletes, write.comment, write.create_item …
    def search(self, ctx: AskerContext, q: CanonicalQuery) -> list[CanonicalRef]: ...
    def get(self, ctx: AskerContext, ref: CanonicalRef) -> CanonicalObject: ...
    def check_access(self, ctx: AskerContext, refs: list[CanonicalRef]) -> dict[CanonicalRef, bool]: ...
    def sync(self, cursor: Cursor) -> Iterator[Change]: ...                 # native / hybrid
    def propose_action(self, ctx: AskerContext, a: CanonicalAction) -> PendingAction: ...  # never executes
```

`AskerContext` is created only by the policy engine. It carries the identity plus a **token handle**, never the token.

**Per-source access rules:**
- **Jira/Confluence:** the asker's own OAuth or PAT is used for the live check (Confluence DC is a week-1 spike, D-03).
- **Slack as a source:** only the channels IT or the PM selected. Asker must be a member of the channel (checked live). Private channels only if the asker is a member. DMs never.
- **MCP:** the call runs as the asker, so the tool enforces its own permissions.

**Slack as a channel:**
- Uses Socket Mode, so there is no inbound port, which works on-prem.
- Message signatures are verified.
- Replies post in the thread and carry the AI label.
- Feedback buttons use Slack interactivity, with each payload verified.

### 8.3 MCP gateway and tool broker

```
 Orchestrator ──ToolIntent + AskerContext──► TOOL BROKER (core API)
                                              1 registry: server+tool enabled? definition hash matches?
                                              2 policy: subject × tool.call:<class> × target(project/team) × context
                                              3 validate args (JSON schema + pinned constraints, e.g. teamId ∈ shared)
                                              4 class=write → Owner approval of exact args · destructive → off
                                              5 quotas: per user/server/tool, ≤ 8 calls/request
                                                    │ approved call (no token)
                                                    ▼
                                             MCP GATEWAY (separate container)
                                              · session per user × server · token vault (OAuth 2.1+PKCE,
                                                audience-bound, NO passthrough of Kanwal tokens)
                                              · remote servers: Streamable HTTP + TLS, egress allow-list per server
                                              · local servers: stdio in a sandbox (no host mounts, RO FS,
                                                non-root, CPU/mem limits, egress only to its own API)
                                                    │ result
                                                    ▼
                                             BROKER: size cap · schema check · drop out-of-scope ·
                                             map to canonical · tag UNTRUSTED · audit (args hashed)
```

**Tool lifecycle (default deny):**

```
IT adds server (URL or image@digest) → INSPECTED (list tools, hash every definition)
→ CLASSIFIED (read | write | destructive | blocked; unclassified = blocked; or apply a connector-pack preset)
→ ENABLED for groups + projects/teams + mode (read: off/auto · write: off/approve · destructive: off)
→ any definition/version change or new tool ⇒ QUARANTINED until IT reviews the diff
```

- Tools are namespaced `server.tool`, and collisions are rejected (stops shadowing).
- Tool descriptions are untrusted (tool poisoning). They're shown to the model only for enabled, in-scope tools.
- Who picks calls: **v0 plain code only, read tools**. v1: the model may *propose* from a pre-filtered list, and the broker re-authorizes each call as untrusted.
- **Adding a PM tool:** IT adds the MCP server, classifies the tools, sets scopes, and users connect once. A **connector pack** (ours or a partner's) bundles: the pinned server, a classification preset, the canonical mapping, heads-up rules, tests. It's signed, with its own feature key `connector.<tool>`.

## 9. Identity and access control

**Identity:** OIDC with the company IdP (Entra, Okta, Google). There are no local passwords. Slack users are mapped to IdP users (verified email, or Enterprise Grid SSO). SCIM or a login-time check deprovisions users and revokes every token and session.

**Roles:** Owner · Asker · IT admin · Auditor, plus persona packs. Roles come from **server-controlled sources only** (IdP groups or our tables). Never from user-editable metadata.

**The three gates** (every question, every tool call):

```
 GATE 3  IT policy        paused? source/server/tool allowed? asker's group allowed? retention ceilings
 GATE 2  Owner sharing    what × who × how for this project (narrow only; no rule ⇒ always_me)
 GATE 1  Source ACL       live, as the asker, before retrieval AND before send
         ─────────────────────────────────────────────────────────────────────────
         Effective access = source ACL ∩ IT ∩ Owner ∩ tool mode ∩ capabilities (license, profile)
         Error / timeout / unknown ⇒ DENY · refusals never reveal existence
```

**Policy engine:**
- v0: typed plain code with property-based tests ("the result is never wider than the source ACL").
- v1: customer-authored IT policies in **Cedar** (embeddable, offline, default-deny, analyzable).
- Platform invariants we ship (e.g. *write tools always need approval*) are a separate `forbid` set that customers can't edit.

## 10. Feature control and customization

**Every layer can only narrow:**

```
 Build (compiled modules) → LICENSE (plan, seats, add-ons, from billing §11) → Deployment profile
 → IT policy → Role/group → Owner/project settings → User prefs  ⇒  ON / OFF(reason)
```

- One `capabilities.evaluate(feature, subject, ctx)`. Errors mean OFF. The UI reads `GET /me/capabilities` and the **server enforces it again** on every route, job, and tool call. CI fails any route without a feature guard.
- Reason codes (`not_in_plan`, `no_seat`, `profile`, `it_policy`, `role`, `owner_rule`) let the UI show *why* a feature is off and *who* can change it ("Upgrade" for `not_in_plan` goes to the billing portal).
- **Kill switches:** cell (pause), feature, connector, MCP server, tool, LLM provider. Each takes effect immediately.

**Feature registry manifest** (every feature, connector, persona, and surface):

```yaml
key: heads_up.release_risk          kind: feature        version: 1.2.0
depends_on: [core.events, connector.any:read.item]
tool_classes: [read]                risk: low            default: off
profiles: [local, private-cloud, managed]               roles: [owner]
billing: { plan_min: pro, addon: null, metered: false } # §11
settings_schema: ./settings.schema.json
metrics: [blockers_flagged_before_standup]
privacy:                                                # REQUIRED, §12
  data_stored: []
  data_leaves_cell: none
  model_calls: local_only
kill_switch: true
```

**Customization levels** (safest first). None can widen access, skip a gate, skip approval, or skip audit.

| Level | What | Who | When |
|-------|------|-----|------|
| 1 Settings | Thresholds, minutes per event, retention within ceilings, ladder tiers per project | Owner/IT | v0 |
| 2 Rules and policies | Sharing rules; IT policies (Cedar) | Owner/IT | v0 / v1 |
| 3 Templates | Answer and recap templates, copy, language | IT/Owner | v1 |
| 4 Packs | Persona packs (new roles), connector packs (new PM tools) | Us/partners/IT | v1 |
| 5 Workflows | Event-triggered steps; writes always need approval | IT/Owner | v2 |
| 6 Extensions | **Customer-built MCP servers** plugged in through the gateway (outside the trust boundary). No in-process plugins | Customer devs | v2 |

## 11. Billing and licensing

**Principle:** billing lives **only in the control plane**. Cells never talk to the payment gateway, and no card data ever touches our servers. A cell learns what it may do from a **signed license**.

```
 PM / IT buyer ──► Website / in-app "Upgrade"
                      │
                      ▼
 ┌──────────────────── CONTROL PLANE ─────────────────────────────────────────────────────┐
 │  Accounts & orgs ── Plans catalog (seat plans, feature add-ons, usage meters)          │
 │        │                                                                                │
 │  Billing service ◄──► PAYMENT GATEWAY (e.g. Stripe Billing + Tax, or Paddle as MoR)    │
 │        │               hosted checkout & customer portal (PCI scope stays at gateway)   │
 │        │◄── signed webhooks: subscription created/updated/canceled, payment failed      │
 │        ▼                                                                                │
 │  Entitlement service: plan + seats + add-ons + status → entitlement record             │
 │        ▼                                                                                │
 │  License signer (Ed25519, key in HSM/KMS) → license.json {cell_id, plan, seats,         │
 │        features[], limits, status, valid_until, grace_until}                            │
 └────────┬──────────────────────────────┬────────────────────────────────────────────────┘
          │ managed / connected cells:    │ offline (local) cells:
          │ cell pulls license (mTLS)     │ IT downloads license file from portal,
          ▼                               ▼ uploads in IT console
 ┌────────────── CELL ──────────────────────────────────────────────────────────────────┐
 │ License verifier (public key baked in) → Capabilities layer "LICENSE" (§10)           │
 │ Seat manager: IT/Owner assigns seats to PMs · over-limit ⇒ new Owner blocked, not     │
 │   existing ones                                                                       │
 │ Usage reporter (only if metered plan): signed TOTALS ONLY (active seats, answers,     │
 │   tokens per tier) → control plane · offline cells: export a signed usage file        │
 └───────────────────────────────────────────────────────────────────────────────────────┘
```

**Charging models** (all expressed as license fields, so they're mixable):

| Model | Who pays | How it's enforced in the cell |
|-------|----------|-------------------------------|
| **Per seat** | Org, per Owner (PM) seat. Askers are free | `seats` in the license. Owners must hold an assigned seat for Owner features |
| **Per feature (add-on)** | Org, or an individual PM for their own seat | `features[]` per seat or per org, e.g. `inbox`, `connector.linear`, `persona.eng_lead` |
| **Plan tiers** | Org | `plan` (`basic` / `pro` / `enterprise`) maps to a feature set in the registry (`billing.plan_min`) |
| **Usage-metered** (optional) | Org | Totals from the usage reporter (answers or tokens per tier). Text-free, signed |

**Self-serve PM path:** a PM signs up and pays by card for their own seat (hosted checkout). Later, an org admin can **consolidate** PM seats into an org subscription, and the seats move without data changes.

**Lifecycle and failure rules:**
- **Payment failed:** the gateway retries (dunning), the license status becomes `past_due`, and there's a grace period (e.g. 14 days) with a banner for IT and Owners.
- **Expired / canceled:** paid features go read-only (review history stays visible, export always works). **Data is never deleted by billing, and security never fails open.**
- **Downgrade:** features above the plan turn OFF at period end, and their data stays exportable.
- **Seat removed:** that PM's cover mode stops ("Sara's assistant is off, ask Sara directly"). Their data is subject to normal retention and export.
- **Webhooks:** signature verified, idempotent (event ID store), processed through a queue, and reconciled daily against the gateway.
- **Tax and invoicing:** EU VAT via the gateway (Stripe Tax) or a merchant of record (Paddle). Enterprise gets invoice billing, contract-based, and the license is issued manually through the same signer.
- **Privacy:** the control plane stores billing contacts and totals only. Usage reporting is off for non-metered plans and never includes content, names of askers, or projects.

## 12. Privacy (MUST)

Every promise = **mechanism + proof**. No proof, no claim. (Patterns from the "Privacy-First Positioning" page: Obsidian's local-first, own-your-data approach and Dia's plain-language promise. D-17 confirms the wording.)

| # | Promise | Mechanism | Proof |
|---|---------|-----------|-------|
| P1 | Your data stays with you | A cell per customer. `local` is the default | Network diagram, egress script |
| P2 | Nothing leaves unless **you** choose | Egress deny. No telemetry, CDNs, or remote fonts. BYO LLM and remote MCP are IT opt-in and listed | Egress script, "data leaves your network to" page |
| P3 | Never trained on | No training pipeline. Provider `data_terms` recorded. Training-on-input providers are banned | Contract + SBOM + provider registry |
| P4 | You own it, you can take it | Open-format export (JSON/MD/CSV) for every type. Clean-exit uninstall | One-click export, uninstall check |
| P5 | See everything Kanwal knows | Generated from manifest `privacy:` blocks | The page always matches the build |
| P6 | Off the record is real | Never written to DB, logs, or model server disk | Integration test |
| P7 | We can't read it, even when hosting | Per-cell key (BYOK option), per-Owner keys, no standing operator access | Key policy, break-glass audit |
| P8 | No tracking or ranking of people | Text-free events, groups ≥ 5, no per-person analytics | Schema + CI check |
| P9 | A policy you can read in 2 minutes | Generated from the same inventory | CI diff policy ↔ inventory |
| P10 | Billing never sees your work | Billing only in the control plane, totals only, no content | Control-plane schema, usage file inspection |

**Data inventory as code:** every manifest's `privacy:` block (data read/stored, retention, encryption, exportable, deletable_by, leaves_cell, model_calls) **generates**: the "What Kanwal knows" page, the retention job config, export/delete coverage (CI fails on gaps), the policy text, the DPIA annex, and the egress list. **A PR changing a privacy block, stored type, or egress needs a privacy reviewer.**

## 13. Security

Assume the model **will** be fooled, and make that harmless. Untrusted: source content, tool results and descriptions, question text, model output, Slack payloads, and billing webhooks until verified.

| # | Threat | Control | Test |
|---|--------|---------|------|
| T1 | Prompt injection in docs/tickets/messages | No model tools in v0. Sources are quoted data. Citation ⊆ permitted. Plain text | 30 poisoned docs per release |
| T2 | Exfiltration via rendered output | No images or auto-links. Strict CSP | Beacon payloads in the suite |
| T3 | Colleague sees restricted data | 3 gates, double live check | 50-case access test |
| T4 | Stale permissions | Live check at send. Sync carries deletes | Revoke mid-flow test |
| T5 | Exposed model server | Private network only, no host port | Port scan in CI |
| T6 | Stolen connector tokens / **LLM API keys** | Vault, envelope encryption, gateway-only use, rotation, never logged | Secret scan, log-redaction tests |
| T7 | Admin snooping | Per-Owner keys. Audit reads are logged | Admin gets ciphertext |
| T8 | Audit tampering | Append-only + hash chain + verifier job | Tamper test |
| T9 | Abuse / DoS / **cost blow-up on BYO key** | Rate limits, caps, bounded queues, provider budgets | Load + budget tests |
| T10 | Supply chain | Pinned weights, lockfiles, SBOM, signed images | CI gate |
| T11 | Session/auth attacks | OIDC+PKCE, short sessions, CSRF, no local passwords. Roles never from user-editable metadata | Pentest |
| T12 | Cross-tenant leak | Cell per customer | Architecture |
| T13 | Telemetry leak | No third-party scripts. Runtime telemetry off | Egress script |
| T14 | MCP tool poisoning | Default deny, IT review, scoped descriptions | Poisoned-description fixtures |
| T15 | MCP rug pull | Hash pinning, auto-quarantine | Definition-change test |
| T16 | Token passthrough / confused deputy | Per-user audience-bound tokens. No shared accounts for answering | Cross-session test |
| T17 | Tool shadowing | `server.tool` namespacing, collision reject | Registry test |
| T18 | Malicious local MCP server | Sandbox, digest pin, egress only to its API | Pentest checklist |
| T19 | Model-driven writes | Approval of exact args. Destructive off. Re-authorization. Call caps | Bypass tests |
| T20 | Customization widens access | Narrow-only layers. Uneditable invariants. Extensions via MCP only | Policy analysis in CI |
| T21 | **Slack spoofing / identity mismatch** | Signature/socket verification. Slack→IdP mapping. Unmapped means refused | Spoofed-payload tests |
| T22 | **Sensitive data sent to external LLM** | IT opt-in, per-project `local-only`, disclosure, provider allow-list | Routing tests per flag |
| T23 | **Forged license / billing webhook** | Ed25519 license verified in the cell. Webhook signatures + idempotency + reconciliation | Forged-license and replay tests |
| T24 | **Verified answer leaks to a new asker** | Re-run Gate 1 for each asker. Auto-invalidate on source change | Access test cases |

Baseline: SSO, TLS everywhere, encryption at rest, encrypted backups, dependency scanning, egress deny. Before the first paid deployment: external pentest, vulnerability disclosure policy, CRA 24 h reporting runbook. 2027: ISO 27001 + SOC 2 Type II on one control set.

## 14. Deployment and hosting

**Profiles** (same signed images. The profile sets *ceilings* that IT can only narrow):

| | `local` | `private-cloud` | `managed` |
|---|---|---|---|
| Where | Customer servers | Customer AWS/Azure/GCP account | A dedicated cell we run (customer-owned account preferred, D-19) |
| When | v0 | v1 | v2 (pilots possible earlier) |
| Packaging | Docker Compose | Terraform + Helm | Terraform + Helm via the fleet manager |
| DB | Postgres container, encrypted volume | Managed Postgres + pgvector, private endpoint | Per-cell managed Postgres or on-box, per-cell key |
| Models | Local GPU | GPU pool, + BYO API if allowed | Per-cell GPU or "lite cell" with a hosted model, + BYO API if allowed |
| BYO LLM / remote MCP | Off by default (can be hard-disabled) | IT opt-in, allow-listed | IT opt-in, allow-listed |
| License | Offline file | File or pull | Pulled from the control plane |
| Updates | Customer pulls a signed release | Customer pipeline | Canary rollout, agreed windows |

**Where cells run for customers without servers** (each option = separate machine/account, DB, key, backups per customer):

| Option | Per customer | ~ Cost/month | Use for |
|--------|--------------|--------------|---------|
| A. Dedicated GPU server (Hetzner EU) | Whole cell on one box, Postgres on an encrypted disk | ~€214 (GPU incl.) | **Pilots, SMB EU** |
| B. AWS account per customer | VPC, RDS + pgvector, GPU instance, KMS key | ~$600–900 | Enterprise, regulated, US |
| C. DigitalOcean droplet + managed DB | Droplet, DB cluster, VPC; hosted model | ~$50–100 | No-GPU "lite cell" tier |
| D. Supabase project | DB only, no GPU, adds a third party | — | **Not for customer data.** Internal tools only |

Cost figures are rough. Check current list prices.

**Release pipeline:**

```
CI (unit + policy property tests + evals: access 50 · injection 30 · quality 100, on every profile)
→ build images · SBOM · sign (cosign) · pin model digests → release registry
→ install/upgrade verifies signatures → mandatory backup → migrations (tested rollback)
→ smoke tests + egress proof → fail ⇒ automatic rollback
```

**Sizing (to measure):** pilot ≤ 10 PMs = 1× 24 GB GPU, 8 cores, 64 GB RAM, 500 GB NVMe · team ≤ 50 = 48 GB+ GPU host + app host · org 50+ = GPU pool, API autoscaling, read replica.

## 15. Scaling

| Pressure | Signal | Move |
|----------|--------|------|
| > ~5 concurrent askers | First token > 2 s | Ollama → vLLM |
| Many PMs per cell | GPU queue depth | More GPUs. Split tiers across GPUs. Shift Tier 2 to BYO API if allowed |
| API CPU | p95 latency | More API replicas (stateless) |
| Sync backlog | Queue lag | More workers. Per-connector concurrency |
| Index > ~5–10 M chunks | Search latency | HNSW tuning → partitioning → dedicated vector DB only if measured |
| Queue contention | Lock waits | Redis/NATS (only if measured) |
| Many customers | Ops toil | Fleet manager: Terraform per cell, canary upgrades, license push |
| Billing volume | Webhook backlog | Control-plane queue workers (the control plane is a normal multi-tenant SaaS, with **no customer content**) |

Rule: design the seams now, and add infrastructure only when a metric demands it.

## 16. Stack and repo

| Layer | Choice |
|-------|--------|
| Web | React + shadcn + Tailwind + React Query (Next.js static export or Vite, D-07). **No business logic in the web tier** |
| Core API + worker | Python (FastAPI), Pydantic, SQLAlchemy/psycopg, Alembic, pytest + Hypothesis |
| DB | PostgreSQL 16+ with pgvector |
| Models | Ollama → vLLM. Shortlist tested in week 1 (Qwen3.5 9B / Ministral, 27B-class, gpt-oss-20b, BGE-M3 embeddings) |
| Control plane | Separate service + DB (ours). Payment gateway via hosted checkout + webhooks |
| Packaging | Docker Compose · Helm · Terraform · cosign |

```
apps/web/              React UI
apps/api/              policy · orchestrator · ladder · review · broker · capabilities · channels
apps/worker/           same package, worker entrypoint
apps/mcp-gateway/      separate container
packages/connectors/   jira/ confluence/ slack/ mcp/ (+ packs)
packages/personas/     pm/
control-plane/         billing · entitlements · license signer · releases · fleet (separate deploy)
deploy/                compose/ helm/ terraform/ egress-proof/
evals/                 quality-100 · access-50 · injection-30 · ladder thresholds
docs/                  product/PRD.md · product/decisions.md · engineering/architecture.md · coding-standards/
```
