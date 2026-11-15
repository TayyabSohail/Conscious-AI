# 02. Product rules

These are the rules the product must follow, independent of how it's built. Requirements ([01](./01-requirements.md)) say *what* we build. This file says *how the product behaves*. Engineering implements these rules. It does not reinterpret them.

## 1. Principles (in priority order)

1. **Security and correct execution first.** If a check fails or errors, Kanwal sends nothing.
2. **Privacy-first, provably.** Every privacy promise has a mechanism and a proof ([privacy architecture](../engineering/architecture/06-privacy-architecture.md)). If we can't prove it, we don't claim it.
3. **The Owner is the hero.** Kanwal suggests, drafts, and covers. The Owner decides, approves, and can undo.
4. **Private by default.** A fresh install saves the least it can, calls nothing outside, and trains nothing.
5. **Permissions are inherited, never invented.** Kanwal mirrors company access. The Owner can only narrow it.
6. **Show your work.** Every answer has sources, and every action is in a log the Owner can read.
7. **When in doubt, do less.** If the sources don't say it plainly, pass the question to the Owner.
8. **Measure before you add.** If the metric doesn't move, the feature goes.
9. **The smallest model that clears the bar.** Plain code first, then a small model, then a larger one. Quality is tested before cost is cut.

## 2. Sharing and ground rules

The Owner sets rules per project or space.

| Dimension | Options |
|-----------|---------|
| **What** | Tickets, specs and pages, decisions, files |
| **Who** | My teams, a company group, named people, everyone |
| **How** | `just_answer` · `check_with_me` · `always_me` · `off` |

Rules:
- **Narrow only.** A rule can't share anything the asker couldn't already open in the source system.
- **Never shared, no switch:** the Owner's email, calendar details, private chats, and drafts.
- **Presets:** "Team only", "Open book", "While I'm away".
- **Preview as \<person\>** shows exactly what that person would get.
- A change applies **instantly**, is **logged**, and offers **undo**.
- **Most specific rule wins.** Person beats group beats team beats everyone. On a tie, the **more restrictive** mode wins (`off` > `always_me` > `check_with_me` > `just_answer`).
- **No rule means `always_me`.** The default is never "answer".

## 3. What happens to a question

| Situation | Result |
|-----------|--------|
| Any access check fails or errors | Nothing sent. Generic refusal. Owner notified |
| The item is restricted for the asker | Refusal that **never reveals the item exists** |
| Mode is `always_me` or `off` | Passed to the Owner: "That one's Sara's call, so I won't guess." |
| Sources don't answer it plainly | Passed to the Owner, or "Want me to ask Lena?" |
| Mode is `check_with_me` | Draft goes to the approval queue. Nothing is sent until approved |
| Mode is `just_answer` and the sources are clear | Answer with sources, labeled as coming from Kanwal, and listed in the recap |

## 4. Agency limits

- **v0** reads, drafts, and answers. It never sends email, edits tickets, or changes permissions.
- Content from documents **never** triggers an action.
- Anything going outward beyond a rule-allowed answer needs the Owner's approval.

## 5. Data and retention defaults

IT sets the outer limits, the Owner chooses inside them, and any chat can go off the record.

| Data | Default | Range | Who sets it |
|------|---------|-------|-------------|
| Model training | **Never** | No switch in v0 or v1 | Nobody |
| Private chats | 30 days (open question D-01) | Off the record, 7, 30, or 90 days | Owner, within IT's ceiling |
| Colleague Q&A | 30 days | 7–90 days | IT ceiling, Owner can shorten |
| Audit log | 90 days | 30–365 days | IT (plus works council) |
| Search index | Mirrors the source, deletes propagate on next sync | Exclude any project or space | IT |
| Usage events (no text) | 12 months | 3–24 months | IT |
| Model server logs | Off | On for 24 h with a visible banner | IT |
| Email and calendar (v1) | Summaries 7 days, bodies never indexed | Off, or summaries only | Owner |
| Data sent to Bullah Labs | None | Opt in to anonymous totals | IT |

One-click rights: **Forget this** (Owner), **My data** export or delete (any colleague), **What Kanwal knows** (lists sources, exclusions, and retention), **Clean exit** (uninstall removes everything).

## 6. Metrics rules

- Each Owner picks **one north star**: hours back, faster answers, fewer interruptions, never the bottleneck, or a custom goal.
- **Only finished events count:** an answer sent and not marked wrong, or a draft the Owner accepted.
- **The Owner sets the minutes** per event type. Until then, use 6 min per covered question. Every report says "estimate".
- **Guardrails sit beside every estimate:** wait time, and the wrong-answer rate.
- **Event log** fields: `ts, event_type, project_id, route, latency_ms, tokens_in, tokens_out, source_count`. **No question or answer text, ever.**
- Events: `question_received`, `answer_sent`, `answer_rated`, `passed_to_owner`, `draft_approved|edited|declined`, `correction_made`, `rule_changed`, `access_denied`.
- **Visibility:** an Owner sees only their own numbers. Admins see team totals only for groups of ≥ 5. **No rankings, no per-person analytics.** This is required by works council rules and keeps us outside the AI Act's high-risk category.

## 7. Voice and copy rules

- The Owner is the hero: "Your call", "I drafted, you approve". Kanwal never takes credit.
- At most one light, witty line per screen. **Jokes stop where risk starts.** Errors, permissions, deletions, and security prompts are plain and exact.
- Use a PM's words (pings, blockers, standup). Never use "agent", "embedding", or "model routing" in user copy.
- "I'm not sure" beats a guess. Every estimate is labeled.
- No guilt, no fake urgency, no flattery.
- Every answer carries the AI label (AI Act Art. 50).
- All copy goes through i18n keys. German ships in v1.

## 8. Experience rules

- Clicks respond in < 0.1 s. First words appear in about 1 s (hard limit: 2 s under load). Anything > 10 s shows progress and lets the user move on.
- Every empty screen says what Kanwal can and can't do.
- Correcting, undoing, or taking over is one click, and corrections stick.
- One page for all rules, and one pause switch.
- A model or rule change comes with a plain note.
- WCAG 2.2 AA, fully keyboard-driven.
