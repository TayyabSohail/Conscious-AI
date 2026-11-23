# Kanwal: Product Plan

## 1. Overview

Kanwal is an AI assistant for a Project Manager (PM). Colleagues ask Kanwal their quick questions about the PM's projects instead of interrupting the PM. Kanwal finds the answer in Jira and Confluence, replies to the person who asked, and passes anything it cannot answer with certainty to the PM.

Principles:

1. **Same access as the asker.** A person is told only what they could already open themselves.
2. **No guessing.** If the sources do not state the answer, Kanwal replies "I don't know, please ask the PM" and notifies the PM.
3. **Never speaks as the PM.** Every reply is labeled as coming from Kanwal, shows its sources, and states that it can be wrong.
4. **Reads and replies only.** Kanwal reads the company's tools and replies to people. It does not edit tickets, pages, or email.
5. **One private space per company.** Each company's data is kept separate, on infrastructure that we host and operate.

## 2. Users

| User | Role |
|---|---|
| Project Manager | Connects the tools, invites the team, sets a rule for each project, chats privately with Kanwal, approves and corrects answers |
| Team member | Asks questions and sees their own questions and answers |
| IT admin | Manages sign-in and tool connections, can pause Kanwal, and reads the audit log. Cannot read the PM's chats |

## 3. How it works

```
SET UP ONCE (the PM)

  Sign in  ->  connect Jira and Confluence  ->  pick projects
           ->  set a rule for each project  ->  invite the team


EVERY QUESTION

  A team member asks Kanwal in the web app
            |
            v
  Is this person invited and allowed to ask
  about this project?  ---------- no ---------->  "Not available"
            | yes
            v
  Which rule did the PM set for the project?
            |
            +-- Always me ---------------------->  goes to the PM
            |
            +-- Just answer, or Check with me
                      |
                      v
            Kanwal looks only in what this person can open
            and tries the cheapest step first:

              Step 1  Lookup script, no AI           free
              Step 2  Low-cost AI model              under a cent
              Step 3  Stronger AI model, if needed   a few cents
                      |
                      |   An answer counts only if every
                      |   statement in it is in the sources.
                      |
                      +-- no step can answer -->  Step 4: the PM.
                      |                           The asker is told
                      | answered                  "I don't know,
                      v                           please ask the PM"
            Just answer:    the reply goes to the person who asked
            Check with me:  the PM approves first, then it goes


AFTERWARDS (the PM)

  One review inbox:  Answered / Needs approval / Unanswered / Flagged
  The PM answers what was passed on, and the reply goes back to
  the person who asked. A wrong answer is corrected once and
  Kanwal reuses the correction.
```

### Project rules

The PM sets one rule for each project. New projects start on Always me.

| Rule | What Kanwal does | What the team member sees |
|---|---|---|
| Just answer | Answers on its own when every check passes | The answer, with its sources |
| Check with me | Writes a draft for the PM to approve, edit, or decline | "Waiting for the PM", then the answer |
| Always me | Passes the question to the PM | "Passed to the PM" |
| Off | Does not answer about this project | "Not available" |

### Answer steps

Kanwal tries the cheapest step first and moves on only when a step cannot produce an answer in which every statement is backed by a source.

| Step | What answers | Cost per question |
|---|---|---|
| 1 | A lookup script, for facts such as due dates, owners, statuses, and counts | Free |
| 2 | A low-cost AI model | Under a cent |
| 3 | A stronger AI model | A few cents |
| 4 | The PM | None |

Questions that need the PM's judgment, such as an opinion, a decision, or a commitment, go straight to step 4.

### Checks before an answer is sent

All five must pass. Any failure or error means nothing is sent, and a refusal never reveals that restricted content exists.

1. Kanwal is active, and the person is an invited, active member.
2. The PM's rule allows this person to ask about this project.
3. Jira or Confluence confirms, live, that this person can open every source used.
4. Every statement in the answer is found in those sources.
5. Access is confirmed again immediately before sending.

## 4. Version 0

### Features

| # | Feature | Description |
|---|---|---|
| 1 | Setup | The PM signs in, connects Jira and Confluence, selects projects and spaces, and receives a first answer in about five minutes. No prompt writing is needed |
| 2 | Team | The PM invites people by email. Each person signs in with their company Google or Microsoft account and connects their own Jira and Confluence account once. Removing a person ends their access immediately |
| 3 | Rules | One rule per project, and who may ask: everyone invited, or selected groups. The PM also sets Kanwal's tone and can add a short introduction to a project |
| 4 | Private chat | The PM asks about their projects and drafts messages. Answers show their sources. Drafts are never sent. Off-the-record chats are not saved |
| 5 | Cover mode | Team members ask in the web app and are answered there. The PM approves, answers, and corrects from one review inbox |
| 6 | Flags | Kanwal monitors Jira with fixed rules: overdue, blocked, no update for a set number of days, and conflicting dates. The PM can dismiss or snooze a flag |
| 7 | Dashboard | Needs my attention today, Waiting on me, Important flags, and the PM's North Star for the week |
| 8 | Trust controls | A "This is wrong" button for team members, data retention, delete and export, a pause switch, and an audit log |

