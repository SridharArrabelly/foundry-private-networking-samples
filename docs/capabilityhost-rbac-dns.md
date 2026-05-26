# capabilityHost, RBAC, and DNS

This page explains the non-obvious plumbing behind the private data-layer pattern used in both samples.

If the deployment "looks right" but the scenario does not work end-to-end, the problem is usually in one of these areas:

- `capabilityHost`
- RBAC sequencing
- private endpoint readiness
- private DNS resolution

## What capabilityHost does

`capabilityHost` is the binding point between the Foundry project and the customer-owned data resources.

At a practical level, it tells the project which resources back the agent runtime, such as:

- Cosmos DB
- Storage
- AI Search

Without this binding, the agent runtime may have the right infrastructure deployed but still not know how to use the intended BYO resources.

The resource itself looks like this:

```
project / capabilityHosts / default
   ├─ threadStorageConnections = [ → your Cosmos connection  ]
   ├─ storageConnections       = [ → your Storage connection ]
   └─ vectorStoreConnections   = [ → your AI Search connection ]
```

When you create the capabilityHost, three things happen automatically:

1. The agent runtime starts using your three resources instead of the hidden Microsoft ones.
2. Foundry creates **Managed Private Endpoints** from its hidden VNet → your three resources (so the runtime can reach them privately).
3. Those managed PEs need approval — your Foundry account managed identity has the **Azure AI Enterprise Network Connection Approver** role on the resource group so it can auto-approve them.

Without `capabilityHost`, the three project connections are inert pointers — there is no token, no managed PE, and no path.

## Why so many private endpoints

A private Foundry deployment ends up with **two layers of PEs** to the same backend resources — one set in your VNet (for portal / jumpbox / your apps), and one set in Microsoft's managed VNet (for the agent runtime).

| PE | Where | Used by | Why it exists |
|---|---|---|---|
| `pep-…-foundry`  | Your `snet-pe` | You, jumpbox, portal | Reach Foundry account control plane / Agents UI / OpenAI inference |
| `pep-…-search`   | Your `snet-pe` | You, jumpbox (indexer) | Create/update Search indexes, query from your own apps |
| `pep-…-cosmos`   | Your `snet-pe` | You, future apps | Manage Cosmos from inside the VNet |
| `pep-…-blob`     | Your `snet-pe` | You, future apps | Upload files to Storage from inside the VNet |
| Managed PE → Cosmos  | MS-managed VNet | Agent runtime | Agent writes thread state |
| Managed PE → Storage | MS-managed VNet | Agent runtime | Agent uploads/reads agent-scoped files |
| Managed PE → Search  | MS-managed VNet | Agent runtime | Agent calls the AI Search tool / file-search vector stores |

Each "your" PE is **paired** with a managed PE to the same backend. Together they cover both user traffic and runtime traffic without any public exposure.

> In the **BYO VNet** flavor the picture collapses: there is no Microsoft-managed VNet, so the agent runtime reuses the same PEs that your jumpbox uses. The dual-PE pattern is specific to Managed VNet.

## Private DNS zones

A private endpoint is just a private IP — clients still need DNS to resolve the public hostname to it. Each backend service uses a different `privatelink.*` zone:

| Zone | Resolves | Required because |
|---|---|---|
| `privatelink.cognitiveservices.azure.com` | Foundry account control plane | Bicep / SDK uses `<acct>.cognitiveservices.azure.com` |
| `privatelink.openai.azure.com` | Foundry OpenAI inference | Agents call `<acct>.openai.azure.com` for chat / embeddings |
| `privatelink.services.ai.azure.com` | Foundry Agents / Threads APIs | Agents UI calls `<acct>.services.ai.azure.com` |
| `privatelink.search.windows.net` | AI Search | Indexer + agent tool call `<svc>.search.windows.net` |
| `privatelink.documents.azure.com` | Cosmos NoSQL | Cosmos SDK calls `<acct>.documents.azure.com` |
| `privatelink.blob.core.windows.net` | Storage blob | Storage SDK calls `<acct>.blob.core.windows.net` |

