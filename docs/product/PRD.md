# Kanwal: product requirements (PRD)

Status: DRAFT · Source: Confluence "Conscious AI for PMs - Design Requirements" (Nov 13, 2026) + team decisions · Architecture: [architecture.md](../engineering/architecture.md) · Open items: [decisions.md](./decisions.md)

IDs are stable. Use them in tickets, PRs, and tests. **MUST** blocks release · **SHOULD** is cut only by decision · **LATER** is after v0.
Releases: **v0 = Dec 31, 2026 (pilot-ready, 264 h)** · **v1 = +280 h** · **v2 = managed hosting**.

---

## 1. What it is

A private assistant that **answers colleagues' questions in place of the PM**. It gathers the answer from the PM's tools (Jira, Confluence, Slack, Linear …), replies to the person who asked, saves the context, and lets the PM **see, correct, and approve** everything it said. It also flags blockers, and from v1 sorts the inbox. Security and correct execution are the top priority, always.

## 2. Principles (priority order)

1. **Security and correct execution first.** Any failed or erroring check means nothing is sent.
2. **Privacy-first, provably.** Every privacy promise has a mechanism and a proof. If we can't prove it, we don't claim it.
3. **The PM is the hero.** Kanwal drafts and covers. The PM decides, approves, corrects, and can undo.
4. **Private by default.** A fresh install stores the minimum, calls nothing outside, and trains nothing.
5. **Permissions are inherited, never invented.** Kanwal mirrors company access. The PM can only narrow it.
6. **Show your work.** Every answer has sources. Every action is logged and visible to the PM.
7. **Cheapest path that's correct.** Script → basic model → high-performance model → the PM.
8. **When in doubt, do less.** Unclear sources mean the question goes to the PM.
9. **Measure before you add.** Every feature ships with its metric.

## 3. Roles

| Role | Who | Can |
|------|-----|-----|
| **Owner** (v0: PM) | Person Kanwal covers for | Private chat, rules, sharing, approvals, review (mark correct/incorrect), metrics |
| **Buyer** | PM (self-serve card) or org admin | Buy seats and add-ons, manage billing in the hosted portal. Lives in the control plane, not the cell |
| **Asker** | Any colleague signed in via company SSO, in the web app or Slack | Ask, flag an answer as wrong, export or delete own questions |
| **IT admin** | Customer IT | Connectors, LLM providers and keys, retention ceilings, audit, pause. **Cannot read Owner private chats** |
| **Auditor** | DPO / works council | Read-only audit. Reading is itself logged |

"Owner" is generic, so other roles (eng lead, designer …) arrive later as **persona packs** without core changes.

## 4. Core loop

```
Colleague asks (Slack / web)  →  Kanwal checks access  →  gathers from Jira/Confluence/Slack…
→ answers via the cheapest tier that passes checks (script → basic model → strong model)
→ or hands it to the PM  →  reply goes back to the asker, with sources and the AI label
→ everything is saved (within retention)  →  PM reviews: answered / needs approval / unanswered / flagged
→ PM marks correct or incorrect, corrects it, and Kanwal learns a verified answer
```

## 5. Functional requirements

### v0

