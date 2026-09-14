# Testplan — OSAC-1433 Unified Networking

## Overview

- **Feature:** OSAC-1433 — Unified Networking API for VMaaS, CaaS, and BMaaS
- **Source design:** [design.md](design.md)
- **Scope:** Shared networking resources, IPv4 field contracts, manager
  capabilities, lifecycle operations, defaulting semantics, dependency
  readiness, allocation transactionality, and cross-service interoperability.
- **Operation contract:** create, read/list, and delete only for network-owned
  fields. Network-owned update, patch, and replace operations are unsupported.
- **Excluded:** East-west networking is not implemented and is governed by its
  own design; this shared plan does not test its update or resize behavior.
  Unsupported in the current boundary: multi-interface, IPv6, dual-stack,
  multi-hub, air-gapped, and VN-peering behavior. These are negative-test
  cases, not supported scenarios.

## Execution strategy

The test plan uses three layers:

- **Unit:** fast deterministic validator, resolver, transaction-decision,
  controller-state, dispatcher-plan, and status-parser tests using fake
  repositories, Kubernetes clients, managers, and job responses.
- **Integration:** fulfillment-service with real ephemeral PostgreSQL,
  protovalidate, OPA/RBAC, public/private handlers, and an envtest or Kind
  cluster running real CRDs/admission/controllers. Manager and AAP/fabric
  behavior is simulated with contract-compatible fakes.
- **E2E:** the supported connected single-hub deployment with real
  authentication, OSAC components, the applicable manager, and dataplane
  connectivity checks. Both combined-manager and K8s-only/BM-only topologies
  are tested where their capability matrix declares them supported.

Every negative test verifies both the expected error/condition and the absence
of an invalid parent, child, allocation, backend operation, or orphan.

## Test cases

### R1: NetworkClass manager and capability resolution

#### TC-R1-01: Combined manager configuration is accepted

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration | critical | automated |

##### Preconditions

- Provider registers one fabric manager and one K8s manager.
- Both advertise the required IPv4 create/read/delete capabilities, and the
  Fabric Manager advertises NATGateway support for this combined-manager case.
- Provider inventory reports exactly one active hub and
  `connectivity_mode=connected`.

##### Steps

1. Create the deployment NetworkClass with both manager references.
2. Resolve a VirtualNetwork and Subnet provisioning plan.
3. Resolve the implementation strategy.

##### Expected results

- NetworkClass is accepted.
- The persisted NetworkClass identifies the single active hub and the
  deployment is admitted as connected.
- The persisted NetworkClass status reports `addressFamily=ipv4`, reports
  `natGateway=true` because the configured Fabric Manager supports it, and
  advertises exactly the manager-derived supported resources and operations.
- The manager combination covers the requested resource operations.
- The implementation strategy is derived from manager capabilities.
- The tenant cannot replace the derived strategy with an arbitrary value.

#### TC-R1-02: K8s-only, BM-only, and combined supported topologies are evaluated

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E | critical | automated |

##### Steps

1. Configure a K8s-only manager that advertises the complete supported VM and
   shared-resource surface except NATGateway.
2. Read the resolved NetworkClass status and verify the VM and shared-resource
   capability resolution.
3. Attempt CaaS BM-worker provisioning in K8s-only mode, including a topology
   that advertises NAT capability.
4. Configure a fabric-only manager and verify BMaaS and BM-only CaaS
   capability resolution when the required BM-worker, MetalLB, and reachability
   capabilities are advertised.
5. Attempt VM placement without a K8s manager.
6. Attempt a resource operation not advertised by the selected manager.

##### Expected results

- K8s-only VM/shared-resource behavior is accepted when advertised.
- K8s-only status reports `addressFamily=ipv4`,
  `natGateway=false`, and does not advertise NATGateway resources or
  operations.
- K8s-only CaaS BM-worker provisioning is rejected before Cluster, worker, or
  networking-resource persistence, regardless of NAT capability.
- Fabric-only BMaaS and BM-only CaaS behavior is accepted when all
  service-specific capabilities and reachability prerequisites are advertised.
- VM creation without a K8s manager is rejected before persistence.
- An unadvertised resource or operation is rejected before backend dispatch.

#### TC-R1-03: Invalid or tenant-controlled manager configuration is rejected

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration | critical | automated |

##### Cases

- neither manager is configured;
- manager is not provider-registered;
- manager registration exists but is disabled;
- required capability is missing;
- manager registration contains an unknown or arbitrary capability name;
- the deployment NetworkClass is absent, Pending, or Failed when a
  networking resource is created;
