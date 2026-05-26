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

The Windows jumpbox runs **PowerShell 5.1** by default. Scripts copied from a dev machine that use `?.` (null-conditional), `??` (null-coalescing), or ternary `a ? b : c` will fail to parse. Either install PowerShell 7 on the jumpbox first, or use 5.1-compatible syntax.

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
