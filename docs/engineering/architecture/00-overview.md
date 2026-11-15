# 00. Architecture overview: start here

Plain-language walkthrough. Read this first. Docs 01–07 go deeper on each part.

---

## 1. The big picture in one sentence

**Every customer gets their own private copy of Kanwal (we call it a "cell"): their own app, their own database, their own AI model. Nothing is shared between customers.**

```
   Customer A                     Customer B                     Customer C
 ┌─────────────────┐           ┌─────────────────┐           ┌─────────────────┐
 │  Kanwal app     │           │  Kanwal app     │           │  Kanwal app     │
 │  Database A     │           │  Database B     │           │  Database C     │
 │  AI model A     │           │  AI model B     │           │  AI model C     │
 └─────────────────┘           └─────────────────┘           └─────────────────┘
   on their servers              on a server we rent            in their AWS account
                                 just for them

        ✗ no connection between A, B and C. A bug or hack in one can't reach the others.
```

Where a cell runs depends on the customer. See [07 Hosting and data isolation](./07-hosting-and-data-isolation.md):
- **They have servers** → they install it there (`local`).
- **They have a cloud account** → we install it in *their* AWS, Azure or GCP account (`private-cloud`).
- **They have nothing** → we rent a dedicated server or cloud account **for that customer only** and run it for them (`managed`).

---

## 2. What's inside one cell

```
                         Users (PM, colleagues, IT)  ──login──►  Company login (Google / Microsoft / Okta)
                                     │
                                     ▼
 ┌──────────────────────────────────────────────────────────────────────────────────┐
 │  CELL (one customer)                       internet blocked except allowed tools │
 │                                                                                  │
 │   ① Web app ───────► ② API (the brain, plain code)                               │
 │      (screens)          ├─ Login check                                           │
 │                         ├─ Permission checker  ◄── decides EVERYTHING about access│
 │                         ├─ Feature switches    ◄── what this customer has unlocked│
 │                         ├─ Question handler                                      │
 │                         └─ Rules, approvals, recap, metrics                      │
 │                               │            │              │                      │
 │                               ▼            ▼              ▼                      │
 │                        ③ Database    ④ AI model     ⑤ Tool connector             │
 │                        (all data,    (runs inside   (MCP gateway: talks to       │
 │                         encrypted)    the cell)      Jira, Linear, Asana…)       │
 │                                                                                  │
 │   ⑥ Background worker: syncs tools, deletes old data, builds recaps              │
 └──────────────────────────────────────────────────────────────────────────────────┘
                                                          │
                                                          ▼
                                              Jira · Confluence · Linear · Asana …
                                              (the customer's own tools)
```

| # | Part | What it does | Can it make access decisions? |
|---|------|--------------|-------------------------------|
| ① | Web app | Screens: chat, rules, approvals, recap, IT console | No, it only shows things |
| ② | API | Runs all the logic, in plain, tested code | **Yes. It's the only place that can** |
| ③ | Database | Stores settings, the search index, chats, logs. Encrypted | No |
| ④ | AI model | Writes answers from the text it's given | **Never.** It only sees what was already approved |
| ⑤ | Tool connector | Calls Jira, Linear, etc. **as the person asking**, with their own login | No, it only carries out approved calls |
| ⑥ | Worker | Background jobs | No |

**The key rule:** the AI model is treated like an untrusted intern. It can read what it's handed and write a draft. It can't decide who sees what, and it can't press any buttons.

---

## 3. Flow: a colleague asks a question (the most important flow)

> Jonas asks: *"When is Release 3.4 due, and is anything blocking it?"* Sara (the PM) is in a meeting.