- the deployment offers the CaaS/MetalLB VIP path but lacks the required
  MetalLB prefix length;
- the MetalLB prefix is zero, outside the IPv4 prefix range, not more specific
  than the participating Subnet prefix, or reserves a range outside the
  Subnet;
- a non-CaaS deployment supplies `metallb_vip_prefix_length`;
- zero hubs, multiple active hubs, or an air-gapped connectivity report;
- a second active NetworkClass exists for the deployment;
- tenant supplies a NetworkClass, manager, implementation strategy, or
  provider-only capability override;
- tenant attempts to create NATGateway in a topology without NAT support.

##### Expected results

- The request is rejected with the documented authorization, validation, or
  failed-precondition status.
- A missing or non-Ready deployment NetworkClass returns
  `FailedPrecondition` before the resource or backend operation is persisted or
  dispatched.
- No networking resource or backend operation is created.

#### TC-R1-04: Provider-owned operations and scope are enforced

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E rejection | critical | automated where user-visible |

##### Cases

- Tenant attempts to create, update, patch, replace, or delete
  `NetworkClass`.
- Tenant attempts to create, update, patch, replace, or delete
  `ExternalIPPool`.
- Tenant attempts to read or list another tenant's pool or use it through a
  typed reference outside the provider-visible scope.
- Tenant attempts to get or list another tenant's VirtualNetwork, Subnet,
  SecurityGroup, ExternalIP, ExternalIPAttachment, NATGateway, or workload
  network attachment/status, including through a name filter, label/filter,
  status field, Catalog policy, or typed reference.
- Tenant references a provider-created pool that is explicitly visible to the
  tenant.

##### Expected results

- Provider-only mutations return the platform's authorization/visibility
  error before semantic validation or persistence.
- A visible provider pool may be read or referenced, but its family, CIDR,
  capacity, and lifecycle cannot be supplied or changed by the tenant.
- Cross-scope reads and references do not reveal the other tenant's resource.
- The same scope result applies to every network resource, attachment, and
  reference: the object is hidden or the request returns the platform's
  non-disclosing visibility error, and no list filter or status field reveals
  its identity or network values.

### R2: Shared resource fields and formats

#### TC-R2-01: Valid IPv4 resources and references are accepted

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration | critical | automated |

##### Steps

1. Create a Ready VirtualNetwork with canonical IPv4 CIDR.
2. Create a contained non-overlapping Subnet.
3. Create a SecurityGroup with a valid rule.
4. Create a Ready IPv4 ExternalIPPool with one CIDR.
5. Allocate separate, valid ExternalIPs and create an ExternalIPAttachment
   and NATGateway reference where supported.
6. Get and list every created resource, including the VirtualNetwork, Subnet,
   SecurityGroup, ExternalIPPool, ExternalIP, ExternalIPAttachment, and
   NATGateway where supported.

##### Expected results

- Every create accepts the documented field types and values.
- References are typed, same-scope, and Ready/Allocated before use.
- Tenant-scoped get/list calls return the caller's resources and complete
  network-owned fields; provider-visible pool reads follow the documented
  provider visibility rule.
- Resources with Ready semantics remain Pending until backend prerequisites
  complete and then become Ready; ExternalIP instead transitions from Pending
  to Allocated before it can be consumed.

#### TC-R2-02: Invalid formats and relationships are rejected

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E rejection | critical | automated |

##### Cases

- malformed IPv4 address or CIDR;
- IPv6 or dual-stack value;
- host bits set in a network CIDR;
- duplicate VirtualNetwork name in the same tenant/project scope;
- Subnet outside, equal to, or overlapping a sibling Subnet;
- wrong reference type, missing reference, cross-tenant/project reference;
- Pending, Failed, or non-Ready dependency;
- ExternalIPPool with `ip_family=UNSPECIFIED`, zero or multiple CIDRs, a CIDR
  outside the provider-permitted address space, or overlapping allocation
  ownership without an explicit disjoint-ownership declaration;
- duplicate ExternalIP consumer or second NATGateway for one VN;
- exact duplicate ExternalIPAttachment for the same ExternalIP, target, and
  endpoint;
- arbitrary tenant-selected ExternalIP address.
- manager-reported ExternalIP address outside the pool, equal to a network or
  broadcast address, reserved, IPv6, or already allocated;
- direct ExternalIPAttachment with a Pending/Failed ExternalIP or non-Ready
  target;

##### Expected results

