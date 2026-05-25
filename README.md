# Foundry Private Networking — Samples

Two reference implementations of **private networking for Azure AI Foundry Agents**. Pick the one that matches your scenario; **both are fully supported by Microsoft going forward**.

| | **Managed VNet** | **BYO VNet (Delegated Subnet)** |
|---|---|---|
| **Repo** | 👉 [foundry-private-managed-vnet](https://github.com/SridharArrabelly/foundry-private-managed-vnet) | 👉 [foundry-private-byo-vnet](https://github.com/SridharArrabelly/foundry-private-byo-vnet) |
| **Agent compute location** | Microsoft-managed VNet (invisible to you) | **Your** VNet, in a delegated subnet (`Microsoft.App/environments`) |
| **Agent types supported** | Hosted + Prompt | Hosted + Prompt |
| **Outbound network plumbing** | Foundry-managed PEs from MS VNet → your VNet | Single-tenant Data Proxy in your delegated subnet |
| **Subnet IP planning** | Not required | **Required** — `/24` recommended, `/26` minimum for 50 concurrent sessions |
| **BYO Cosmos + Storage + AI Search** | ✅ Mandatory | ✅ Mandatory |
| **capabilityHost binding** | ✅ Required | ✅ Required |
| **Private endpoints on data resources** | ✅ Plus Foundry auto-creates a 2nd PE | ✅ You create them yourself (no auto-create) |
| **Jumpbox + Bastion for portal access** | ✅ Included | ✅ Included |
| **Operational complexity** | Lower — no IP budget to manage | Higher — IP exhaustion is a real risk under load |
| **Network audit clarity** | Lower — traffic from MS VNet is opaque | Higher — every IP and flow is in your VNet |
| **Best for** | Most production workloads; default starting point | Highly regulated industries (banking, government, healthcare) that require agent compute IPs in the customer VNet |

## Which one should I use?

Walk through these questions in order. Stop at the first **Yes**.

1. **Does compliance require agent compute IPs to live in our own VNet, or that all agent traffic be auditable at the customer's NSG/Firewall?**
   → **BYO VNet.** No way around it. If SecOps demands all outbound traffic flow through an Azure Firewall / NVA with FQDN allowlists, only BYO gives you that — Managed VNet's egress is opaque and not routable through your firewall.

2. **Do we need to allow-list specific agent IPs at downstream firewalls (legacy PaaS, on-prem systems via ExpressRoute), or call any non-Azure-native backend privately (internal API behind a PE, on-prem service)?**
   → **BYO VNet.** Managed VNet IPs are dynamic and not exposed, and its hidden VNet isn't peered with yours — so it can't reach anything in your hub-spoke topology, your ExpressRoute, or your private endpoints to non-Foundry resources.

3. **Will agents have very high concurrency (>50 sessions/region) AND we want predictable scaling without hitting platform-side limits?**
   → **BYO VNet** with a `/24` subnet. The platform supports up to 50 concurrent sessions per subscription per region; sizing your own subnet gives explicit IP headroom for revisions and upgrades.

4. **Do we need Hosted agents (custom container images via our own ACR)?**
   → **BYO VNet.** Hosted agents only ship on the delegated-subnet model.

5. **Otherwise (most cases): is "private endpoints on data + no public Foundry endpoint" enough?**
   → **Managed VNet.** Simpler to reason about, no subnet sizing, fewer moving parts.

If you answered **No** to 1-4 and **Yes** to 5, start with [Managed VNet](https://github.com/SridharArrabelly/foundry-private-managed-vnet). You can migrate later — the data layer (Cosmos/Storage/Search + capabilityHost + RBAC + DNS zones) is identical between the two, so the work isn't thrown away.

## What's the same between them

Both repos share the same **data layer + RBAC chain**:

- BYO Cosmos DB (thread state) + Storage (file/agent dirs) + AI Search (vector store)
- All four data resources have `publicNetworkAccess: Disabled` and a private endpoint in your VNet
- Private DNS zones: `privatelink.cognitiveservices.azure.com`, `privatelink.openai.azure.com`, `privatelink.services.ai.azure.com`, `privatelink.documents.azure.com`, `privatelink.blob.core.windows.net`, `privatelink.search.windows.net`
- Project `capabilityHost` binds the three BYO connections to the agent runtime (this is non-negotiable platform requirement)
- Two-phase RBAC: pre-caphost roles (Storage Blob Data Contributor, Cosmos DB Operator, Search Index/Service Contributor) → capabilityHost provisions → post-caphost roles (Storage Blob Data Owner with ABAC condition on `*-azureml-agent`, Cosmos SQL Built-in Data Contributor)
- Windows jumpbox + Azure Bastion for accessing the private Foundry portal

## What's different

The **network injection layer** is the only thing that changes:

| Concern | Managed VNet | BYO VNet |
|---|---|---|
| Foundry account `networkInjections` property | Not set (or `Managed`) | Set to subnet IDs + `Microsoft.App/environments` delegation |
| Customer subnet for agent compute | None — Foundry's own VNet | `/24` recommended, delegated to `Microsoft.App/environments` |
| Auto-created managed PEs | Foundry creates them into your VNet from its hidden VNet (you approve via auto-approver role) | None — you create all PEs yourself |
| Data Proxy | None | One per project, runs in your delegated subnet |
| IP budget consumed in your VNet | Just PEs (1 IP each) | PEs + Data Proxy (~1 IP per 10 pods) + Hosted agent Micro VMs (~1 IP each + revisions) |

## Concrete triggers — when each one is the only right answer

If any of these describes your environment, the choice is already made for you:

**Pick BYO VNet if:**
- The agent has **custom function tools that call your own internal API** behind a private endpoint, or on-prem via ExpressRoute. Managed VNet can't reach those — its hidden VNet isn't peered with yours.
- Compliance / SecOps requires **all outbound traffic flow through Azure Firewall or an NVA** with FQDN-based allowlists. Managed VNet's egress is governed by Microsoft's allowed-outbound list, not yours.
- You need **end-to-end network observability** — NSG flow logs, packet captures, Network Watcher diagnostics on the agent traffic itself.
- You already operate a **hub-spoke landing zone** and the agent is "just another workload" that drops into a spoke alongside your apps, Bastion, jumpboxes.
- You need a **Private DNS Resolver / corporate DNS forwarder** in the agent's resolution path (e.g., split-horizon DNS with on-prem zones).

**Pick Managed VNet if:**
- The agent only ever calls **Azure-native backends** that Microsoft can reach via its auto-created PEs — your BYO Cosmos / Storage / Search, plus Azure OpenAI.
- You want the **lowest operational burden** — no subnet sizing, no delegation, no NAT Gateway, no NSG rules to tune.
- You're OK with **Microsoft owning the agent VNet** and treating that runtime as a black box you can't `tcpdump`.

**Rule of thumb:** *if the agent needs to call anything in your own network, pick BYO; if it only talks to Azure-native data services, pick Managed.* Most regulated-industry production deployments (banking, healthcare, government) end up on BYO because of trigger #2 above. Most internal POCs and SaaS-style workloads pick Managed.

## Where to start

- **Most clients:** [foundry-private-managed-vnet](https://github.com/SridharArrabelly/foundry-private-managed-vnet) — battle-tested baseline, end-to-end-deploy validated. Has a deep "Understanding the design" section that's worth reading before any troubleshooting call.
- **Regulated clients:** [foundry-private-byo-vnet](https://github.com/SridharArrabelly/foundry-private-byo-vnet) — baseline implementation derived from the [Foundry networking deep-dive](https://learn.microsoft.com/azure/foundry/agents/concepts/agents-networking-deep-dive) and [virtual-networks how-to](https://learn.microsoft.com/azure/foundry/agents/how-to/virtual-networks).

## References

- [Foundry Agent Service — Set up private networking](https://learn.microsoft.com/azure/foundry/agents/how-to/virtual-networks?tabs=portal) (BYO requirements + step-by-step)
- [Foundry Agent Service — Networking deep-dive](https://learn.microsoft.com/azure/foundry/agents/concepts/agents-networking-deep-dive) (subnet sizing, IP allocation, Hosted vs Prompt agent traffic flows)
- [Microsoft sample 18 — Managed VNet + BYO data layer](https://github.com/Azure-Samples/azure-ai-agents) (origin of the Managed VNet pattern)
