# Product docs: Kanwal (working name)

Status: **DRAFT for brainstorming**. Nothing here is final until it is marked `DECIDED` in [03-decisions-and-questions.md](./03-decisions-and-questions.md).

This folder covers **what** we build and **how the product behaves**. For how it's built (architecture, codebase, coding standards), see [../engineering/](../engineering/README.md).

Source of truth for scope: the Confluence page "Conscious AI for PMs - Design Requirements" (Nov 13, 2026).

| # | Doc | What it answers |
|---|-----|-----------------|
| 01 | [Requirements](./01-requirements.md) | Roles, features, platform, non-functional and security requirements, with IDs for Linear tickets |
| 02 | [Product rules](./02-product-rules.md) | Principles, sharing logic, question outcomes, retention, metrics, voice, UX rules |
| 03 | [Decisions and open questions](./03-decisions-and-questions.md) | What we still need to decide, with a recommendation for each |

## The product in one paragraph

A private assistant for project managers. It answers colleagues' simple questions while the PM is busy, flags blockers and risks from Jira, and (from v1) sorts the inbox and plans the day. It runs on the customer's own infrastructure on open models. It inherits company permissions, never widens them, cites its sources, logs every action, and proves its value with a metric the PM picks.
