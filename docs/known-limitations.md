# Known Limitations

This page captures the **named issues** you will hit when deploying this repo family, plus broader caveats.

Use it before positioning the samples as a production baseline, and use it as the first stop when `azd up` or `azd down` misbehaves.

## Named issues you will probably hit

These are not hypothetical — each has bitten this sample at least once. They are listed in roughly the order you encounter them.

### 1. `azd down` fails with `unmarshalling type *armcognitiveservices.NetworkInjections`

**Symptom:**

```
ERROR: deleting infrastructure: deleting resource …Microsoft.CognitiveServices/accounts/<acct>:
unmarshalling type *armcognitiveservices.NetworkInjections: …
```

**Cause:** the Azure SDK version used by `azd` (and `az cognitiveservices`) cannot deserialize the `NetworkInjections` property that gets stamped on Foundry accounts in private deployments. Delete fails before it gets to the resource.

**Workaround:** delete the resource group directly and purge the soft-deleted Cognitive Services account.

```bash
az group delete -n <rg> --yes --no-wait
# wait for the RG to disappear, then:
az cognitiveservices account list-deleted --query "[?name=='<acct>']" -o table
az cognitiveservices account purge -l <region> -g <rg> -n <acct>
```

Until this is purged, the next `azd up` will fail with the `CustomDomainInUse` error below.

### 2. `azd up` fails with `CustomDomainInUse`

**Symptom:**

```
{"code":"CustomDomainInUse","message":"Requested custom domain <acct> is already in use."}
```

**Cause:** Cognitive Services / Foundry accounts hold their **subdomain** for **48 hours** after deletion via soft-delete. The new deployment cannot reuse the name until the old one is purged.

**Workaround:** purge the soft-deleted account (see step 1), then re-run `azd up`. If you cannot wait or purge, change `AZURE_ENV_NAME` (which feeds the account name) and redeploy.

### 3. `RoleAssignmentExists` on re-runs

**Symptom:**

```
{"code":"RoleAssignmentExists","message":"The role assignment already exists."}
```

**Cause:** Bicep role assignments use deterministic GUIDs (`guid(scope, principalId, roleDefId)`). On a partial-failure re-run, the assignment already exists but Bicep tries to create it again.

**Workaround:** this is almost always cosmetic — let `azd up` retry, or delete the offending assignment and re-run. If it blocks deployment, wrap the assignment module in `existing` or guard it with a condition; do not change the GUID computation.

### 4. `IfMatchPreconditionFailed` during parallel child resource creation

**Symptom:** deployment of multiple connections or capabilityHosts running in parallel fails intermittently with an ETag mismatch on the **parent** Foundry project.

**Cause:** ARM serializes writes to the same parent. Parallel children update the parent's ETag at the same time.

**Workaround:** serialize the creation of children using `dependsOn` or `batchSize(1)` on the Bicep loop. Avoid parallel `azd provision` runs against the same RG.

### 5. Region capacity / quota traps

- **Tested in `swedencentral`** as the primary path. `eastus` and `eastus2` are commonly capacity-constrained for `gpt-4o`, `gpt-4o-mini`, and `text-embedding-3-large`.
- `InsufficientQuota` on the embedding deployment usually means the embedding model quota is 0 in the target region — check `az cognitiveservices usage list -l <region>` and request quota or change region before assuming the template is wrong.
- Foundry preview features (Managed VNet, Hosted/Prompt agents, capabilityHost) are not in every region. Verify before changing `AZURE_LOCATION`.

### 6. Bastion provisioning takes ~5 minutes and needs `/26`

The Bastion subnet **must** be named `AzureBastionSubnet` and **must** be at least `/26`. Anything smaller fails validation. First-time provisioning typically takes 5 minutes — that is normal, not a hang.

### 7. Jumpbox PowerShell version

The Windows jumpbox runs **PowerShell 5.1** by default. Scripts copied from a dev machine that use `?.` (null-conditional), `??` (null-coalescing), or ternary `a ? b : c` will fail to parse. Either install PowerShell 7 on the jumpbox first, or use 5.1-compatible syntax — `scripts/jumpbox-bootstrap.ps1` in both samples is intentionally PS 5.1-compatible.

### 7a. Jumpbox image — Windows Server 2022 Azure Edition (not Windows 11)

Both samples deploy `MicrosoftWindowsServer / WindowsServer / 2022-datacenter-azure-edition` for the jumpbox instead of a Windows 11 desktop image. Reasons:

- **No multi-tenant entitlement requirement.** Windows 11 desktop images on Azure VMs require a per-user EA entitlement (Win 10/11 Enterprise E3/E5, Win VDA, or M365 F3/E3/E5). Server-with-Desktop-Experience uses standard per-vCPU PAYG licensing — works in any subscription, no entitlement gymnastics.
- **Stable VM-agent lifecycle.** During earlier testing, the Win 11 image showed the **"VM agent stuck after deallocation"** symptom (the VM would start back up but `az vm run-command invoke` and Bastion would silently fail until a full delete/redeploy). The Server SKU is designed for the Azure-fabric restart lifecycle and does not exhibit this.