The North Star is one measure the PM selects: hours back, faster answers for the team, fewer interruptions, never the bottleneck, or a goal of their own. Time-saved figures are labeled as estimates, and individuals are never ranked.

### Integrations

| Tool | What Kanwal reads | Use |
|---|---|---|
| Jira Cloud | Projects, tickets, comments, assignees, due dates | Answers and flags |
| Confluence Cloud | Spaces and pages | Answers |

Both connections are read-only, and access is checked live for the person asking.

### Reading and replying

| System | Kanwal reads | Kanwal writes |
|---|---|---|
| Jira and Confluence | The projects and spaces the PM selected | Nothing |
| Kanwal web app | Questions | Replies |
| Email | Nothing | Invitation emails from Kanwal's own address, containing a sign-in link only |

- Kanwal replies only to the person who asked, in the web app.
- Three things are sent without the PM: an answer that passed every check, the "I don't know" reply, and a status such as "Passed to the PM".
- Drafts, the PM's own replies, and corrections are sent only when the PM approves them.

## 5. Architecture

### Tech stack

| Layer | Technology | Purpose |
|---|---|---|
| Codebase | One repository: `apps/web`, `apps/api`, `infra`, `evals`. pnpm for the web app, uv for Python | Code, infrastructure, and tests in one place |
| Web app | Next.js, React, TypeScript, Tailwind, shadcn/ui, TanStack Query | User interface. No access decisions are made in the browser |
| Server | Python 3.12, FastAPI, Pydantic, SQLAlchemy, Alembic | API and access rules, written as testable code |
| Background jobs | Procrastinate: a Python worker with its job queue and schedules in PostgreSQL | Keeps the copy of Jira and Confluence current, raises flags, and deletes data when retention ends. No separate queue service to run |
| Database | PostgreSQL 16 with pgvector | Application data, search index, sessions, jobs, and audit log |
| AI models | Amazon Bedrock: a low-cost model first, a stronger model when needed | Pay per use. The text stays inside AWS and is not stored or used for training |
| Search index | An open embedding model running on the app server | Tickets and pages are indexed on our own servers |
| Jira and Confluence | Atlassian Cloud APIs. Each person connects once through Atlassian sign-in, read-only | Kanwal sees only what that person can see |
| Sign-in | OpenID Connect with Google and Microsoft | No passwords are stored |
| Edge | Caddy for incoming traffic, Squid for the outgoing allow-list | Automatic certificates, and outbound traffic limited to approved hosts |
| Packaging | Docker, Docker Compose, Amazon ECS, Terraform | Every company runs the same signed release. Compose runs it on a developer's machine, ECS in production. Terraform adds each company's database, key, role, and containers the same way every time |
| Testing | pytest and Hypothesis | Access rules are tested with generated cases as well as written ones |

### AWS services

Kanwal runs on AWS in US West (Oregon), in one production AWS account that we operate. Companies share the servers and the database server. Each company has its own database, database login, encryption key, permissions role, file storage, and set of application containers. Nothing is installed on the company's systems, and people use Kanwal through a web browser.

| AWS service | Role | Purpose |
|---|---|---|
| Organizations | Separate AWS accounts for production and for staging, under one bill. A company on dedicated hosting gets its own account | Test systems never share an account with company data |
| IAM | One role per company's containers, and roles for staff | A company's containers can use only that company's key, secrets, and files |
| VPC | The private network around the production system | The app and database are not reachable from the internet |
| EC2 | Shared servers: an edge host and app hosts | Run the application containers |
| ECS | One set of containers per company on the shared app hosts | Each set holds only its own company's database login and role |
| EBS | Encrypted disks for those servers | Server storage |
| RDS for PostgreSQL | One shared database server, with a separate database and login per company | A company's login cannot open another company's database. Automatic backups, seven days of point-in-time recovery, managed patching |
| KMS | One encryption key per company, and one platform key | The company key protects each PM's own key and the company's stored Jira and Confluence connections. The platform key encrypts the shared database server, disks, and backups |
| S3 | One encrypted bucket per company for exports | Durable storage, reached privately from inside the network, readable only by that company's role |
| Bedrock | The AI models, reached through a private endpoint inside the network | Questions and passages do not cross the internet. Each company's role can use only the approved models, and no AI key is stored |
| SES | Sends invitation emails | Kanwal's own sender address |
| Route 53 | The web address for each company | DNS |
| Systems Manager | Maintenance access and setup secrets | No remote-login port, and every staff session is logged |
| CloudWatch | Metrics, alarms, and logs | Monitoring. Logs contain no question or answer text |
| CloudTrail | A record of every change in the AWS account | Accountability for all access, including ours |
| GuardDuty | Threat detection | Alerts on signs of attack |
| ECR | Signed container images | Every company runs the same verified release |

