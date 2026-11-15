# 07. Hosting and data isolation (customers without their own servers)

**Question:** if a customer has no servers, where do their database and app run, and how do we make sure nothing is shared with other customers?

**Answer:** we give each customer a **dedicated cell** on rented infrastructure: their own server or cloud account, their own database, and their own encryption key. **We never put two customers in one database.**

Prices below are rough starting points. Check current list prices before quoting.

---

## 1. Ways to separate customer data (weakest → strongest)

```
 Level 1  One database, tenant_id column        ┌──────── DB ────────┐
          (typical SaaS)                        │ A rows │ B rows │ C │   ✗ one bug in a WHERE clause
                                                └────────────────────┘     = data leak
 Level 2  One DB server, schema per customer    ┌──────── DB server ──┐
                                                │ schema A │ schema B │   ✗ shared server, shared admin
                                                └─────────────────────┘
 Level 3  One DB server, database per customer  ┌──────── DB server ──┐
                                                │ [DB A]   │ [DB B]   │   ~ better, still shared machine
                                                └─────────────────────┘
 Level 4  Separate DB instance per customer     [DB server A]   [DB server B]       ✓ strong
 Level 5  Separate EVERYTHING per customer      [app+DB+AI+key A] [app+DB+AI+key B] ✓✓ strongest  ◄── OUR CHOICE
          (the "cell")
```

**We use Level 5.** That's the only level where our privacy promise "your data is never mixed with anyone else's" (P1) holds without asterisks. It also protects more than the database: the AI model, the logs, the backups, and the encryption keys are all per customer.

## 2. Docker is the packaging, not the place

Docker isn't an alternative to AWS or DigitalOcean. It's **how we package Kanwal** (app + worker + database + AI model as containers). The same Docker images run on every option below:

```
                      ┌─► customer's own server
 Kanwal Docker images ├─► a dedicated rented server (Hetzner, DigitalOcean, OVH…)
 (signed, one version)├─► customer's AWS / Azure / GCP account
                      └─► an AWS account we open for that customer
```

## 3. Hosting options compared

| | **A. Dedicated GPU server per customer** (Hetzner / OVH / DigitalOcean GPU) | **B. Cloud account per customer** (AWS, or Azure / GCP) | **C. DigitalOcean droplet + managed DB per customer** | **D. Supabase project per customer** |
|---|---|---|---|---|
| **What each customer gets** | One rented server running the whole cell in Docker. Postgres runs on it, on an encrypted disk | A separate AWS account: its own network (VPC), its own managed Postgres (RDS), its own GPU machine, its own KMS key | Its own droplet (app), its own managed Postgres cluster, its own VPC. AI from a GPU droplet or a hosted model | Its own Supabase project, which is its own Postgres instance |
| **Isolation** | ✓✓ Physically separate machine | ✓✓ Strongest: separate account, IAM, network, keys | ✓ Separate DB cluster + VPC, inside our DO team | ✓ Separate DB per project, but auth/API come from Supabase's platform |
| **AI model (GPU)** | ✓ Included on the box (e.g. 24 GB GPU) | ✓ GPU instance in the same account | ~ GPU droplets cost more and exist in fewer regions. Or a "lite cell" using a hosted model | ✗ No GPU. The AI must run elsewhere |
| **"Nothing leaves the cell"** | ✓ Easy: one firewall | ✓ Easy: security groups + egress firewall | ✓ Firewall + VPC | ✗ Hard. Supabase is an extra third party, and its API is on the internet by default |
| **Regions (EU)** | ✓ Germany / Finland (Hetzner), France (OVH) | ✓ Frankfurt, Paris, Stockholm, Ireland … + US | ~ Amsterdam, Frankfurt, London (check GPU availability) | ✓ Several EU regions |
| **Compliance story** | ~ Provider is ISO 27001 certified. We own more of the controls | ✓✓ Best (AWS SOC 2/ISO/C5 + per-account isolation is easy to explain) | ~ Good certifications, fewer enterprise features | ~ SOC 2. Adds a sub-processor |
| **Rough cost per customer / month** | **~€214** (Hetzner GEX45, GPU incl.), + backups | **~$600–900** (GPU instance dominates) + RDS ~$30–60 | ~$50–100 without GPU (lite cell). GPU droplets much more | ~$25+ per project, **plus** GPU elsewhere |
| **Ops effort for us** | ~ We patch the OS and run Postgres ourselves (scripted) | ~ More setup, but fully automatable with Terraform | ✓ Simple | ✓ Simple for the DB only |
| **Fits** | **Pilots, small/medium EU customers (v1)** | **Enterprise, regulated, US customers (v2)** | Cheap "lite" tier without GPU | **Not for customer data.** OK for our own internal tools and dev |

