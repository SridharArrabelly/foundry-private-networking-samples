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

## Failure patterns to watch for

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
- [capabilityHost, RBAC, and DNS](./capabilityhost-rbac-dns.md)
- [Known limitations](./known-limitations.md)
- [Managed VNet architecture](./architecture-diagrams/managed-vnet.md)
- [BYO VNet architecture](./architecture-diagrams/byo-vnet.md)
