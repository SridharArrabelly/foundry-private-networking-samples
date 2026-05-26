# Validation Checklist

Use this page after deployment to confirm that the environment works end-to-end.

The goal is not just to confirm that resources were created. The goal is to prove that the **agent runtime can use the private data path successfully**.

## Validation goals

A successful validation should demonstrate all of the following:

- the Foundry environment is reachable through the intended access path
- the agent can use AI Search
- thread state can be written to Cosmos DB
- files can be written to Storage
- the environment behaves as a private deployment, not a public fallback

## Validation order

Run validation in this order:

1. control-plane access
2. private connectivity checks
3. data-plane function checks
4. end-to-end agent workflow checks
5. network-path confirmation

## 1. Control-plane access

Confirm you can reach the environment in the intended way.

Check:

- the Foundry account and project exist
- the expected private access path is available
- any jumpbox / Bastion access path works if the sample includes one
- there is no accidental dependence on an unintended public route

## 2. Private connectivity checks

Before testing agent behavior, confirm the underlying network pieces are healthy.

Check:

- private endpoints exist for the expected resources
- endpoint status is healthy or approved as required
- DNS resolution is correct from the relevant network path
- the dependent resources are not silently falling back to public access

## 3. Data-plane function checks

Validate each shared data resource individually.

### AI Search
Confirm that the agent or related workflow can issue a search / retrieval operation successfully.

What success looks like:

- the call succeeds
- the index is reachable as intended
- retrieval returns expected results for known test content

### Cosmos DB
Confirm that thread or conversation state is written successfully.

What success looks like:

- the runtime persists state
- the expected database / container receives records
- the workflow can continue using stored state

### Storage
Confirm that file operations work successfully.

What success looks like:

- uploads succeed
- expected containers or directories receive content
- the agent can reference the uploaded content as intended

## 4. End-to-end workflow checks

After validating each individual dependency, test the complete agent path.

A simple validation flow:

1. create or open an agent scenario
2. trigger a retrieval action
3. confirm the workflow can access Search
4. confirm state persistence lands in Cosmos DB
5. confirm any file-related operation lands in Storage

The point is to prove that the pieces work **together**, not only in isolation.

## 5. Network-path confirmation

The final step is confirming that the traffic is using the intended network model.

### Managed VNet
Confirm:

- the scenario works through the Managed VNet design
- there is no need to expose the data resources publicly
- the private path is sufficient for the runtime behavior you expect

### BYO VNet
Confirm:

- the delegated-subnet path is being used
- the expected network visibility exists in customer controls
- the design behaves as intended inside the customer VNet

## CLI verification — 7 concrete checks

Before relying on the prose validation flow below, run these commands. They take about 5 minutes and catch the majority of "looks healthy but doesn't actually work" cases. Run them from inside the VNet (jumpbox) where indicated.

Set common variables once:

```bash
RG="<your-rg>"
ACCT="<your-foundry-account-name>"
PROJ="<your-foundry-project-name>"
SEARCH="<your-search-service>"
COSMOS="<your-cosmos-account>"
STORAGE="<your-storage-account>"
```

### Check 1 — provisioning succeeded

```bash
az group show -n "$RG" --query properties.provisioningState -o tsv
az cognitiveservices account show -n "$ACCT" -g "$RG" --query properties.provisioningState -o tsv
```

Both should return `Succeeded`. If the Foundry account is `Failed` or `Accepted` and stuck, stop and check the deployment operation errors before continuing.

### Check 2 — public network access is OFF on all 4 data resources

```bash
az cognitiveservices account show -n "$ACCT" -g "$RG" --query properties.publicNetworkAccess -o tsv
az search service show --name "$SEARCH" -g "$RG" --query publicNetworkAccess -o tsv
az cosmosdb show -n "$COSMOS" -g "$RG" --query publicNetworkAccess -o tsv
az storage account show -n "$STORAGE" -g "$RG" --query publicNetworkAccess -o tsv
```

All four must be `Disabled`. If any returns `Enabled`, the resource is leaking to the public internet and the rest of the validation is meaningless.

### Check 3 — Managed VNet outbound rules exist (Managed VNet only)

```bash
az resource show \
  --ids "/subscriptions/<sub>/resourceGroups/$RG/providers/Microsoft.CognitiveServices/accounts/$ACCT/networkSecurityPerimeterConfigurations" \
  --query "value[].properties.privateEndpoints[].properties.privateLinkServiceConnectionState.status"
```

You should see at least 3 managed PEs, all `Approved` (one each for Cosmos, Storage, Search). If they are `Pending`, the Foundry account MI is missing the **Azure AI Enterprise Network Connection Approver** role on the resource group.

### Check 4 — capabilityHost is bound to all 3 connections

```bash
az rest --method get \
  --url "https://management.azure.com/subscriptions/<sub>/resourceGroups/$RG/providers/Microsoft.CognitiveServices/accounts/$ACCT/projects/$PROJ/capabilityHosts/default?api-version=2025-04-01-preview" \
  --query "properties.{thread:threadStorageConnections, store:storageConnections, vector:vectorStoreConnections}"
```