- Invalid format/relationship returns `InvalidArgument`.
- Existing but unusable dependency returns `FailedPrecondition`.
- Caller-invalid creates leave no persisted object or backend operation.
- An invalid manager-reported allocation is recorded as a provisioning
  failure, never becomes `Ready`, and does not activate an attachment or leave
  an allocated address/consumer behind.

#### TC-R2-03: Cross-tenant VN CIDR overlap follows the documented isolation rule

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration | high | automated |

##### Expected results

- Overlapping VirtualNetwork CIDRs for different tenants are accepted where
  fabric isolation is the declared boundary.
- Subnet overlap within one VirtualNetwork remains rejected.
- Different VirtualNetworks remain isolated; the API does not implement VN
  peering or cross-VN routing.

#### TC-R2-04: ExternalIPPool readiness requires complete manager support

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration | critical | automated |

##### Steps

1. Configure a provider manager that advertises ExternalIPPool creation but
   omits `allocate`, `release`, or `report` support, one capability at a time.
2. Attempt to create a provider-scoped IPv4 ExternalIPPool.
3. Configure all three operations and make the manager return an invalid
   family, range, or lifecycle response during pool reconciliation.
4. Create two pools with overlapping allocation ranges, first without and then
   with an explicit manager declaration of disjoint allocation ownership.

##### Expected results

- A pool is not marked Ready unless allocate, release, and report operations
  are all advertised and usable.
- Invalid manager capability or lifecycle responses leave the pool Pending or
  Failed and prevent ExternalIP allocation.
- Overlapping pools are rejected unless the manager explicitly provides
  disjoint allocation ownership; accepted pools cannot allocate the same
  address.

### R3: SecurityGroup rules and deployment baseline

#### TC-R3-01: Rule fields and rule-set semantics are enforced

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E | critical | automated |

##### Cases

- valid allow/deny, ingress/egress, tcp/udp/icmp/any rule;
- required ports for TCP/UDP and omitted ports for ICMP/any;
- valid direction-specific canonical IPv4 CIDR;
- invalid action, direction, protocol, port range, or source/destination;
- duplicate normalized rule;
- conflicting equal-specificity rule;
- tenant-created empty rule list;
- onboarding-created fallback SecurityGroup with an empty list;
- authorized Tenant Admin replacement fallback SecurityGroup with an empty
  list after the old default is fully removed;
- ordinary tenant-created empty SecurityGroup;
- attempted configurable baseline action or tenant baseline override.

##### Expected results

- Ordinary tenant-created groups require at least one valid rule.
- The onboarding-created or authorized replacement fallback group is accepted
  empty only when it is the single validated tenant default; an ordinary
  tenant-created empty group is rejected.
- No tenant or resource field can change the deployment baseline from its
  hard-coded `permit` action.
- Invalid, duplicate, and conflicting rules are rejected before persistence.

#### TC-R3-02: Most-specific rule behavior is verified

| Test type | Priority | Automation |
|---|---|---|
| Unit, E2E | critical | automated |

##### Expected results

- Deployment-wide least-specific permit remains active.
- The most-specific matching tenant rule wins.
- Exact protocol outranks `any`; exact port outranks omitted port.
- The tenant fallback SecurityGroup is not confused with the deployment
  baseline rule.

### R4: Attachment defaulting and cardinality

#### TC-R4-01: Omitted, empty, partial, and complete attachments resolve correctly

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E | critical | automated |

##### Matrix

| Input | Expected result |
|---|---|
| Missing attachment field | All applicable defaults are resolved |
| Explicit empty list/message | Same default behavior as documented for the resource |
| Only Subnet supplied | Preserve Subnet; fill only SecurityGroups |
| Only SecurityGroups supplied | Preserve SecurityGroups; fill only Subnet |
| Complete attachment supplied | Preserve every supplied value |
| Explicit invalid supplied value | Reject; never repair with a default |

##### Expected results

- Catalog and Template precedence is resolved before tenant defaults.
- Direct tenant creates cannot persist a Pending dependency graph.
- All resolved references are Ready, same-scope, same-VirtualNetwork, and
  unique before persistence.

#### TC-R4-02: Typed local-reference wire format is enforced

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E rejection | critical | automated |

##### Cases

- `subnet` supplied as `{ "name": "subnet-a" }`;
- `subnet` supplied as `{ "id": "subnet-123" }` without `name`;
- `security_groups` supplied as `[{ "name": "web" }]`;
- `security_groups` supplied as `[{ "name": "web", "id": "sg-123" }]`;
- `security_groups` supplied as `[{ "id": "sg-123" }]` without `name`;
- both `id` and `name` supplied and resolving to the same resource;
- raw string values such as `"subnet-a"` or `["sg-123"]`;
- `id` and `name` supplied but resolving to different resources;
- typed Subnet/SecurityGroup references from a different tenant/project.

