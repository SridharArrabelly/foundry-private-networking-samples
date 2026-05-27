# Shared Data Plane

Both private-networking samples in this repo family use the same core data-layer pattern.

That is true whether you choose:

- **Managed VNet**, where agent compute runs in a Microsoft-managed network boundary
- **BYO VNet**, where agent compute is injected into a delegated subnet in your own VNet

The network path is different between the two flavors, but the data plane is intentionally the same.

## What stays the same in both samples

In both patterns:

- The data resources are **customer-owned**
- The Foundry project must be able to reach them privately
- The identities involved must receive the right RBAC assignments
- `capabilityHost` is the binding mechanism that tells Foundry which resources to use

This is one of the most useful design choices in the repo family. It means:

- You can compare **Managed VNet** and **BYO VNet** without changing the core data resources
- The difference between the two samples is mostly in the **network model**, not in the data model
- Teams can start with one flavor and still preserve the same Cosmos / Storage / Search design

## Shared resource roles in the architecture

### Cosmos DB
Stores thread and state data that the agent needs to persist over time.

Typical responsibilities:

- conversation or thread state
- runtime state associated with agent interactions
- durable backing store for agent workflows

### Storage
Used for files and agent-related data paths.

Typical responsibilities:

- uploaded files
- agent file directories
- artifacts that need durable storage

### AI Search
Used as the retrieval layer.

Typical responsibilities:

- indexed content
- retrieval-augmented lookup
- vector-backed search scenarios

## What changes between the two samples

The data plane stays the same. The network path changes.

### Managed VNet
- Agent compute runs in a Microsoft-managed VNet
- Foundry manages the agent-side network boundary
- You do not manage a customer subnet for agent compute

### BYO VNet
- Agent compute runs in your VNet
- A delegated subnet is required
- Network visibility and control move further into the customer boundary

## Design principle

Think about the repo family in two layers:

### Layer 1: Shared data plane
- Cosmos DB
- Storage
- AI Search
- capabilityHost
- RBAC
- private access goals

### Layer 2: Network flavor
- Managed VNet
- BYO VNet

That separation is deliberate. It makes the comparison easier and keeps the architecture easier to reason about.

## Related docs

- [capabilityHost, RBAC, and DNS](./capabilityhost-rbac-dns.md)
- [Validation checklist](./validation-checklist.md)
- [Known limitations](./known-limitations.md)
- [Managed VNet architecture](./architecture-diagrams/managed-vnet.md)
- [BYO VNet architecture](./architecture-diagrams/byo-vnet.md)
- [Side-by-side comparison](./architecture-diagrams/side-by-side.md)
