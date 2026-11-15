# 06. Privacy-first architecture (MUST)

Privacy is **not a settings page. It's a property of the architecture**, and the product's positioning is built on it. Inspired by the patterns in "Examples of Privacy-First Positioning" (Confluence, Umer Sohail): **Obsidian** (your data lives with you, in open formats, and you own it) and **Dia** (a plain-language privacy promise users can actually read).

> The screenshots in that page weren't available to me while drafting. Confirm the specific claims we want to mirror and add them to PRIV-1.

Related: [02 Security](./02-security-architecture.md) · [04 Deployment](./04-deployment.md) · [05 Feature control](./05-feature-control-and-customization.md) · [Product rules §5](../../product/02-product-rules.md)

---

## PRIV-0. The privacy promises (positioning → architecture)

Every promise we make in marketing must map to a **mechanism** and a **proof**. If we can't prove a promise, we don't make it.

| # | Promise (user-facing words) | Architectural mechanism | How anyone can verify it |
|---|------------------------------|-------------------------|--------------------------|
| P1 | **"Your data stays with you."** | Cell per customer. Local profile by default. No shared tenancy ([04](./04-deployment.md)) | Network diagram + the egress-proof script |
| P2 | **"Nothing phones home."** | Egress deny-by-default. No telemetry, analytics, crash reporters, CDNs, or remote fonts. Model runtime telemetry off | Egress-proof script, CSP report, firewall logs |
| P3 | **"We never train on your data."** | No training or fine-tuning pipeline exists in the product. Model weights are read-only and pinned | Contract clause + an SBOM with no training components |
| P4 | **"You own it, and you can take it with you."** | Export in open formats (JSON + Markdown + CSV) for every data type. A clean-exit uninstall | One-click export. The uninstall check in the guide |
| P5 | **"You can see everything Kanwal knows."** | A data inventory generated from feature manifests (PRIV-2) → the "What Kanwal knows" page | The page itself, which always matches the running build |
| P6 | **"Off the record means off the record."** | An off-the-record path never writes chat content to the DB, the logs, or the model server's disk | Integration test + model server logging off |
| P7 | **"We can't read it, even when we host it."** | Per-cell KMS key (customer-managed key option). Per-Owner envelope keys for private chats. No standing operator access | Key policy, break-glass audit trail |
| P8 | **"No tracking, no ranking of people."** | The events table has no text columns. Admins see team totals only for groups of ≥ 5. No per-person analytics anywhere in the code | Schema + a CI check that fails on new text columns in `events` |
| P9 | **"A privacy policy you can read in two minutes."** | The plain-language policy is generated from the same data inventory, so it can't drift from reality | Diff between the policy and the inventory in CI |

## PRIV-1. Positioning patterns we adopt

| Pattern | From | How Kanwal applies it |
|---------|------|------------------------|
| Local-first: the default install keeps everything on the customer's machines | Obsidian | `local` is the first and default profile. Cloud is opt-in |
| Open formats, no lock-in | Obsidian | Export is a first-class feature (FR-09), not a support ticket |
| No account with *us* required | Obsidian | Sign-in is only through the customer's IdP. We never hold user accounts for local or private-cloud installs |
| Optional, explicit sync or cloud, chosen by the user | Obsidian | Remote MCP servers, external model endpoints, and anonymous-totals sharing are all **off** until IT turns them on, and each one is listed on the "data leaves your network to" page |
| Plain-language privacy promise up front | Dia | A short "Our promises" page (P1–P9) in the product, on the website, and in the trust pack |
| Say what is collected and why, in context | Dia | Kanwal says what's kept and how to undo it **at the moment of action** (product rules §7) |

## PRIV-2. Data inventory as code

Every feature, connector, and MCP pack **must declare its data footprint** in its manifest ([05 FC-2](./05-feature-control-and-customization.md)). A feature without a privacy block fails CI.

```yaml
privacy:
  data_read:   [canonical.work_item, canonical.doc]     # what it reads
  data_stored:
    - type: colleague_qa
      contains_text: true
      retention: { default: 30d, min: 7d, max: 90d, setter: it_then_owner }
      encrypted: per_owner_key
      exportable: true
      deletable_by: [owner, subject]
  data_leaves_cell: none        # none | allow-listed host list (needs IT opt-in)
  model_calls: local_only       # local_only | it_approved_endpoint
  personal_data: [asker_identity]
  lawful_basis_hint: legitimate_interest   # input for the customer's DPIA
```

From this one source we **generate**:
1. The **"What Kanwal knows"** page (per install, reflecting the enabled features only).
2. The **retention job config**, so retention can't be forgotten.
3. The **export and delete coverage map**. CI fails if a stored type isn't covered.
4. The **plain-language privacy policy** sections and the **DPIA / records-of-processing annex** for the trust pack.
5. The **"data leaves your network to"** list. Any non-`none` entry needs an IT opt-in in the feature control layer.

## PRIV-3. Privacy-by-default rules for engineers

- A new data type is **not stored** unless the manifest says so, with a retention rule.
- **Minimize at the edge.** Store references (`source_id`, URL) instead of copies, wherever the feature still works.
- **No content in logs, metrics, traces, or error reports.** Only IDs and hashes (see [project standards](../coding-standards/project-standards.md)).
- **Encryption by default:** at rest (DB, backups, volumes), in transit (TLS everywhere inside the cell too), and per-Owner keys for private content.
- **Deletion is real.** Deleting a record also removes it from the index, embeddings, caches, and the next backups (after the backup retention window, which is stated in the policy).
- **Privacy review gate:** any PR that changes a `privacy:` block, adds a stored type, or adds egress needs a privacy reviewer (CODEOWNERS).

## PRIV-4. Privacy across deployment profiles

| | local | private-cloud | managed |
|---|---|---|---|
| Where data lives | Customer hardware | Customer's cloud account | Dedicated cell in our cloud, in a region the customer picks |
| Who holds the keys | Customer | Customer KMS | Per-cell key. **Customer-managed key (BYOK) option** |
| Our access | None | None (unless the customer grants support access) | Break-glass only, customer-approved, audited |
| What reaches us | Nothing | Nothing | Health and usage totals only, no content |
| Our GDPR role | Software vendor | Software vendor | Processor (DPA/AVV required) |

The privacy promises P1–P9 hold in **every** profile. Only the "who operates" column changes.