##### Expected results

- Local-reference objects require `name`; a name-only reference resolves and
  is canonicalized with the resolved `id`, while a supplied `id` is verified
  against the named resource before persistence.
- ID-only local references, raw strings, mismatched `id`/`name`, wrong
  reference types, and cross-scope references are rejected with a
  field-specific error.
- The same nested representation is used by Compute, Cluster, BMaaS, Catalog
  materialization, private ClusterOrder handoff, and direct CR validation.
- No invalid parent, child, worker request, or backend operation is created.

#### TC-R4-03: Service cardinality restrictions are enforced centrally

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E rejection | critical | automated |

##### Cases

- Compute list with more than one entry;
- BM list with more than one entry;
- Compute or BM sole entry with explicit `primary: false`;
- Cluster repeated/multi-attachment representation;
- duplicate SecurityGroup references;
- direct CR containing values rejected by the public API.

##### Expected results

- Rejection identifies the most specific field path.
- No template, worker, port-move, or backend operation is dispatched.

### R5: ExternalIP and NATGateway lifecycle

#### TC-R5-01: Allocation, readiness, and capacity are atomic

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E | critical | automated |

##### Steps

1. Provide multiple Ready IPv4 pools with different capacity.
2. Request auto external access.
3. Exhaust all pools and repeat the request.
4. Inject a failure after capacity reservation but before parent persistence.

##### Expected results

- Greatest-capacity pool is selected; equal-capacity selection is deterministic.
- Exhaustion returns an API error and persists no parent, child, or capacity
  reservation.
- Rollback releases capacity and leaves no orphan.
- Internal auto-provisioning children begin Pending and activate only after
  the target IP and ExternalIP are ready.

#### TC-R5-02: ExternalIPAttachment and NATGateway endpoint rules are enforced

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E | critical | automated |

##### Cases

- empty or multiply populated ExternalIPAttachment target oneof;
- Compute or BaremetalInstance target with `API` or `INGRESS` endpoint;
- Cluster target with `UNSPECIFIED` or an unsupported endpoint;
- target and endpoint combination that does not match the target type;
- direct attachment using a Pending, Failed, unallocated, or already
  consumed ExternalIP;
- direct attachment targeting a Pending, Failed, or non-Ready resource;
- caller-supplied, malformed, IPv6, out-of-subnet, or duplicate discovered
  endpoint status;
- NATGateway with a non-Ready VirtualNetwork or consumed ExternalIP;
- NATGateway with a Ready VirtualNetwork whose backend segment is absent or
  not ready.

##### Expected results

- Compute/BM attachments use `UNSPECIFIED` endpoint.
- Cluster attachments use exactly `API` or `INGRESS`.
- Target oneof has exactly one arm.
- NATGateway requires a Ready VN and unconsumed Allocated ExternalIP.
- SNAT is not dispatched until the referenced VN segment exists and is ready.
- Malformed manager SNAT/DNAT status never produces Ready.
- Every invalid target, endpoint, dependency, or discovered endpoint is
  rejected or fails closed before DNAT/SNAT dispatch and persistence.

#### TC-R5-03: Automatic external access is disabled by omission or false

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E | high | automated |

##### Steps

1. Create a VM, Cluster, and BaremetalInstance with the automatic external
   access field omitted.
2. Create equivalent resources with `auto_external_ip_attachment=false`.
3. Inspect ExternalIP/ExternalIPAttachment resources, pool capacity, and
   backend allocation calls.

##### Expected results

- Omission and explicit `false` both succeed when the ordinary network
  attachment is valid.
- No automatic ExternalIP or ExternalIPAttachment is created, no pool
  capacity is reserved, and no allocation or DNAT backend operation is
  dispatched.
- Changing the switch after creation is rejected by the shared immutability
  rule.

### R6: Create/read/delete-only operations and dependency guards

#### TC-R6-01: Network-owned updates are rejected everywhere

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E rejection | critical | automated |

##### Cases

- API update, patch, replace, and field-mask mutation on every tenant-facing
  network resource: VirtualNetwork, Subnet, SecurityGroup, ExternalIP,
  ExternalIPAttachment, and NATGateway;
- nested Subnet, SecurityGroup, CIDR, rule, pool/allocation identity,
  interface, primary, endpoint, target, list-length/order, and
  auto-external mutations on ComputeInstance, Cluster, and
  BaremetalInstance;
