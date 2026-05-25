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

## Why capabilityHost matters so much

It is the key design choice in this repo family. Once the project is bound to the data resources:

- the runtime knows which data resources it is expected to use
- dependent access patterns become meaningful
- RBAC requirements become concrete
- private connectivity requirements become visible

A lot of the rest of the deployment flows from this one decision.

## RBAC: why sequencing matters

RBAC is not just about assigning roles. The timing matters.

In practice, private Foundry deployments behave like a two-phase chain:

1. **Pre-binding / pre-readiness** — some roles must exist early so the project and related identities can reference or prepare the dependent resources
2. **Post-binding / post-readiness** — once the project and capability binding exist, additional access may be required for the runtime path to work end-to-end

That is why a template can appear to deploy successfully but still fail functionally if role assignments are incomplete, late, or attached to the wrong identity.

## Practical RBAC guidance

When debugging RBAC, confirm all of the following:

- the correct identity is being granted access
- the role is assigned at the correct scope
- the assignment exists before the dependent operation runs
- propagation time has been allowed for new assignments
- there are no duplicate or conflicting assignments that make troubleshooting harder

## Common RBAC symptoms

Typical signs of RBAC problems:

- resource deployment succeeds, but runtime calls fail
- AI Search lookups do not work even though the service exists
- Storage access fails for file operations
- Cosmos-based state persistence fails
- post-provision steps succeed intermittently or only after rerunning

## Private endpoints: readiness matters

Private endpoints are not useful just because they were requested. You need to confirm:

- the private endpoint exists
- it is approved if approval is required
- the target resource shows the expected private connectivity state
- DNS resolves the service name to the expected private address path

Until that chain is complete, the runtime may behave like the resource is unavailable.

## DNS: why it breaks otherwise healthy deployments

DNS is one of the most common reasons a private deployment fails after the infrastructure deploys.

A typical failure pattern looks like this:

- the resource exists
- the private endpoint exists
- the role assignments look correct
- but name resolution still points to a public path or fails entirely

In a private deployment, correct name resolution is part of the data path.

## DNS checks to make first

When validating DNS, check:

- the correct private DNS zones exist
- the right links are attached to the right virtual networks
- records were created for the private endpoints you expect
- the resolution path from the relevant runtime environment returns the expected result

If name resolution is wrong, the rest of the setup can appear broken even when the infrastructure is mostly correct.

## How to think about the dependency chain

A useful mental model is:

1. deploy the data resources
2. deploy private connectivity
3. establish capability binding
4. assign access to the right identities
5. confirm DNS resolution
6. validate the scenario end-to-end

If one part is missing, the symptom may show up somewhere else. Example:

- a DNS issue may look like a runtime issue
- an RBAC issue may look like a Search or Storage issue
- a capability binding issue may look like a generic agent failure

## Managed VNet vs BYO VNet

The same concepts apply in both samples, but the network path differs.

### Managed VNet
- The project uses a Microsoft-managed network boundary for agent compute
- The data plane still needs correct binding, access, and resolution

### BYO VNet
- The project uses customer-controlled network placement for agent compute
- The same binding and access concepts apply, plus customer subnet and network path considerations

## Troubleshooting order

When something is not working, use this order:

1. Confirm the resources exist
2. Confirm private endpoints are ready
3. Confirm DNS resolution
4. Confirm capabilityHost was created and bound as expected
5. Confirm RBAC on the actual identities in use
6. Re-run end-to-end validation

## What this page is not

This page is not meant to replace product documentation or show every role and every DNS zone in prose. Its purpose is to explain why these moving parts exist and why failures often cluster around them.

## Related docs

- [Shared data plane](./shared-data-plane.md)
- [Validation checklist](./validation-checklist.md)
- [Known limitations](./known-limitations.md)
- [Managed VNet architecture](./architecture-diagrams/managed-vnet.md)
- [BYO VNet architecture](./architecture-diagrams/byo-vnet.md)