### Scalability

| Growth | Approach | Basis |
|---|---|---|
| More people in one company | More containers for that company, then larger shared servers | The server holds no state between requests; all state is in the database |
| More companies | A new database, login, key, role, and container set from the same template, and more shared servers as they fill | Companies share servers, never a database, login, or key |
| A company that needs full separation | Its own AWS account from the same template, as a custom service | The same containers and PostgreSQL run in both setups |
| A company's own servers | The same containers, with the company's own PostgreSQL | The application is containers and PostgreSQL only |
| More source applications | A connector for each application, or an MCP server | Every source uses the same contract: search, read, check access |
| Kanwal inside other applications | The Kanwal API and MCP server | The web app already uses the same API |

Four components are built as replaceable modules from the start: the source connector, the AI provider, the encryption key store, and the mail sender. New integrations and deployment options are then additions to the base.

## 6. Security and data handling

### Data handling

- Each company's data is stored in its own database, with its own database login and encryption key.
- Companies share the servers, the database server, and the network. The disks and backups of the shared database server are encrypted with one platform key.
- A company's key is deleted when the company leaves, which makes its PM keys and stored connections unreadable. Its database and files are deleted at the same time.
- A company that needs full separation can have a dedicated AWS account as a custom service.
- Kanwal is hosted and operated by us. A data processing agreement is signed before any production data is connected.
- When an answer needs AI, Kanwal sends only the question and the passages that passed the access checks. They go to Amazon Bedrock inside our AWS account and are not sent to any other company. AWS does not store them, use them for training, or share them with the model makers.
- The PM is shown what is sent to AI during setup and agrees before AI is enabled.
- The search index is built on our own servers. Source content is not sent out for indexing.
- A project marked "keep local" sends nothing to AI. It is answered by lookup or passed to the PM.
- IT can disable AI at any time. Kanwal then answers by lookup and passes all other questions to the PM.
- Each company has a monthly AI spending cap and an AI rate limit, both enforced in code. When the cap is reached, Kanwal answers by lookup and passes all other questions to the PM.

### Security controls

- **Sign-in:** company Google or Microsoft accounts only, limited to the company's domain. Team members must be invited by the PM. No passwords are stored.
- **Company separation:** the company is taken from the signed-in session only. Each company's containers hold only that company's database login and role, so they cannot open another company's database, key, or files. If the company cannot be confirmed, no database connection is made.
- **Access decisions:** made in code, not by AI. Any error means nothing is sent.
- **Source permissions:** checked live as the person asking. Each person connects their own Jira and Confluence account once, read-only.
- **Encryption:** all data is encrypted in transit and at rest. Chat and Q&A text is encrypted again with a key per PM.
- **Network:** the app and database have no public address. There is one entry point, and outbound traffic is limited to an approved list.
- **Untrusted content:** text from tickets and pages is treated as information, never as instructions, and the AI cannot take actions.
- **Logging:** no question or answer text is written to logs, and the audit log is tamper-evident.
- **Operations:** our staff have no standing access. Emergency access is logged and reported to the company.

## 7. Performance

- The web app, server, and database run continuously, so nothing has to start on demand.
- Facts such as a due date, an owner, or a status are answered by lookup, without AI.
- AI answers are streamed as they are written, with a target of first words within about one second.

### Limits per company

Companies share the servers, the database server, and the AI rate limits of the AWS account. Every limit below is set for each company, so one company's use does not take another's share. All of them are required before production data.

| Limit | Where it is set | When it is reached |
|---|---|---|
| Memory and processor for the company's containers | ECS | Only that company's containers slow down or restart |
| Database connections and query time | The company's database login | The company's extra connections wait, and long queries are stopped |
| Background job speed | The company's worker | Its Jira and Confluence sync runs more slowly |
| Monthly AI spending cap | Application code, which counts each company's AI use. Bedrock has no cap of its own | Kanwal answers by lookup and passes all other questions to the PM |
| AI requests per minute | Application code. Bedrock's own limits are shared by the whole account | The company's extra requests wait briefly, then go to the PM |
| Database load per company | A CloudWatch measure and alarm for each company, read from PostgreSQL's own statistics | We are alerted, and a company with consistently heavy use is moved to its own database server |