All six are linked to your VNet so anything inside (the jumpbox, the agent runtime's managed PEs, your future apps) resolves these hostnames to the corresponding PE IP automatically.

## RBAC in two phases

Some role assignments must exist **before** `capabilityHost` is created (or `capabilityHost` validation hangs trying to reach the resources). Other role assignments can only be created **after** `capabilityHost`, because their scope depends on the workspace GUID that only exists once the project is fully provisioned.

| Phase | Role | Scope | Why |
|---|---|---|---|
| Pre-caphost | **Storage Blob Data Contributor** | Storage account | Caphost validates the project MI can read/write Storage |
| Pre-caphost | **Cosmos DB Operator** | Cosmos account | Caphost validates the project MI can create databases/containers |
| Pre-caphost | **Search Index Data Contributor** + **Search Service Contributor** | AI Search service | Caphost validates the project MI can manage indexes |
| Post-caphost | **Storage Blob Data Owner** (ABAC condition scoped to `*-azureml-agent` containers) | Storage account | Project's workspace GUID is now known; lock data-plane access to the agent's own containers only |
| Post-caphost | **Cosmos SQL Built-In Data Contributor** (`00000000-0000-0000-0000-000000000002`) | Cosmos account | Cosmos data-plane RBAC because we disabled local auth |

That is why a template can appear to deploy successfully but still fail functionally — if the pre-caphost roles are late, `capabilityHost` provisioning hangs; if the post-caphost roles are missing, runtime calls succeed against capabilityHost but fail against the data resources themselves.

## Practical RBAC guidance

When debugging RBAC, confirm all of the following:

- the correct identity is being granted access (project MI, not user MI, not jumpbox MI)
- the role is assigned at the correct scope (data-plane roles on the data resource, not on the resource group)
- the assignment exists before the dependent operation runs (especially pre-caphost roles before `capabilityHost` creation)
- propagation time has been allowed for new assignments (5–10 minutes is typical)
- there are no duplicate or conflicting assignments that make troubleshooting harder

## Common RBAC symptoms

Typical signs of RBAC problems:

- resource deployment succeeds, but runtime calls fail
- AI Search lookups do not work even though the service exists
- Storage access fails for file operations
- Cosmos-based state persistence fails
- post-provision steps succeed intermittently or only after rerunning

> **After an `azd down` → `azd up` cycle**, role assignments can end up pointing at an **orphaned project managed identity** (the one from the previous deploy). Symptom: AI Search tool works, but Playground File Search returns a generic `500`. See [Known limitations → File Search 500 after project recreate](./known-limitations.md#8-file-search-returns-generic-500-in-playground-after-project-recreate).

## Private endpoint readiness

A private endpoint is not useful just because it exists. Before treating it as live, confirm:

- the private endpoint exists
- it is **approved** if approval is required (managed PEs are auto-approved by the Foundry account MI's network-approver role)
- the target resource shows the expected private connectivity state
- DNS resolves the service name to the expected private IP

Until that chain is complete, the runtime behaves as if the resource is unavailable.

## How to think about the dependency chain

A useful mental model is:

1. deploy the data resources
2. deploy private connectivity (your PEs + DNS zones)
3. deploy the Foundry project + connections
4. assign **pre-caphost** RBAC
5. create `capabilityHost` (this auto-creates managed PEs)
6. assign **post-caphost** RBAC (Storage Blob Data Owner + Cosmos SQL data role)
7. validate end-to-end

If one part is missing, the symptom often shows up somewhere else. Example:

- a DNS issue may look like an agent runtime issue
- an RBAC issue may look like a Search or Storage issue
- a capability-binding issue may look like a generic agent failure (`Invalid endpoint or connection failed`)

## Managed VNet vs BYO VNet

The same `capabilityHost` / RBAC / DNS concepts apply in both samples, but the network path differs.

### Managed VNet
- Two layers of PEs (yours + Microsoft-managed) to each backend resource
- The project uses a Microsoft-managed network boundary for agent compute
- The data plane still needs correct binding, access, and resolution

### BYO VNet
- One layer of PEs (yours only) — the agent runtime reuses them
- The project uses customer-controlled network placement for agent compute
- The same binding and access concepts apply, plus customer subnet and network-path considerations

## Troubleshooting order

When something is not working, use this order:

1. Confirm the resources exist and `publicNetworkAccess` is `Disabled`
2. Confirm private endpoints are healthy (and managed PEs are `Approved` for Managed VNet)
3. Confirm DNS resolution returns the expected private IP from the runtime path
4. Confirm `capabilityHost` was created and bound to all 3 connections
5. Confirm RBAC on the actual identities in use (project MI, not user MI)
6. Re-run end-to-end validation

## Related docs

- [Shared data plane](./shared-data-plane.md)
- [Design rationale](./design-rationale.md)
- [Validation checklist](./validation-checklist.md)
- [Known limitations](./known-limitations.md)
- [Managed VNet architecture](./architecture-diagrams/managed-vnet.md)
- [BYO VNet architecture](./architecture-diagrams/byo-vnet.md)
