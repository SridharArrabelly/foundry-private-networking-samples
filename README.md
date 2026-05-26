# Foundry Private Networking Samples

Two deployable reference implementations for private networking with Azure AI Foundry Agents.

Use this repo as the **decision hub**. Start here, choose the right flavor for your scenario, then go to the child repo for deployment.

## Choose your flavor

| Option | [Managed VNet](https://github.com/SridharArrabelly/foundry-private-managed-vnet) | [BYO VNet (Delegated Subnet)](https://github.com/SridharArrabelly/foundry-private-byo-vnet) |
|---|---|---|
| Agent compute location | Microsoft-managed VNet | Your VNet, in a delegated subnet |
| Best for | Most production workloads | Highly regulated workloads that require agent IPs inside the customer VNet |
| Network planning | Simpler, no subnet sizing | Requires delegated subnet and IP planning |
| Audit visibility | Lower, compute is Microsoft-managed | Higher, flows and IPs are visible in your VNet controls |
| Operational complexity | Lower | Higher |

## Which one should I use?

Walk through these in order. Stop at the first **Yes**.

1. **Do you need agent compute to live inside the customer VNet?** If yes, choose **BYO VNet**.
2. **Do you need agent traffic to be visible in the customer's NSG or firewall logs?** If yes, choose **BYO VNet**.
3. **Do downstream systems need explicit agent IP allow-listing or customer-owned network controls?** If yes, choose **BYO VNet**.
4. **Do you want the simplest private setup with the fewest moving parts?** If yes, choose **Managed VNet**.

If none of the first three apply, start with **Managed VNet**.

> **Note on agent types.** Both Managed VNet and BYO VNet now support Prompt and Hosted Agent services in the current Foundry portal. Choose between the two flavors based on **runtime placement, auditability, and customer network control** — not on agent type.

## What is shared across both samples?

Both samples share the same private data-layer pattern:

- BYO **Cosmos DB** for thread state
- BYO **Storage** for files and agent directories
- BYO **AI Search** for retrieval / vector store
- **capabilityHost** binds those resources to the agent runtime
- The templates are **azd-deployable**

See [Shared data plane](./docs/shared-data-plane.md) for the full picture.

## Repos

### Managed VNet
Recommended starting point for most private-networking scenarios.

- Repo: [foundry-private-managed-vnet](https://github.com/SridharArrabelly/foundry-private-managed-vnet)
- Pattern: Managed VNet + Microsoft-managed agent runtime + capabilityHost
- Use this when private data access is required, but you do not need agent compute inside your own VNet

### BYO VNet
Use this when agent compute must run inside the customer VNet.

- Repo: [foundry-private-byo-vnet](https://github.com/SridharArrabelly/foundry-private-byo-vnet)
- Pattern: BYO VNet + delegated subnet (agent runtime inside the customer VNet) + Data Proxy
- Use this for highly regulated environments, IP allow-listing scenarios, or customer-owned traffic visibility

## Docs

Read these roughly in this order — visual context first, then the design reasoning, then the operational detail.

**Architecture:**
1. [Side-by-side comparison](./docs/architecture-diagrams/side-by-side.md) — start here for the visual difference between Managed and BYO
2. [Managed VNet architecture](./docs/architecture-diagrams/managed-vnet.md)
3. [BYO VNet architecture](./docs/architecture-diagrams/byo-vnet.md)

**Design and implementation detail (shared across both flavors):**

4. [Design rationale](./docs/design-rationale.md) — the four "why" questions: full BYO triple, what happens if you skip `capabilityHost`, dual PE paths, `authType: AAD`
5. [Shared data plane](./docs/shared-data-plane.md) — what Cosmos, Storage, AI Search are used for
6. [capabilityHost, RBAC, and DNS](./docs/capabilityhost-rbac-dns.md) — concrete tables: 7 PEs, 6 DNS zones, two-phase RBAC
7. [Validation checklist](./docs/validation-checklist.md) — 7 copy-paste CLI checks plus prose validation flow
8. [Known limitations](./docs/known-limitations.md) — named issues you will hit (`azd down` SDK bug, `CustomDomainInUse` 48h soft-delete, region capacity)

## Why this repo family exists

These samples are meant to be practical references you can use to:

- Validate a customer's private-networking design
- Show the difference between Managed VNet and BYO VNet clearly
- Give teams a starting point they can adapt into production
- Reduce time spent re-explaining the same networking patterns in every engagement