| ID | Requirement | Pri | Done when | Metric |
|----|-------------|-----|-----------|--------|
| FR-01 | **Five-minute setup**: connect sources, pick a north star, set ground rules | MUST | First answer in < 5 min | Install → first answer |
| FR-02 | **Think it through**: private chat for the PM, with sources, streamed, with an off-the-record mode | MUST | Streams with citations. Off the record writes nothing | Drafts accepted |
| FR-03 | **Cover mode**: colleagues ask while the PM is busy or away. Kanwal replies **in the channel they asked in** | MUST | Correct outcome: answer / hold / pass to PM | Answered without PM; wait |
| FR-04 | **Answer ladder**: Tier 0 script/lookup → Tier 1 basic model → Tier 2 high-performance model → Tier 3 PM. Escalates automatically when a tier fails its checks | MUST | Every answer records its tier. Failed checks escalate, never send | % per tier; tokens/answer |
| FR-05 | **LLM provider choice**: built-in local open models (default), **the customer's own API key** (Anthropic Claude, OpenAI GPT), or our hosted models (managed). Set by IT, optionally per project | MUST | Provider switch with no code change. External use is visible to the PM and IT | Cost/answer per provider |
| FR-06 | **Ground rules and sharing picker**: per project or space, *what* (tickets, pages, decisions, files, Slack channels), *who* (team / group / people / everyone), *how* (just answer · check with me · always me · off). Presets and "Preview as \<person\>" | MUST | Instant, logged, undoable | Coverage; leaks = 0 |
| FR-07 | **Approval queue**: "check with me" drafts wait for the PM: approve / edit / decline | MUST | Nothing sent without approval | Approved without edits |
| FR-08 | **PM review inbox** with four views: **Answered** (what Kanwal said, tier, sources), **Needs approval**, **Unanswered** (passed to the PM or failed; the PM replies inline and the reply goes to the asker's thread), **Flagged** (an asker said it's wrong, or a check flagged it, or the PM marked it incorrect) | MUST | Every question appears in exactly one view and moves between views by state | Time-to-review |
| FR-09 | **Mark correct / incorrect**: on any answer. Incorrect requires a correction, optionally sent to the asker. Correct can be **promoted to a verified answer** that Tier 0 can reuse | MUST | Verdict stored and reflected in metrics. Corrections stick | Wrong-answer rate |
| FR-10 | **Asker feedback**: "This is wrong" (Slack button or reaction, web button) flags the answer to the PM | MUST | The flag appears in Flagged within seconds | Flags/answers |
| FR-11 | **Context memory**: the thread context (follow-ups in the same thread), Q&A history, corrections, and verified answers are stored in the cell DB under retention and encryption rules | MUST | Follow-ups resolve within a thread. Retention deletes on time | Follow-up success |
| FR-12 | **Recap**: daily/weekly summary of everything said, what needs the PM, one-click fixes | MUST | 100% of sent answers listed | Wrong-answer rate |
| FR-13 | **Heads-up**: stuck tickets and clashing dates from Jira, by **plain rules** | SHOULD (first to cut) | Stuck tickets flagged | Blockers before standup |
| FR-14 | **North star and weekly report**: one PM-chosen goal, evidence, always labeled "estimate" | MUST | The weekly report shows the goal and its guardrails | The PM's number |
| FR-15 | **Data controls**: off the record, retention, export, delete, "forget this", "what Kanwal knows" | MUST | All work end to end | Trust score |
| FR-16 | **IT console**: SSO, connectors, MCP servers and tools, LLM providers and keys, feature switches, audit log, pause, a "can X see Y?" tester | MUST | IT runs the install unaided | Days to IT approval |
| FR-17 | **Slack** as channel *and* source (Socket Mode). Asker identity is mapped to the IdP. Only selected channels are indexed | MUST if pilot uses Slack (D-22), else v1 | Ask in Slack → reply in thread | Questions via Slack |

### v1 and later

| ID | Requirement | When |
|----|-------------|------|
| FR-20 | Inbox triage, reply drafts, day plan (M365 / Google). Bodies are never indexed | v1 |
| FR-21 | Teams channel | v1 |
| FR-22 | More sources: SharePoint, OneDrive, Google Drive, **Linear**, JSM, and any PM tool via **MCP** | v1 |
| FR-23 | German UI (i18n from v0) · offline installer | v1 |
| FR-24 | Managed hosting (one dedicated cell per customer) | v2 |
| FR-25 | Workflows ("release at risk → draft note → PM approves") · customer extensions via their own MCP servers | v2 |
| FR-26 | Kanwal exposed as an MCP server ("Ask Kanwal" from Claude Desktop / IDEs), off by default | v2 |
| FR-27 | **Billing**: charge per Owner (PM) seat and/or per feature add-on and plan tier, via a payment gateway (hosted checkout, customer portal, invoices, EU VAT). A PM can self-serve with a card, and orgs can consolidate seats. Delivered to cells as a signed license; lapses mean grace, then read-only, never data loss | v1 (license layer v0) |

**Not building, on purpose:** auto-sent email, ticket edits without approval, per-person analytics or rankings, fine-tuning, meeting transcription, voice, mobile app.

