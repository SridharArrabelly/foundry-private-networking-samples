# Design Rationale

This page answers the four design questions that are easy to get wrong, hard to find documented, and almost always come up the first time someone debugs a private Foundry deployment.

If a deployment looks healthy but the agent run still fails, the answer is usually one of these four.

---

## 1. Why must I bring all three BYO resources, even if I only care about private Search?

A common starting assumption is: *"I just want my AI Search private. Can I leave Cosmos and Storage as the default Microsoft-managed ones?"*

The answer is **no, and the reason is `capabilityHost`**.

`capabilityHost` is a single resource that binds **three** connection arrays on the project:

```
project / capabilityHosts / caphostproj
   ├─ threadStorageConnections = [ → your Cosmos connection  ]
   ├─ storageConnections       = [ → your Storage connection ]
   └─ vectorStoreConnections   = [ → your AI Search connection ]
```

There is no `capabilityHost` variant that lets you bind one or two of these and leave the others as system-managed. The API requires the whole triple. If any of the three is missing, `capabilityHost` validation fails and the agent runtime keeps using the hidden Microsoft-managed resources for everything — including Search.

The practical consequence: **the moment you decide you want any one BYO data resource for the agent runtime, you have signed up to bring all three**. That is why every template in this repo family deploys all three even when the customer scenario only cares about one of them.

> **Reviewer tip:** if a customer pushes back on "why do I need to bring Cosmos when I only want private Search?", this is the answer. It is not over-engineering; the platform requires it.

---

## 2. What happens if you skip capabilityHost

This is the single most common silently-failing scenario in private Foundry deployments.

The infrastructure looks perfect:

- Foundry account ✅
- Project ✅
- BYO Cosmos / Storage / Search ✅
- Connections on the project ✅
- Private endpoints ✅
- DNS ✅
- RBAC ✅

You create an agent, send a message, and the run fails with:

```
Invalid endpoint or connection failed
```

That is the **only diagnostic** you get. No "missing capabilityHost". No "connection X not bound". Nothing.

The reason is that the project connections are **inert pointers** without `capabilityHost`. They describe a target but they have no runtime mapping. When you create `capabilityHost`, three things actually wire up:

1. The agent runtime is told "use these connections for thread storage / file storage / vector store".
2. The Foundry control plane creates **managed private endpoints** from its hidden VNet to each BYO resource.
3. The Foundry account managed identity uses its **Azure AI Enterprise Network Connection Approver** role to auto-approve those managed PEs.

Without `capabilityHost`, none of that happens — and the agent runtime falls back to a code path that returns the unhelpful "Invalid endpoint" error.

**How to verify in 30 seconds:** see [Validation checklist, Check 4](./validation-checklist.md#check-4--capabilityhost-is-bound-to-all-3-connections).

---

## 3. Why are there two completely separate PE paths to the same backend resources?

A private Managed VNet deployment ends up with what looks like duplicate private endpoints — your `pep-…-cosmos` in your VNet, **and** a managed PE to the same Cosmos account from Microsoft's side. The same for Storage and Search.

This is intentional. It exists because there are **two unpeered VNets** in play, and each one needs its own path to the backend:

| VNet | Owned by | Used for | Needs PE to backend? |
|---|---|---|---|
| Your VNet | You | Jumpbox, portal access, your own apps, your indexer | Yes — your PE |
| Foundry-managed VNet | Microsoft (hidden) | The agent runtime itself | Yes — managed PE |

The two VNets are not peered. They cannot share private endpoints. So the same Cosmos account ends up with two private connections to it — one from each side.

That is also why your jumpbox and the agent runtime can both talk to the same Cosmos account privately, but you cannot just look at "your" PEs and tell whether the runtime is healthy. You have to look at both.

> In the **BYO VNet** flavor this collapses to a single set of PEs, because the agent runtime is injected into your VNet via a delegated subnet — there is no hidden Microsoft VNet to bridge to.

See the [PEs table](./capabilityhost-rbac-dns.md#why-so-many-private-endpoints) for the full enumeration.

---

## 4. Why `authType: AAD`, not `ProjectManagedIdentity`?

When you create the Cosmos / Storage / Search connections on the project, the connection resource has an `authType` property. The natural-sounding choice for a managed-identity-based deployment is `ProjectManagedIdentity` — and it deploys cleanly. It looks right. It is wrong.

The actual semantics:

| `authType` value | What it means | Works with capabilityHost? |
|---|---|---|
| `AAD` | "Use the caller's Azure AD token at create time, and let the runtime substitute its own identity at execution time" | **Yes** |
| `ProjectManagedIdentity` | "Use the project MI for both create and execute" | No — runtime calls fail |
| `ApiKey` | "Use the key embedded in the connection" | Yes, but defeats the purpose of private MI-based deployment |

`AAD` is the right answer because at **runtime**, `capabilityHost` overrides the connection's auth context anyway and presents the **project managed identity** token to the backend resource. The connection only needs to validate at create time that the caller has enough rights to register the resource, which is what `AAD` provides.

`ProjectManagedIdentity` sounds like exactly what you want, but it triggers a different runtime code path that does not interoperate with `capabilityHost`. The connection is created successfully, the resource is registered, but the agent run fails (often again with `Invalid endpoint or connection failed`).

> **This is the value used in Microsoft's [sample 18 / Standard Agent](https://github.com/Azure-Samples/azure-ai-foundry-samples) reference template.** If you find yourself reading the template and wondering whether `AAD` is a typo — it is not. Use `AAD`. Confirm via [Validation checklist, Check 5](./validation-checklist.md#check-5--connections-use-the-correct-auth-type).

---

## Summary

The four "whys" all trace back to the same root cause: **`capabilityHost` is the binding layer between the project, the runtime, and the BYO data resources, and it has opinions you have to design around**.

| Question | Short answer |
|---|---|
| Why all three BYO resources? | `capabilityHost` requires the full triple. |
| What if I skip `capabilityHost`? | Connections are inert pointers; agent run fails with `Invalid endpoint or connection failed`. |
| Why dual PE paths? | Two unpeered VNets (yours + Microsoft-managed) each need their own PE. Disappears in BYO VNet. |
| Why `authType: AAD`? | Runtime calls happen under the project MI via `capabilityHost`; `AAD` is the only value that interoperates correctly. |

## Related docs

- [Shared data plane](./shared-data-plane.md) — what the three BYO resources do
- [capabilityHost, RBAC, and DNS](./capabilityhost-rbac-dns.md) — concrete tables of PEs, zones, roles
- [Validation checklist](./validation-checklist.md) — CLI checks for each rationale above
- [Known limitations](./known-limitations.md) — what to expect when this goes wrong
- [Managed VNet architecture](./architecture-diagrams/managed-vnet.md)
- [BYO VNet architecture](./architecture-diagrams/byo-vnet.md)