```
 Jonas
   │ 1. asks in Kanwal
   ▼
 ┌──────────────────────────────────────────────┐
 │ 2. Who is this?                              │  Logged in via company login? No → stop
 │ 3. Is Kanwal on? Is this source allowed?     │  IT rules              No → stop
 │ 4. What did Sara allow for this project?     │  Sara's sharing rule:
 │      "off" or "always me" ───────────────────┼──► pass to Sara, tell Jonas "Sara's call"
 │      "check with me" / "just answer" ↓       │
 │ 5. Find relevant tickets/pages               │  search the index
 │ 6. Can JONAS open each one? (ask Jira live,  │  any "no" → drop it
 │    using Jonas's own login)                  │  any error → STOP, send nothing
 │ 7. AI drafts an answer from ONLY those       │
 │ 8. Check the draft: only cites allowed       │  cites something else → send to Sara instead
 │    sources? Re-check access once more        │
 │ 9. Sara's rule = "check with me"? ───────────┼──► waits in Sara's approval queue
 └──────────────────────────────────────────────┘
   │ 10. answer + sources + "from Kanwal" label
   ▼
 Jonas                         11. logged in the audit log + Sara's recap (no answer text in metrics)
```

**The safety idea:** there are **three locks** (IT rules → Sara's rules → Jira's own permissions). Every lock is checked by code. If anything is unclear or breaks, **nothing gets sent**.

---

## 4. Flow: a new customer gets set up

```
 1. Pick where it runs      → their servers / their cloud / dedicated server we rent   (see doc 07)
 2. Install the cell        → one command (Docker Compose) or Terraform+Helm for cloud. < 1 hour
 3. Load the license file   → unlocks the features they bought (signed, checked offline)
 4. IT connects company login (OIDC) → no passwords stored in Kanwal
 5. IT connects tools       → Jira/Confluence (built-in) or any MCP tool (Linear, Asana…)
                              every tool starts OFF; IT approves each one
 6. IT sets limits          → retention, allowed projects, which teams may use what
 7. Each PM: 5-min setup    → pick a north star, set sharing rules per project
 8. Colleagues connect their own Jira/Linear login once → Kanwal can now check THEIR access
```

---

## 5. Flow: adding a new tool through MCP (e.g. Linear)

```
 1. IT: "Add MCP server" → URL of Linear's MCP server
 2. Kanwal lists every tool it offers (search_issues, create_issue, …) and fingerprints each one
 3. Every tool starts BLOCKED. IT marks each one:
        read         → can be used automatically     (search, get issue)
        write        → only with the PM's approval     (comment, create issue)
        destructive  → always off                      (delete, change permissions)
 4. IT picks who may use it (groups) and on what (teams/projects)
 5. Users connect their own Linear login once
 6. If Linear later changes a tool → it's automatically blocked again until IT re-approves it
```

---

## 6. Flow: is a feature unlocked?

Every feature passes through these switches in order. **Any "no" means it's off.** Each later step can only turn things off, never on.

```
 Built into this version? → Customer's license has it? → Allowed in this hosting type?
   → IT allows it? → User's role allows it? → PM's project rules allow it? → User preference? → ✅ ON
```

Example: the license includes "Inbox", but IT turned it off → nobody sees Inbox, and the API refuses it too.

---

## 7. Flow: data is deleted (privacy)

```
 Retention timer runs out   ─┐
 PM clicks "Forget this"    ─┼─► worker deletes it from: database · search index · AI context cache
 Colleague: "Delete my data"─┘     → logs that it happened (not what it was)
 Page deleted in Confluence ─────► removed from Kanwal's index at the next sync
 Customer leaves            ─────► uninstall wipes everything. The guide shows how to verify
```

---

## 8. Where to read next

| If you want to know… | Read |
|----------------------|------|
| Where the database lives when a customer has no servers | [07 Hosting and data isolation](./07-hosting-and-data-isolation.md) |
| The technical components in detail | [01 System architecture](./01-system-architecture.md) |
| MCP and tool connections in detail | [03 Integrations and MCP](./03-integrations-and-mcp.md) |
| Local vs cloud deployment | [04 Deployment](./04-deployment.md) |
| Locking/unlocking features, customization | [05 Feature control](./05-feature-control-and-customization.md) |
| Privacy promises and how we prove them | [06 Privacy-first](./06-privacy-architecture.md) |
| Attacks and how we block them | [02 Security architecture](./02-security-architecture.md) |