## 6. Platform requirements

| ID | Requirement | Pri |
|----|-------------|-----|
| PLT-01 | **Cell per customer**: own app, DB, model, keys, backups. Nothing shared between customers | MUST |
| PLT-02 | **One signed build, deployment profiles**: `local` (on-prem), `private-cloud` (customer's AWS/Azure/GCP), `managed` (our dedicated cell) | MUST (`local` v0) |
| PLT-03 | **Connector framework**: native, MCP, or hybrid adapters behind one interface, mapped to a canonical PM model | MUST |
| PLT-04 | **MCP connectivity**: IT adds any MCP server without code. Every tool starts blocked and is classified read / write / destructive. Definitions are hash-pinned | MUST (design v0, ship v1) |
| PLT-05 | **Per-call authorization**: server, tool, and target project/team × role/group × Owner rule × profile × license | MUST |
| PLT-06 | **Layered feature control**: build → license → profile → IT → role → Owner → user. Each layer only narrows. Kill switches everywhere | MUST |
| PLT-07 | **LLM gateway**: one internal interface for local, BYO-key, and hosted providers, with routing, budgets, and token metering | MUST |
| PLT-08 | **Customization**: settings → rules/policies → templates → packs → workflows → extensions (via MCP). None can widen access | Levels 1–2 v0 |
| PLT-09 | **Privacy data inventory as code**: every feature declares its data footprint. That generates "What Kanwal knows", retention, export/delete coverage, and the policy text | MUST |
| PLT-10 | **Control plane (ours) separate from cells**: billing, entitlements, license signing, releases, fleet. Holds no customer content. Cells never call the payment gateway | MUST (design v0) |

## 7. Product rules

**Sharing resolution.** The most specific rule wins (person > group > team > everyone). On a tie, the more restrictive wins (`off` > `always_me` > `check_with_me` > `just_answer`). **No rule means `always_me`.** The PM's email, calendar, private chats, and drafts are never shared, and there's no switch for that.

**Question outcomes.**

| Situation | Result |
|-----------|--------|
| Any check fails or errors | Nothing sent · generic reply · goes to **Unanswered** |
| Item restricted for the asker | Refusal that never reveals the item exists |
| `always_me` / `off` | Goes to **Unanswered**. Asker told "That one's Sara's call" |
| No tier passes its checks | Goes to **Unanswered**. "The docs don't say. I've asked Sara." |
| `check_with_me` | Goes to **Needs approval** |
| `just_answer` + checks pass | Sent with sources and AI label · goes to **Answered** |
| Asker flags / PM marks incorrect | Goes to **Flagged** → correction → optionally re-sent |

**External LLM rule.** Content goes to an external LLM (BYO key) **only if** IT enabled that provider **and** the project isn't marked `local-only` **and** the profile allows it. The PM and IT see "AI runs at \<provider\>" on every such answer.

**Agency.** v0 reads, drafts, and answers. No email sends, ticket edits, or permission changes. Document and tool content never triggers an action.

**Retention defaults.**

| Data | Default | Range | Set by |
|------|---------|-------|--------|
| Model training | Never | No switch | — |
| PM private chats | 30 d (D-01) | Off the record, 7, 30, or 90 d | PM within IT ceiling |
| Colleague Q&A + thread context | 30 d | 7–90 d | IT ceiling, PM shortens |
| Corrections / verified answers | Until deleted or the source changes | — | PM |
| Audit log | 90 d | 30–365 d | IT (+ works council) |
| Search index | Mirrors the source, deletes propagate | Exclude any project, space, or channel | IT |
| Usage events (no text) | 12 mo | 3–24 mo | IT |
| Model server logs | Off | 24 h with a banner | IT |
| Sent to Bullah Labs | Nothing | Opt-in anonymous totals | IT |

**Metrics.** Only finished events count (an answer sent and not marked wrong, a draft accepted). The PM sets the minutes per event (default 6 min per covered question). Wait time and wrong-answer rate sit beside every estimate. Events carry **no text**. Admins see totals only for groups ≥ 5. No rankings.

**Voice.** The PM is the hero ("Your call", "I drafted, you approve"). At most one light line per screen, and no jokes in errors, permissions, deletes, or security. Use PM words, never "agent/embedding/routing". "I'm not sure" beats a guess. Every answer carries the AI label (AI Act Art. 50). All copy goes through i18n.

**UX.** Clicks < 0.1 s · first words ~1 s (≤ 2 s at 10 concurrent) · > 10 s shows progress · one-click correct/undo/take over · one rules page and one pause switch · WCAG 2.2 AA, keyboard-only.

## 8. Non-functional requirements

| ID | Target |
|----|--------|
| NFR-01 | First token < 2 s at 10 concurrent askers |
| NFR-02 | 100 real questions (EN/DE) meet the bar set in week 1 |
| NFR-03 | A non-builder installs it in < 1 h from the guide |
| NFR-04 | Backup restores cleanly. RTO ≤ 4 h. Tested before each release |
| NFR-05 | WCAG 2.2 AA · last two versions of Chrome/Edge/Firefox/Safari · i18n from day one |
| NFR-06 | Pilot sizing: ≤ 10 PMs on 1× 24 GB GPU, 8 cores, 64 GB RAM, 500 GB NVMe |
| NFR-07 | ≥ 50% fewer tokens than the single-big-model baseline at equal quality (measured) |
| NFR-08 | Signed releases, customer-controlled updates, no auto-update |

## 9. Security requirements

| ID | Requirement |
|----|-------------|
| SEC-01 | Three gates per question: IT policy → Owner sharing → live source ACL **as the asker**. Fail closed |
| SEC-02 | Access is checked before retrieval **and** before send, in plain code, never by the model |
| SEC-03 | Retrieved and tool content is data, never instructions. It can't trigger actions |
| SEC-04 | Plain-text rendering: no auto-loaded images or links except verified source links |
| SEC-05 | Refusals never reveal that a restricted item exists |
| SEC-06 | OIDC with the company IdP only. No local passwords. Slack users are mapped to IdP identities; unmapped means refused |
| SEC-07 | Model servers are private-network only, with telemetry off |
| SEC-08 | Egress deny-by-default. Allow-list per connector/provider. A shipped proof script |
| SEC-09 | Read-only least-privilege connector tokens and **LLM API keys in the secrets vault**, envelope-encrypted, rotated, never logged, never sent to the browser |
| SEC-10 | Rate limits per asker, output caps, bounded queues, per-provider spend caps |
| SEC-11 | Supply chain: pinned and checksummed weights (safetensors), SBOM, dependency scans, signed images |
| SEC-12 | Append-only, hash-chained audit. Admins cannot read private chats |
| SEC-13 | TLS everywhere, encryption at rest, encrypted backups with per-cell keys |
| SEC-14 | MCP: default-deny tools, hash pinning, per-user audience-bound tokens, sandboxed local servers, write tools need approval |

**Release gates (Dec 31):** access test 50 cases with 0 leaks · injection test 30 poisoned docs with 0 actions and 0 leaks · load (10 concurrent, first token < 2 s) · restore · install < 1 h · accessibility · quality (100 questions).

## 10. Plan (v0, 264 h)

| Week | Dates | Work | Milestone |
|------|-------|------|-----------|
| 1 | Nov 16–20 | Foundations; model test + answer ladder (start); **spikes: Confluence DC ACL check, egress proof** | Re-plan |
| 2 | Nov 23–27 | Ladder (finish); connectors + index | Jira/Confluence sync |
| 3 | Nov 30–Dec 4 | Access checks; Think it through | 50-case access test, 0 leaks |
| 4 | Dec 7–11 | Cover mode, rules, sharing picker, approvals, **review inbox** | Answers with sources |
| 5 | Dec 14–18 | Recap, heads-up, metrics; data controls + IT console (start) | Colleague-facing complete |
| 6 | Dec 21–24 | Data controls + IT console (finish); hardening | Dec 22 feature freeze |
| 7 | Dec 28–31 | Hardening; buffer | Dec 31 pilot-ready |

**Cut order if behind:** heads-up → report polish → sharing by person (sharing by project stays). **BYO-key providers and Slack add scope**, which needs a re-plan in week 1 (D-22, D-23).
