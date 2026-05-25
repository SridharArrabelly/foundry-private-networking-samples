# Known Limitations

This page captures the main caveats, constraints, and reality checks for this repo family.

Use it before positioning the samples as a production baseline.

## Read this page the right way

These are not reasons to avoid the samples. They are reasons to validate deliberately.

The repo family is intended to be a practical reference and starting point, not a promise that every customer environment will behave the same way without adaptation.

## Repo-family-wide limitations

These apply to both samples.

### 1. Platform behavior can evolve
Azure AI Foundry and Agent Service capabilities continue to change over time.

That means:

- supported regions can change
- networking behavior can evolve
- required RBAC details can shift
- preview-to-GA transitions may affect guidance and implementation detail

Always re-check platform documentation for the exact scenario you are targeting.

### 2. Private networking failures are often indirect
In private deployments, the visible failure is not always the root cause.

Examples:

- a DNS issue can look like an agent failure
- an RBAC issue can look like a Search failure
- a binding issue can look like a runtime issue

Plan for validation and troubleshooting, not just deployment success.

### 3. Successful deployment does not guarantee functional readiness
An infrastructure deployment can complete successfully while the actual scenario still fails.

Reasons often include:

- role propagation delay
- private endpoint readiness lag
- incomplete capability binding
- missing or incorrect DNS resolution

Treat deployment completion as the start of validation, not the end.

### 4. Customer environments add variation
These samples are references, not universal blueprints. Real customer environments can add constraints such as:

- strict network policy
- naming standards
- pre-existing DNS architecture
- centralized private endpoint approval workflows
- subscription or RBAC boundaries
- policy enforcement that affects deployment order

## Managed VNet-specific limitations

### 1. Lower customer visibility into agent-side network behavior
Managed VNet is simpler, but customer visibility into the agent-side network path is lower than in BYO VNet. That trade-off is part of why it is simpler operationally.

### 2. Not the right choice for every regulated scenario
If compliance requires agent compute to live inside the customer VNet, Managed VNet is the wrong pattern. Use BYO VNet instead.

### 3. Simpler does not mean trivial
Managed VNet removes customer subnet planning, but it does not remove the need for:

- correct data-plane private access
- correct RBAC
- correct DNS
- end-to-end validation

## BYO VNet-specific limitations

### 1. More moving parts
BYO VNet gives more customer control, but it adds more operational complexity. Typical added complexity includes:

- delegated subnet planning
- IP capacity planning
- more customer-owned network dependencies
- additional troubleshooting surface area

### 2. Subnet sizing matters
Undersized delegated subnets can become an operational problem. Do not treat subnet planning as an afterthought.

### 3. Validation is more important
Because more of the network path is customer-controlled, environment-specific differences matter more. That means:

- stricter validation is required
- customer firewall / NSG behavior matters more
- DNS and route assumptions should be checked early

### 4. Treat newer paths cautiously
If your BYO implementation is based on a documentation-derived baseline, validate it carefully in the target environment before presenting it as production-ready.

## Production-readiness guidance

Before calling either sample production-ready for a customer scenario, confirm:

- the exact target region is supported
- the exact agent pattern is supported
- the networking model matches the customer's control requirements
- required roles can actually be assigned in the target subscription model
- private endpoints and DNS can be implemented within the customer's network architecture
- the end-to-end scenario has been validated with customer-like data and workflows

## When to escalate or deepen validation

Do not keep guessing if you hit any of these situations:

- repeated runtime failures after deployment success
- inconsistent behavior across retries
- DNS results that do not match the intended private path
- uncertainty about which identity is actually being used
- customer network policy that differs significantly from the sample assumptions

At that point, gather evidence and validate systematically.

## Best way to use these samples

Use the samples as:

- a reference architecture
- a validation baseline
- a conversation accelerator with customers
- a starting point for adaptation

Do not use them as:

- a guarantee that every enterprise network will behave the same way
- a substitute for customer-specific validation
- a replacement for current product documentation

## Related docs

- [Shared data plane](./shared-data-plane.md)
- [capabilityHost, RBAC, and DNS](./capabilityhost-rbac-dns.md)
- [Validation checklist](./validation-checklist.md)
- [Managed VNet architecture](./architecture-diagrams/managed-vnet.md)
- [BYO VNet architecture](./architecture-diagrams/byo-vnet.md)
- [Side-by-side comparison](./architecture-diagrams/side-by-side.md)
