# Foundry Private Networking Samples

Two deployable reference implementations for private networking with Azure AI Foundry Agents.

Use this repo as the **decision hub**. Start here, choose the right flavor for your scenario, then go to the child repo for deployment.

## Choose your flavor

| Option | [Managed VNet](https://github.com/SridharArrabelly/foundry-private-managed-vnet) | [BYO VNet (Delegated Subnet)](https://github.com/SridharArrabelly/foundry-private-byo-vnet) |
|---|---|---|
| Agent compute location | Microsoft-managed VNet | Your VNet, in a delegated subnet |
| Best for | Most production workloads | Highly regulated workloads that require agent IPs inside the customer VNet |
| Agent types | Standard Agent | Hosted agents and Prompt agents |
| Network planning | Simpler, no subnet sizing | Requires delegated subnet and IP planning |
| Audit visibility | Lower, compute is Microsoft-managed | Higher, flows and IPs are visible in your VNet controls |
| Operational complexity | Lower | Higher |

## Which one should I use?

Walk through these in order. Stop at the first **Yes**.

1. **Do you need agent compute to live inside the customer VNet?** If yes, choose **BYO VNet**.
2. **Do you need agent traffic to be visible in the customer's NSG or firewall logs?** If yes, choose **BYO VNet**.
3. **Do you need Hosted agents or custom container images?** If yes, choose **BYO VNet**.
4. **Do downstream systems need explicit agent IP allow-listing?** If yes, choose **BYO VNet**.
5. **Do you want the simplest private setup with the fewest moving parts?** If yes, choose **Managed VNet**.

If none of the first four apply, start with **Managed VNet**.

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
- Pattern: Managed VNet + Standard Agent + capabilityHost
- Use this when private data access is required, but you do not need agent compute inside your own VNet

### BYO VNet
Use this when agent compute must run inside the customer VNet.

- Repo: [foundry-private-byo-vnet](https://github.com/SridharArrabelly/foundry-private-byo-vnet)
- Pattern: BYO VNet + delegated subnet + Data Proxy + Hosted/Prompt agents
- Use this for highly regulated environments, IP allow-listing scenarios, or customer-owned traffic visibility

## Architecture

See the architecture pages for side-by-side visuals and deeper explanation.

- [Managed VNet architecture](./docs/architecture-diagrams/managed-vnet.md)
- [BYO VNet architecture](./docs/architecture-diagrams/byo-vnet.md)
- [Side-by-side comparison](./docs/architecture-diagrams/side-by-side.md)

## Deep docs

Use these for implementation detail that is shared across both repos:

- [Shared data plane](./docs/shared-data-plane.md)
- [capabilityHost, RBAC, and DNS](./docs/capabilityhost-rbac-dns.md)
- [Validation checklist](./docs/validation-checklist.md)
- [Known limitations](./docs/known-limitations.md)

## Why this repo family exists

These samples are meant to be practical references you can use to:

- Validate a customer's private-networking design
- Show the difference between Managed VNet and BYO VNet clearly
- Give teams a starting point they can adapt into production
- Reduce time spent re-explaining the same networking patterns in every engagement
