# Engineering docs

## Architecture

| # | Doc | Covers |
|---|-----|--------|
| 01 | [System architecture](./architecture/01-system-architecture.md) | Cells, components, the life of a question, data, personas, models, scaling |
| 02 | [Security architecture](./architecture/02-security-architecture.md) | Trust boundaries, the three gates, threat model (incl. MCP threats T14–T20) |
| 03 | [Integrations and MCP](./architecture/03-integrations-and-mcp.md) | Native, MCP, and hybrid adapters, MCP gateway and tool broker, tool lifecycle, per-call authorization, adding PM tools |
| 04 | [Deployment](./architecture/04-deployment.md) | `local`, `private-cloud`, `managed` profiles. Release, install, upgrade. Sizing |
| 05 | [Feature control and customization](./architecture/05-feature-control-and-customization.md) | Layered restriction, feature registry, kill switches, customization levels, Cedar policies |
| 06 | [Privacy-first architecture](./architecture/06-privacy-architecture.md) | **MUST.** Promises P1–P9 with mechanism and proof. Data inventory as code |

## Other

| Section | Contents |
|---------|----------|
| [coding-standards/](./coding-standards/README.md) | General UI and backend standards (imported) plus project-specific standards that override them |
| [codebase/](./codebase/README.md) | How the current boilerplate works: structure, routing, env, Button, forms, tables, React Query, Supabase, QA |

Product requirements and rules: [../product/](../product/README.md).