**Trade-off:** Edge / Chrome are not pre-installed on Server with Desktop Experience. If you Bastion in and need a browser (e.g. to open the Foundry portal), run once:

```powershell
$msi = "$env:TEMP\edge.msi"
Invoke-WebRequest -Uri 'https://go.microsoft.com/fwlink/?linkid=2093437' -OutFile $msi -UseBasicParsing
Start-Process msiexec.exe -ArgumentList '/i', $msi, '/qn' -Wait
```

The validation checklist itself (CLI + Python smoke test) needs no browser — Python and the Az CLI are installed by `scripts/jumpbox-bootstrap.ps1` during `azd up`.

### 8. File Search returns generic 500 in Playground after project recreate

**Symptom:** The AI Search tool works normally in agents, but the Playground **File Search** tool fails with:

```
The server had an error processing your request. Sorry about that!
```

The vector store appears to be created, the dataset gets registered under **Data → Datasets**, and the AI Search index returns results when queried directly. Only the agent's File Search call fails. The same agent works fine in a non-private / non-BYO setup.

**Cause (most likely):** After an `azd down` → `azd up` cycle, the Foundry project is recreated with a **new system-assigned managed identity**. The pre-caphost and post-caphost role assignments are named via `guid(projectPrincipalId, …)` in Bicep, which produces a name tied to the **old** principal. After redeploy:

- Old role assignments persist on the agent Storage account, pointing at an **orphaned** principal (the deleted project MI).
- Depending on partial-failure paths, the new MI may not receive a fresh `Storage Blob Data Contributor` + `Storage Blob Data Owner` (ABAC) assignment on the agent Storage account.
- File Search reads/writes chunked content into the agent Storage account; without those Storage roles on the **current** project MI, the post-retrieval step fails inside the service and surfaces to the client as a generic `500`.

The AI Search tool keeps working because it uses a different role chain (Search-side roles, which survive the recreate) and does not depend on the agent Storage container.

**How to confirm:**

```bash
# Current project MI principalId
PROJ_MI=$(az resource show \
  --ids "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<acct>/projects/<proj>" \
  --query identity.principalId -o tsv)

# Who actually has Storage Blob roles on the agent Storage account?
STORAGE_ID=$(az storage account show -n <agent-storage> -g <rg> --query id -o tsv)
az role assignment list --scope "$STORAGE_ID" \
  --query "[?contains(roleDefinitionName,'Storage Blob')].{role:roleDefinitionName, principalId:principalId}" \
  -o table
```

If the listed `principalId`s do not match `$PROJ_MI`, the existing assignments are pointing at the orphaned identity and the current MI has no Storage access.

**Workaround:**

```bash
az role assignment create --assignee-object-id $PROJ_MI \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Contributor" --scope "$STORAGE_ID"

az role assignment create --assignee-object-id $PROJ_MI \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Owner" --scope "$STORAGE_ID"
```

Then delete the failed vector store in the portal and re-upload the document. Cleaning up the orphaned assignments is optional but recommended for tidiness.

> **Under investigation.** This is the most likely root cause based on the observed RBAC state in a reproducing deployment, but end-to-end confirmation that granting the roles fully unblocks the `500` is still pending. Treat the workaround as a starting point, not a guaranteed fix. Track this section for updates.

### 9. Resource group delete leaves orphan VNet, NSGs, NAT gateway and Public IP

**Symptom:** after running `azd down`, or deleting the resource group from the portal, these resources refuse to delete and stay behind:

- `vnet-<prefix>` (and all four subnets)
- `vnet-<prefix>-snet-…-nsg-…` (all four NSGs)
- `natgw-<prefix>` + `pip-<prefix>-natgw`

The activity log on the failed RG delete shows:

```
InUseSubnetCannotBeDeleted
  Subnet snet-<prefix>-agent is in use by …/serviceAssociationLinks/legionservicelink
  and cannot be deleted.
```

**Cause:** the Foundry project's `capabilityHost` (kind `Agents`) provisions a Microsoft-managed Container Apps environment **in an internal subscription you cannot see**. That managed env attaches a `legionservicelink` *serviceAssociationLink* (SAL) to your agent subnet. If the Foundry account is deleted while the `capabilityHost` still exists (which is what happens when you delete the RG straight from the portal, or when `azd down` removes the account before tearing the capabilityHost down), the managed env is left **orphaned in the internal subscription** with no API surface for you to clean it up. The SAL on your subnet survives, and from then on nothing in the agent subnet — or anything that depends on it — can be deleted.

You can verify the SAL is the blocker with:

```bash
az network vnet subnet show \
  -g <rg> --vnet-name vnet-<prefix> -n snet-<prefix>-agent \
  --query "serviceAssociationLinks"
```

If `linkedResourceType` is `Microsoft.App/environments` and `allowDelete` is `false`, you have hit this issue. Patching `allowDelete` from your own tenant does **not** work — only the owning RP can flip it.

**Prevention (already wired into both samples):** the `predown` hook in `azure.yaml` deletes the `capabilityHost` **before** azd tears down the Foundry account, which lets the managed env (and its SAL) cascade away cleanly. Make sure you use `azd down` rather than deleting the RG from the portal.

**Recovery if you already hit it:**

1. Try purging the soft-deleted Foundry account first — sometimes this is enough on its own:
   ```bash
   az cognitiveservices account list-deleted --query "[?name=='<acct>']" -o table
   az cognitiveservices account purge -l <region> -g <rg> -n <acct>
   ```
   Re-attempt `az group delete -n <rg> --yes`. If the SAL is still there after the purge, continue.
2. **Open an Azure support ticket** referencing the orphan `Microsoft.App/environments` in the internal subscription. Microsoft has to delete it from their side. This is the only fully reliable path once the SAL is orphaned.
3. Leave the residual resources running while you wait — NAT gateway + public IP are small (~$1.20/day combined per RG); NSGs and the empty VNet are free.

## Repo-family-wide caveats

These apply to both samples.

### Platform behavior can evolve
Azure AI Foundry and Agent Service capabilities continue to change — supported regions, networking behavior, required RBAC details, and preview-to-GA transitions can all shift. Always re-check platform documentation for the exact scenario you are targeting.

### Private networking failures are often indirect
In private deployments, the visible failure is rarely the root cause. A DNS issue can look like an agent failure. An RBAC issue can look like a Search failure. A binding issue can look like a generic runtime error. Plan for systematic validation, not just deployment success.

### Successful deployment ≠ functional readiness
An infrastructure deployment can complete while the actual scenario still fails — typical reasons are role propagation delay, private endpoint approval lag, incomplete capability binding, or DNS records that have not propagated yet. Treat deployment completion as the start of validation, not the end. Use the [validation checklist](./validation-checklist.md).

### Customer environments add variation
These samples are references, not universal blueprints. Real environments add constraints: strict network policy, naming standards, pre-existing DNS architecture, centralized PE approval, subscription / RBAC boundaries, and policy enforcement that affects deployment order.

## Managed VNet-specific caveats

- **Lower visibility into agent-side network behavior.** Managed VNet is simpler, but customer visibility into the agent-side network path is lower than in BYO VNet — that is part of the trade-off.
- **Not the right choice for every regulated scenario.** If compliance requires agent compute to live inside the customer VNet, use BYO VNet.
- **Simpler ≠ trivial.** Managed VNet removes customer subnet planning, but you still need correct data-plane private access, correct RBAC (both phases), correct DNS, and end-to-end validation.

## BYO VNet-specific caveats

- **More moving parts.** Delegated subnet planning, IP capacity planning, more customer-owned network dependencies, and additional troubleshooting surface area.
- **Subnet sizing matters.** Undersized delegated subnets become an operational problem fast. `/24` gives headroom; `/26` is a hard floor.
- **Validation is more important.** Because more of the network path is customer-controlled, customer firewall / NSG behavior and DNS / route assumptions matter more. Check them early.
- **Treat newer paths cautiously.** If your BYO implementation is based on a documentation-derived baseline, validate it carefully in the target environment before presenting as production-ready.

## Production-readiness guidance

Before calling either sample production-ready for a customer scenario, confirm:

- the exact target region is supported for every preview feature in use
- the agent pattern (Standard / Hosted / Prompt) is supported in that region
- the networking model matches the customer's control requirements
- required roles can actually be assigned in the customer's subscription model
- private endpoints and DNS can be implemented within the customer's network architecture (especially if DNS is centrally managed)
- the end-to-end scenario has been validated with customer-like data and workflows

## When to stop guessing

If you hit repeated runtime failures after deployment success, inconsistent behavior across retries, DNS results that do not match the intended private path, or uncertainty about which identity is being used — stop. Run the [7 CLI checks](./validation-checklist.md#cli-verification--7-concrete-checks) and gather evidence before changing more code.

## Related docs

- [Shared data plane](./shared-data-plane.md)
- [Design rationale](./design-rationale.md)
- [capabilityHost, RBAC, and DNS](./capabilityhost-rbac-dns.md)
- [Validation checklist](./validation-checklist.md)
- [Managed VNet architecture](./architecture-diagrams/managed-vnet.md)
- [BYO VNet architecture](./architecture-diagrams/byo-vnet.md)
- [Side-by-side comparison](./architecture-diagrams/side-by-side.md)