All three arrays must be non-empty. If any is empty or the resource is `404 NotFound`, the agent runtime has no idea which BYO resources to use — runtime calls will fail with `Invalid endpoint or connection failed` regardless of how healthy everything else looks. See [Design rationale: what happens if you skip capabilityHost](./design-rationale.md#what-happens-if-you-skip-capabilityhost).

### Check 5 — connections use the correct auth type

```bash
az cognitiveservices account connection list -n "$ACCT" -g "$RG" \
  --query "[].{name:name, target:properties.target, authType:properties.authType}" -o table
```

The Cosmos, Storage, and Search connections must show `authType: AAD`. If you see `ProjectManagedIdentity`, the connection was created with a value that looks correct but is not supported by capabilityHost — see [Design rationale: why authType AAD](./design-rationale.md#why-authtype-aad-not-projectmanagedidentity).

Application Insights, if you added it, will show `ApiKey` — that is correct.

### Check 6 — DNS resolves to private IPs from the jumpbox

From the **jumpbox** (not your laptop):

```powershell
nslookup $env:ACCT.cognitiveservices.azure.com
nslookup $env:ACCT.openai.azure.com
nslookup $env:ACCT.services.ai.azure.com
nslookup $env:SEARCH.search.windows.net
nslookup $env:COSMOS.documents.azure.com
nslookup $env:STORAGE.blob.core.windows.net
```

All six should return **private IPs** in your VNet's PE subnet range (typically `10.x.x.x`), not public IPs. If a name returns a public IP, the corresponding `privatelink.*` zone is missing or not linked to your VNet. See the [DNS zones table](./capabilityhost-rbac-dns.md#private-dns-zones).

**Check 6b — observability private resolution**

If you deployed AMPLS, also confirm:

```powershell
nslookup <ampls-name>.monitor.azure.com
```

returns a private IP, and that the **6 monitor zones** are linked to your VNet: `privatelink.monitor.azure.com`, `privatelink.oms.opinsights.azure.com`, `privatelink.ods.opinsights.azure.com`, `privatelink.agentsvc.azure-automation.net`, `privatelink.blob.core.windows.net`, `privatelink.applicationinsights.azure.com`.

### Check 7 — end-to-end agent smoke test

From the jumpbox:

```bash
# get a token for the project
az account get-access-token --resource https://ai.azure.com --query accessToken -o tsv > token.txt

# create an agent, run a single message, confirm response
python <<'PY'
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
c = AIProjectClient(endpoint="https://<acct>.services.ai.azure.com/api/projects/<proj>",
                    credential=DefaultAzureCredential())
agent = c.agents.create_agent(model="gpt-4o-mini", name="smoke", instructions="be brief")
thread = c.agents.threads.create()
c.agents.messages.create(thread_id=thread.id, role="user", content="say hi")
run = c.agents.runs.create_and_process(thread_id=thread.id, agent_id=agent.id)
print(run.status)  # must print: completed
PY
```

If `run.status` is `failed` with `Invalid endpoint or connection failed`, check (4). If it is `failed` with a Storage/Cosmos/Search error, check (5) and post-caphost RBAC ([Storage Blob Data Owner with ABAC, Cosmos SQL data role](./capabilityhost-rbac-dns.md#rbac-in-two-phases)).

If it returns `completed`, you have proven the full chain works: private DNS → managed PE → capabilityHost binding → BYO data resource → agent runtime.

---



These are the most common patterns during validation:

### Pattern 1: Infrastructure exists, but agent calls fail
Usually check:

- capabilityHost binding
- RBAC
- DNS
- private endpoint readiness

### Pattern 2: Search works intermittently
Usually check:

- role propagation
- index readiness
- DNS resolution consistency
- post-provision step ordering

### Pattern 3: Storage or Cosmos operations fail while Search works
Usually check:

- resource-specific RBAC
- container / database configuration
- identity actually used by the runtime
- endpoint-specific DNS and private connectivity

### Pattern 4: Portal access works, but agent runtime behavior fails
Usually check:

- project-to-resource binding
- runtime identity permissions
- data-plane private path
- post-deployment readiness timing

## Suggested evidence to capture

For customer validation or internal handoff, capture evidence such as:

- deployment outputs
- screenshots of relevant resources
- private endpoint status
- DNS resolution results
- successful Search / Cosmos / Storage test results
- notes on what was validated and from where

## Minimum sign-off checklist

Use this compact checklist before calling the deployment validated:

- [ ] Foundry account and project deployed successfully
- [ ] Private endpoints exist and are healthy
- [ ] DNS resolves through the intended private path
- [ ] AI Search works
- [ ] Cosmos DB state persistence works
- [ ] Storage file operations work
- [ ] End-to-end agent scenario works
- [ ] The environment is not relying on unintended public access

## Related docs

- [Shared data plane](./shared-data-plane.md)
- [Design rationale](./design-rationale.md)
- [capabilityHost, RBAC, and DNS](./capabilityhost-rbac-dns.md)
- [Known limitations](./known-limitations.md)
- [Managed VNet architecture](./architecture-diagrams/managed-vnet.md)
- [BYO VNet architecture](./architecture-diagrams/byo-vnet.md)