## 4. Recommendation

```
 Phase            Customer has no servers?   Where their cell runs                          Who owns the account
 ───────────────  ─────────────────────────  ─────────────────────────────────────────────  ─────────────────────
 Pilot (v0/v1)    yes                        A. Dedicated GPU server (Hetzner, EU)          customer (preferred) or us
 Growth (v1/v2)   yes, small budget          C. "Lite cell": droplet + managed DB +         us
                                                in-region hosted open model
 Growth (v2)      yes, enterprise/regulated  B. One AWS account per customer                customer (preferred) or us
 Any              they have a cloud account  B. In THEIR account (private-cloud profile)    customer
 Any              they have servers          Their own servers (local profile)              customer
```

**Why "customer owns the account" is preferred even when we run it:** if the server or cloud account is in the customer's name and on their bill, the data is legally theirs and on their infrastructure. That makes us a software vendor, not a data processor, which keeps our GDPR duties much lighter. We just get admin access to deploy and maintain it, and that access is audited.

**Supabase:** use it only for **our own** things, like the website, the waitlist, or internal dashboards. It can also hold the fleet control plane's metadata **if that contains no customer content**. Don't use it for customer data, because:
1. it adds a third party into every customer's data path (it breaks "nothing leaves the cell"),
2. it doesn't host the AI model, so the cell gets split across providers, and
3. its built-in auth and API are things our architecture deliberately doesn't use (decision D-07).

## 5. How one hosted cell is built (option A: dedicated server)

```
 ┌──────────────────── Dedicated server for Customer X (Hetzner, Germany) ────────────────────┐
 │  Firewall: only HTTPS in (from the customer's IP range / VPN). Out: only Jira, Linear, …   │
 │                                                                                            │
 │  Docker Compose                                                                            │
 │   ├─ proxy (HTTPS)                                                                         │
 │   ├─ web + api + worker                                                                    │
 │   ├─ mcp-gateway (+ sandboxed MCP servers)                                                 │
 │   ├─ model server (GPU)                                                                    │
 │   └─ postgres + pgvector   ◄── Customer X's database, encrypted disk, key unique to X      │
 │                                                                                            │
 │  Nightly: encrypted backup ──► object storage bucket owned by Customer X (separate key)    │
 └────────────────────────────────────────────────────────────────────────────────────────────┘
        ▲
        │ our deploy/upgrade: signed release pulled · our admin access logged in X's audit log
```

## 6. How one hosted cell is built (option B: AWS account per customer)

```
 AWS Organization (ours or customer's)
 ├─ Management account (Bullah Labs): fleet control plane · NO customer data
 ├─ Account: Customer A (eu-central-1)
 │    VPC (private subnets only)
 │     ├─ app containers (ECS/EKS)       ├─ GPU instance (vLLM)
 │     ├─ RDS Postgres + pgvector        ├─ KMS key A (encrypts DB, disks, backups)
 │     └─ egress firewall: allow-list only
 ├─ Account: Customer B (us-east-1)
 │    … same, with KMS key B …
 └─ (accounts never peer with each other)
```

Creating a new customer = run one Terraform module with `customer_id`, `region`, and `size`. That gives a new account, network, DB, key, GPU, and app. It takes about 30–60 min, fully automated.

## 7. Rules that apply whatever we choose

- **One customer = one database instance, one encryption key, one backup bucket.** Never shared.
- **No network path between cells.** No VPC peering, no shared DB users.
- **Our admin access is break-glass**: approved by the customer, time-limited, and recorded in *their* audit log.
- **Backups are encrypted with the customer's key** and stored in the customer's bucket or region.
- **The fleet control plane** (how we manage many cells) stores only: customer name, region, version, health, and usage totals. **Never content.**

## 8. Decisions needed

| ID | Question | Lean |
|----|----------|------|
| D-18 | First hosting provider for customers without servers | Hetzner dedicated GPU (EU, ~€214/mo) for pilots |
| D-19 | Who owns the hosted account: customer or us? | Customer-owned where possible |
| D-20 | Offer a no-GPU "lite cell" tier? | Yes, using an EU-hosted open model by API, clearly disclosed as "AI runs at <provider> in <region>" |
| D-21 | Supabase role | Internal tools only. Never customer data |