Very heavy database use by one company can still slow the others until it is throttled, because PostgreSQL cannot cap processor or disk use per database. The usage counts and measures hold numbers only, never question or answer text.

## 8. Cost

Monthly cost in US dollars, with five companies on the shared servers. The shared figures are rough estimates until the server sizes are confirmed by a load test.

| Item | Five companies | Per company |
|---|---|---|
| Shared application servers and disks | About 140 | About 28 |
| Shared database server | About 110 | About 22 |
| Backups, encryption keys, monitoring, and network | About 40 | About 8 |
| AI usage | About 165 | About 33 |
| Total | About 455 | About 91 |

| Configuration | Per company per month |
|---|---|
| Standard (v0), at five companies | About 91 |
| Company's own AI key (v1) | About 58, plus the company's own AI charges |
| Dedicated AWS account (custom) | About 170 |
| Dedicated private GPU (custom) | About 240 to 655, depending on GPU hours |

- Five companies on the standard configuration cost about 455 dollars a month, or about 9 dollars per PM at ten PMs per company.
- The shared servers cost about 290 dollars a month whether they hold one company or five, so the cost per company is higher until they fill.
- AI usage assumes about 2,000 team questions and 4,400 PM chat messages per company per month, with one in ten needing the stronger model. Actual cost depends on the models selected and on usage, and each company has a monthly spending cap.
- The AI usage figure was priced for OpenRouter and has to be priced again for the Bedrock models we choose. The Bedrock private endpoint adds about 8 dollars a month for each availability zone.
- Shared platform costs are about 170 to 350 dollars a month in total, regardless of the number of companies.
- Sales tax, AWS support plans, and engineering time are not included.

## 9. Delivery plan

**Stage 1: Foundation**

1. AWS template, database, and deployment, running as a staging environment with test data. Includes the per-company container and database limits.
2. Sign-in, workspaces, access checks, encryption, audit, and data retention.

**Stage 2: Core product (Version 0)**

3. Jira Cloud and Confluence Cloud connections, with the live access check.
4. Lookup answers and the "I don't know" reply.
5. AI answers through Amazon Bedrock, with source checking, the off switch, and the per-company spending cap and rate limit. The PM's private chat.
6. Cover mode: rules, approvals, review inbox, and corrections.
7. Flags, dashboard, and trust controls.

**Stage 3: Version 1**

8. Team chat: Slack and Teams.
9. Email and calendar: Gmail and Outlook.
10. The remaining Version 1 features.

**Release criteria before production data**

- A signed data processing agreement with the customer.
- 50 attempts to reach restricted content, with zero leaks.
- Attempts from one company's containers to reach another company's database, key, and files, with every attempt refused.
- A company that reaches its AI spending cap or rate limit gets no further AI answers, and other companies are unaffected.
- 30 poisoned tickets and pages, with no action taken and no leak.
- 100 real questions that meet the quality bar.
- A successful restore from backup.
- A load test with ten people asking at once.

## For future: Version 1

| Feature | Description |
|---|---|
| Bring your own AI key | A company, or a PM where IT allows it, adds its own API key for Claude or ChatGPT models, and Kanwal uses that account. Each answer shows which provider produced it. This requires an API key from Anthropic or OpenAI; a chat subscription does not include one |
| Custom tools through MCP | IT, or a PM where IT allows it, connects another tool through an MCP server. Each tool starts disabled. Read tools can be enabled, and any tool that writes requires the PM's approval each time |
| Kanwal API and MCP server | Other applications and AI tools can ask Kanwal a question and receive the sourced answer. Every request runs as a named, signed-in person and passes the same checks |
| Team chat: Slack and Teams | Team members ask Kanwal in a private message and are answered there. Kanwal can also read the Slack channels the PM selects, and uses a message only for members of that channel |
| Email: Gmail and Outlook | Read-only access to the folders or shared mailbox the PM selects, used in the PM's private chat only |
| Calendar | Meeting times and titles |
| More sources | Trello, Google Drive, OneDrive, SharePoint, Linear, Jira Service Management |
| Jira and Confluence Data Center | Reached through a relay inside the company's network that connects out to Kanwal, so no inbound ports are opened |
| Approved actions | Kanwal proposes a comment or a new ticket, and the PM approves each one |
| Advanced sharing | Rules for a named person, and workspaces with more than one owner |
| Recap | A daily or weekly summary of what Kanwal answered and what needs the PM |
| Private AI and custom hosting | A dedicated GPU running an open model so that no text leaves our infrastructure, a dedicated AWS account that we operate for one company alone, or Kanwal deployed on the company's own servers or AWS account. Offered as a custom service |
| Additional sign-in providers | Other single sign-on providers, such as Okta |
| Localization and billing | German interface, and billing per PM seat |