- direct hub-CR mutation;
- status write attempting to mutate spec.

##### Expected results

- Every network-owned mutation is rejected.
- Controller may change only status, conditions, timestamps, discovered IPs,
  and finalizers.
- Delete/recreate is the only supported network-spec change path.

#### TC-R6-02: Leaf-first deletion and finalizers protect dependencies

| Test type | Priority | Automation |
|---|---|---|
| Integration, E2E | critical | automated |

##### Cases

- Delete or release an ExternalIP while it is consumed by an
  ExternalIPAttachment.
- Delete or release an ExternalIP while it is consumed by a NATGateway.

##### Expected results

- Parent deletion is blocked while children or reverse references exist.
- Auto-created ExternalIPAttachment is deleted before ExternalIP.
- ExternalIP is deleted before ExternalIPPool.
- ExternalIPPool deletion is blocked while any ExternalIP still references it.
- ExternalIP deletion/release is rejected while an attachment or NATGateway
  consumes it; no manager release operation is dispatched.
- Subnets, SecurityGroups, and NATGateway are gone before VN deletion.
- SecurityGroup deletion is blocked by workload attachment references and
  governed Catalog policy references.
- NetworkClass deletion is blocked while any dependent networking resource,
  workload attachment, or manager integration remains.
- VN is deleted before any provider VPC/backend parent.
- A manually created ExternalIPAttachment blocks deletion of its target until
  the tenant deletes the attachment; it is never implicitly detached.
- Concurrent deletion is idempotent and does not orphan backend state.

### R7: Direct CR bypass, state machine, and recovery

#### TC-R7-01: Admission and controllers enforce the same contract as the API

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration | critical | automated |

##### Expected results

- Direct CRs with invalid cardinality, IPv6, bad references, unsupported
  operations, or false Ready status are rejected or fail closed.
- Every persisted networking resource, including VirtualNetwork, Subnet,
  SecurityGroup, ExternalIPPool, ExternalIP, ExternalIPAttachment, and
  NATGateway, carries `status.hub` equal to the single active deployment hub.
- A missing or mismatched `status.hub` cannot be used to report a resource
  Ready or dispatch a backend operation.
- Resource-specific terminal transitions are enforced only through the
  documented reconciliation path: ExternalIP uses `Pending -> Allocated` or
  `Pending -> Failed`, while resources with Ready semantics use
  `Pending -> Ready` or `Pending -> Failed`.
- Attempts to force `Ready -> Pending`, `Allocated -> Pending`, or
  `Failed -> Ready` through a caller or an
  unauthorized controller path are rejected or ignored, and do not dispatch
  a backend operation.
- Pending dependencies requeue; terminal failures remain Failed.
- Restarting controllers does not duplicate jobs, allocations, rules,
  segments, or finalizers.

### R8: Cross-service interoperability

#### TC-R8-01: VMaaS, CaaS, and BMaaS consume a shared Ready Subnet

| Test type | Priority | Automation |
|---|---|---|
| Integration, E2E | critical | automated |

##### Expected results

- VMaaS uses one virtual interface.
- CaaS uses one cluster attachment and BM worker enrichment.
- BMaaS uses one physical tenant attachment.
- All services preserve shared IPv4, same-VN, readiness, SecurityGroup, and
  create/read/delete-only rules.
- Multiple hosting clusters receive the required overlay for a shared Subnet
  when the K8s manager advertises that capability.
- Workloads in different VNs remain isolated.

### R9: Explicitly unsupported surface

#### TC-R9-01: Unsupported shared behavior is rejected or excluded

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E rejection | critical | automated where user-visible |

##### Cases

- IPv6 or dual-stack;
- multi-CIDR pool;
- multi-interface VM/BM or multi-NIC tenant attachment;
- VN peering/cross-VN routing;
- multi-hub or air-gapped deployment;
- tenant-selected provider implementation or arbitrary IP;
- unsupported NATGateway capability;
- unsafe parent deletion;
- a configurable deployment baseline action or tenant baseline override.

##### Expected results

- Unsupported user-visible requests fail with the documented error.
- Provider-only or future behavior is not advertised as a capability and does
  not dispatch a backend operation.

## Graduation gate

- Every normative shared validation rule maps to a unit or integration test.
- Every supported user workflow maps to an E2E test.
- Every user-visible unsupported workflow maps to an E2E rejection test.
- Negative tests verify no partial persistence or backend side effect.
- Concurrent create/delete, controller restart, manager failure, and cleanup
  tests pass.
