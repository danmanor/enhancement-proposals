---
title: default-networking
authors:
  - dmanor@redhat.com
creation-date: 2026-07-08
last-updated: 2026-09-24
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-1433
prd: "prd.md"
see-also:
  - "/enhancements/OSAC-1433-unified-networking"
  - "/enhancements/OSAC-1435-vmaas-networking"
  - "/enhancements/OSAC-1436-caas-networking"
  - "/enhancements/OSAC-1437-bmaas-networking"
replaces:
  - N/A
superseded-by:
  - N/A
---

# Default Networking — Simplified Resource Creation

At tenant onboarding, default networking provisions an IPv4 VirtualNetwork,
its system-created default NetworkACL, a Subnet associated with that ACL,
and a NATGateway. Every other VirtualNetwork also receives one
system-created default NetworkACL, but no Subnet is created automatically.
A Subnet create request that omits an ACL resolves to the default ACL in that
same VirtualNetwork; an explicit ACL is preserved after validation. Workload
attachments can use the tenant's default Subnet, auto ExternalIP provisioning
is available, and auto-created external-access resources are cleaned up on
deletion. IPv6 and dual-stack networking are not supported. Networking
resources use read, create, and delete operations; NetworkACL rules, Subnet
associations, address configuration, and workload attachments are immutable
after creation.

The current workload contract is at most one tenant network attachment for
VMaaS, BMaaS, and CaaS. VMaaS and BMaaS keep their plural
`network_attachments` fields for API compatibility and reject more than one
entry; CaaS keeps its singular `network_attachment` field. When one
attachment is present it is implicitly the primary/default route. The BMaaS
attachment's existing optional `primary` field may be omitted or set to true;
`primary: false` is rejected. VMaaS has no primary field and CaaS has no
primary concept.

## Summary

This document expands the [Unified Networking EP](/enhancements/OSAC-1433-unified-networking/design.md)
with default networking automation and simplified resource creation. It
inherits the [Unified Networking deployment support boundary](/enhancements/OSAC-1433-unified-networking/design.md#deployment-support-boundary):
default networking supports connected deployments only. At tenant creation,
the system provisions a default VirtualNetwork, its default NetworkACL, an
IPv4 Subnet associated with that ACL, and a NATGateway based on NetworkClass
configuration. Every other tenant VirtualNetwork also receives its own
default NetworkACL; the tenant default Subnet is created only in the default
VirtualNetwork. ComputeInstance, Cluster, and BaremetalInstance resources can
omit their network attachment field and use the tenant default Subnet. Auto
ExternalIP modes enable fully connected resources in one API call. See the
[PRD](prd.md) for requirements.

Default networking also inherits the [Unified Networking hub support
boundary](/enhancements/OSAC-1433-unified-networking/design.md#networking-hub-support-boundary):
OSAC networking supports exactly one provider-owned hub per deployment.
Multi-hub networking placement, cross-hub resource coordination, and
cross-hub network connectivity are unsupported. This boundary applies only to
the networking area and does not define hub behavior for other OSAC areas.
Multiple hosting/workload clusters remain supported where a networking feature
explicitly specifies them.

## Motivation

A reachable resource in OSAC requires networking resources: VirtualNetwork, NetworkACL, Subnet, the resource itself, ExternalIP, and ExternalIPAttachment. Default networking eliminates this friction — a single create command produces a reachable instance by leveraging tenant defaults provisioned at onboarding.

### Goals

- Single-call resource creation with sensible networking defaults
- Default networking resources (VN, IPv4 Subnet, NetworkACL, NATGateway) provisioned at tenant onboarding
- Optional resource-specific network attachment field on all resource types, with at most one tenant attachment per workload
- Auto ExternalIP mode for inbound connectivity
- Auto-cleanup of auto-created resources on deletion
- Tenant-scoped default resources (visible and managed through the unified
  read/create/delete lifecycle; ACL rules and Subnet association are fixed at creation)

### Non-Goals

- Custom default configurations per tenant (all tenants receive the same defaults)
- Auto-provisioning of additional VirtualNetworks or Subnets beyond the initial default
- UI support for simplified creation (deferred — API and CLI only)
- Automatic migration of existing resources to use defaults

## Proposal

The design covers three capabilities: default networking (including NATGateway) at tenant onboarding, optional resource-specific network attachment fields with auto-population, and auto ExternalIP provisioning.

### Workflow Description

#### Default Networking at Tenant Onboarding

1. **Cloud Infrastructure Admin creates the NetworkClass with defaults:**
   The NetworkClass is created with the deployment-wide IPv4 CIDRs and
   provider-default ingress and egress NetworkACL rules before tenant
   onboarding. Every VirtualNetwork receives one default ACL from these rules.
   Each default ACL's ingress and egress rule sets include an `ALLOW ALL`
   catch-all for `0.0.0.0/0`; more-specific matching rules in the NetworkClass
   take precedence based on their CIDR, protocol, and port match fields. The ACL
   remains stateless, so each direction is evaluated independently; reply
   traffic passes only when the reverse-direction evaluation allows it.
   NetworkClass changes follow the unified create/read/delete contract and
   require replacement.

   **NetworkClass replacement lifecycle:** `VirtualNetwork.spec.network_class`
   is required and immutable. The old NetworkClass cannot be deleted while any
   VirtualNetwork references it; reverse-reference checks must block that
   delete. Only one NetworkClass may exist per deployment, so v1 cannot create
   the replacement alongside the old class and does not support a seamless
   transition. Replacing it requires a disruptive maintenance cutover:

   1. Validate the replacement configuration offline and schedule downtime.
   2. Pause tenant onboarding, default-based creates, and tenant network and
      workload changes across every affected tenant.
   3. Drain workloads and delete dependent networking resources in their
      required deletion order, including NATGateways, Subnets, NetworkACLs,
      and VirtualNetworks. Delete the old NetworkClass only after all
      VirtualNetwork references are gone.
   4. Create the replacement as the deployment's sole NetworkClass. Recreate
      tenant VirtualNetworks, NetworkACLs, Subnets, NATGateways, and workload
      attachments; recreate workloads that need the replacement network.
   5. Resume tenant and default-based creates only after the replacement
      defaults and affected tenant networking resources are READY.

   The ExternalIPPool does not reference NetworkClass in this design and is
   not rebound by this transition.

2. **Cloud Provider Admin creates Tenant:**
   ```bash
   osac create tenant --name acme-corp
   ```
   - **fulfillment-service** creates Tenant record, then creates default networking resources through its own API (same path as tenant-created resources — persisted in PostgreSQL, reconciled to K8s CRs):
     - Creates default VirtualNetwork with label `osac.openshift.io/default: "true"`, using CIDR from NetworkClass defaults
     - The system creates the VirtualNetwork's default NetworkACL with label
       `osac.openshift.io/default: "true"`, using NetworkClass ingress and
       egress rules. It does the same for every tenant-created VirtualNetwork.
     - Creates default IPv4 Subnet with label
       `osac.openshift.io/default: "true"`, using `ipv4SubnetCIDR` from
       NetworkClass defaults. The request omits `spec.network_acl`; Subnet
       creation resolves and stores this VirtualNetwork's default ACL reference.
     - Creates default NATGateway with an auto-allocated ExternalIP on the default VirtualNetwork, labeled `osac.openshift.io/default: "true"`
   - Reads NetworkClass defaults configuration (single NetworkClass per deployment)
   - Default resources go through the normal reconciliation path: fulfillment-service reconciler pushes CRs → osac-operator networking controllers dispatch to configured networking managers → resources transition to READY
- fulfillment-service tracks default networking readiness on the Tenant: sets `DefaultNetworkingReady` condition once the default VN, its default NetworkACL, IPv4 Subnet and resolved association, and NATGateway are READY (via feedback)
   - Tenant overall status becomes READY only when DefaultNetworkingReady condition is true

3. **If default networking provisioning fails:**
   - Tenant remains in non-READY state
   - Tenant status condition shows: `DefaultNetworkingReady: false, reason: SubnetProvisioningFailed, message: "Subnet 'default' failed to provision"`
   - Cloud Provider Admin inspects failure, fixes root cause, and retries by deleting and re-creating the tenant

#### Shared Attachment Resolution

Default Networking uses the same presence and field-level defaulting rules as
the [Unified Networking attachment contract](/enhancements/OSAC-1433-unified-networking/design.md#attachment-presence-and-defaulting):

- An omitted attachment, an empty attachment list, or an empty CaaS attachment
  message requests the tenant defaults.
- For VMaaS and CaaS, an omitted Subnet resolves to the tenant's default
  Subnet. Its associated NetworkACL supplies traffic policy. If an attachment
  names a Subnet, the server preserves that Subnet and validates that it and
  its associated ACL are READY; it does not substitute the tenant default.
  For BMaaS, the first `fabric` port from `BareMetalInstanceType.network_ports`
  is defaulted as well.
- A single supplied attachment is completed field-by-field. A missing Subnet
  receives the default Subnet; BMaaS also defaults a missing interface to the
  first `fabric` port. The attachment does not contain an ACL reference.
  Supplied values are never replaced.
- Every Subnet has exactly one associated NetworkACL in its VirtualNetwork.
  A Subnet created without an ACL uses that VirtualNetwork's default ACL; an
  explicit single ACL is validated and stored. The Subnet's policy applies
  uniformly to workloads attached to it.
- The fully resolved attachment is stored with the workload and is immutable
  after creation. A missing or non-Ready Subnet or associated ACL causes
  creation to fail.

#### Simplified Resource Creation with Defaults

4. **Tenant User creates VM without networking parameters:**
   ```bash
   # No network_attachments specified
   osac create computeinstance --template ocp_virt_vm --name my-vm
   ```
   - fulfillment-service:
     - Detects `network_attachments` field is omitted or empty
     - Queries the tenant's default Subnet (labeled `osac.openshift.io/default: "true"`)
     - Populates the resource-specific network attachment field with the default Subnet
     - For a supplied single attachment, defaults only a missing Subnet and, for BMaaS, a missing interface; the selected Subnet's associated NetworkACL supplies policy
     - Stores resolved attachments in spec
   - Creates ComputeInstance CR with resolved network_attachments
   - osac-operator reconciles normally (VM provisioned on default subnet)

5. **Tenant User retrieves resource and sees resolved defaults:**
   ```bash
   osac get computeinstance my-vm -o yaml
   ```
   Output shows:
   ```yaml
   spec:
     network_attachments:
       - subnet: "default-subnet-id"
   ```

#### Auto ExternalIP for Single-Call Inbound Connectivity

6. **Tenant User creates VM with auto ExternalIP:**
   ```bash
   osac create computeinstance --template ocp_virt_vm \
     --external-ip-attachment --name my-vm
   ```
   - fulfillment-service (synchronous, during create API call):
     - Populates the resource-specific network attachment field with defaults (if omitted)
     - Reads `auto_external_ip_attachment: true`
     - Auto-selects an IPv4 ExternalIPPool (READY, most available capacity)
     - Creates ExternalIP in the same DB transaction as the workload. ExternalIP starts in **Pending** state. Pool capacity is decremented atomically.
     - ExternalIP labeled `osac.openshift.io/auto-created: "true"` and `osac.openshift.io/auto-created-for: <resource-id>`.
     - ExternalIPAttachment is **not** created at this point — its dependencies (ExternalIP Allocated + target Ready) are not yet met.
   - fulfillment-service (asynchronous, internal reconciler):
     - osac-operator reconciles ExternalIP → fabric manager allocates address → ExternalIP transitions to Allocated
     - osac-operator reconciles ComputeInstance → VM provisioning → ComputeInstance transitions to Ready
     - Once ExternalIP is Allocated AND ComputeInstance is Ready: fulfillment-service internal reconciler creates ExternalIPAttachment (readiness gate satisfied). ExternalIPAttachment labeled `osac.openshift.io/auto-created: "true"`.
     - osac-operator ExternalIPAttachment controller creates DNAT rule → ExternalIPAttachment transitions to Ready
   - See [Unified Networking — Auto-provisioning lifecycle](/enhancements/OSAC-1433-unified-networking/design.md#auto-provisioning-lifecycle-auto_external_ip_attachment) for the full stepped flow
   - Result: VM is reachable via ExternalIP

7. **If ExternalIPPool has no capacity:**
   - fulfillment-service returns error: `ExternalIPPool exhaustion: no available capacity in any READY pool for IPv4`
   - Resource is NOT persisted
   - Tenant User must use explicit ExternalIP allocation from another pool or contact admin

#### Auto ExternalIP for Clusters (Prerequisite Ordering)

8. **Tenant User creates Cluster with auto ExternalIP for API and ingress:**
   ```bash
   osac create cluster --template ocp_4_17_small \
     --external-ip-attachment --name my-cluster
   ```
   - fulfillment-service (synchronous, during create API call):
     - Populates the resource-specific network attachment field with defaults (if omitted)
     - Reads `auto_external_ip_attachment: true`
     - Auto-selects ExternalIPPool (same algorithm)
     - Creates two ExternalIPs in the same DB transaction as the Cluster. Both start in **Pending** state. Pool capacity decremented atomically.
     - Both labeled `osac.openshift.io/auto-created: "true"` and `osac.openshift.io/auto-created-for: <cluster-id>`.
     - ExternalIPAttachments are **not** created at this point.
   - fulfillment-service (asynchronous, internal reconciler):
     - osac-operator ExternalIP controller dispatches to fabric manager → ExternalIPs transition to Allocated
     - Cluster provisioning proceeds — MetalLB allocates internal VIPs. Template discovers VIPs → ClusterOrder status → feedback controller → Cluster status. Cluster transitions to Ready.
     - Once ExternalIPs are Allocated AND Cluster is Ready: fulfillment-service internal reconciler creates two ExternalIPAttachments (one for API, one for ingress). Readiness gate satisfied.
     - ExternalIPAttachment controllers create DNAT: external IP → internal VIP → Ready
   - See [Unified Networking — Auto-provisioning lifecycle](/enhancements/OSAC-1433-unified-networking/design.md#auto-provisioning-lifecycle-auto_external_ip_attachment) for the full stepped flow
   - Result: Cluster is reachable via ExternalIPs for both API and ingress

9. **CLI flag mapping for clusters:**
   - `--external-ip-attachment` → `auto_external_ip_attachment: true` (auto-provision ExternalIP + ExternalIPAttachment for both API and ingress)

#### Auto-Cleanup on Deletion

10. **Tenant User deletes resource with auto-created ExternalIP:**
    ```bash
    osac delete computeinstance my-vm
    ```
    - If manually-created ExternalIPAttachments target this resource, the
      delete is **rejected** — the tenant must remove them first. See
      [Unified Networking — Deletion Dependency Guards](/enhancements/OSAC-1433-unified-networking/design.md#deletion-dependency-guards).
    - If only auto-created ExternalIPAttachments exist (or none), the
      delete proceeds. osac-operator ComputeInstance controller finalizer:
      - Queries ExternalIPAttachment and ExternalIP labeled `osac.openshift.io/auto-created: "true"` referencing this ComputeInstance
      - Deletes ExternalIPAttachment first (DNAT rule removed)
      - Deletes ExternalIP second (IP returned to pool)
      - If cleanup fails permanently (after retries): finalizer is removed, parent resource deleted, orphaned resources left in cluster
    - **Manually created resources are NOT cleaned up** — if tenant created ExternalIP/ExternalIPAttachment explicitly (not labeled auto-created), they persist after parent deletion
    - **Default networking resources (VN, Subnet, NetworkACL, NATGateway) are NOT cleaned up** — they are tenant-scoped and shared across resources

11. **Tenant Admin inspects default resources:**
    ```bash
    # List default resources
    osac get virtualnetworks --filter 'labels["osac.openshift.io/default"]="true"'
    osac get subnets --filter 'labels["osac.openshift.io/default"]="true"'
    osac get network-acls --filter 'labels["osac.openshift.io/default"]="true"'
    osac get natgateways --filter 'labels["osac.openshift.io/default"]="true"'

    # Changes require creating replacement networking resources after
    # dependencies on the defaults have been removed.
    ```
    - Default resources follow the unified read/create/delete contract.
      NetworkACL rules, the Subnet's NetworkACL association, and
      VirtualNetwork/Subnet address configuration are immutable after creation.
      A default ACL cannot be deleted directly; it is removed with its
      VirtualNetwork.
    - Default resources cannot be deleted while any resource depends on them
      (subnet deletion is blocked if VMs reference it).
    - The default Subnet cannot be reassociated in place. Replacing its policy
      follows the coordinated process in [Default Resource Lifecycle](#default-resource-lifecycle),
      including every Subnet that references the shared ACL and its dependent
      workloads.
    - Replacing the default address/resource set (VirtualNetwork, Subnet,
      and NATGateway) is a coordinated transition: pause
      default-based creates for every affected existing tenant, drain or delete
      workloads attached to the old defaults, and run reverse-reference checks
      before deleting any old resource. The old VirtualNetwork must have no
      Subnet, custom NetworkACL, or NATGateway references; its system-created
      default ACL is removed as part of VirtualNetwork deletion.
      The old NATGateway must have no remaining dependents; deleting it releases
      its auto-allocated ExternalIP back to the pool through the normal
      ExternalIP cleanup path. A replacement NATGateway receives a newly
      allocated ExternalIP; the old address is not rebound. Create each
      replacement with the same tenant scope and
      `osac.openshift.io/default: "true"` label. Attachments are immutable, so
      existing workloads are not rebound and the replacement applies to later
      creates only.
    - Defaults cannot be unlabeled in place because networking metadata is
      immutable. Tenant-default workload selection must find exactly one
      active, READY resource of the requested default type. Subnet ACL
      defaulting must find exactly one active, READY default ACL within the
      Subnet's VirtualNetwork; each tenant may have multiple default ACLs,
      one per VirtualNetwork. Zero or multiple matches within the required
      scope is a configuration error. Default-based creates remain paused
      until the replacement VirtualNetwork, its default ACL, Subnet and
      resolved association, and NATGateway are READY.

### API Extensions

#### Proto (fulfillment-service)

**NetworkClass defaults configuration:**

```protobuf
message NetworkClassSpec {
  // ... existing fields ...
  NetworkDefaults defaults = 10; // new field
  int32 metallb_vip_prefix_length = 11; // e.g., 28 — reserves sub-range of each subnet for MetalLB VIPs (CaaS)
}

message NetworkDefaults {
  string virtual_network_cidr = 1;  // e.g., "10.0.0.0/16"
  string ipv4_subnet_cidr = 2;      // e.g., "10.0.1.0/24"
  // Each rule set includes an ALLOW ALL catch-all for 0.0.0.0/0;
  // more-specific matching rules take precedence.
  repeated NetworkACLRule ingress_rules = 3;
  repeated NetworkACLRule egress_rules = 4;
}

message NetworkACLRule {
  NetworkACLAction action = 1; // ALLOW or DENY
  Protocol protocol = 2;       // ALL, TCP, UDP, or ICMP
  optional int32 port_from = 3; // optional destination port range; TCP/UDP only
  optional int32 port_to = 4;
  string ipv4_cidr = 5;        // ingress source or egress destination
}
```

**Resource-level auto external access fields:**

```protobuf
// ComputeInstance
message ComputeInstanceSpec {
  // ... existing fields ...
  bool auto_external_ip_attachment = 19;  // auto-provision ExternalIP + ExternalIPAttachment
}

// BaremetalInstance
message BareMetalInstanceSpec {
  // ... existing fields ...
  bool auto_external_ip_attachment = 9;  // auto-provision ExternalIP + ExternalIPAttachment
}

// Cluster
message ClusterSpec {
  // ... existing fields ...
  bool auto_external_ip_attachment = 10;  // auto-provision ExternalIP + ExternalIPAttachment for API and ingress
}
```

**Default label on auto-created resources:**

The tenant default VirtualNetwork, its default Subnet, its default NetworkACL,
and the default NATGateway receive label `osac.openshift.io/default: "true"`.
Each other system-created default NetworkACL receives the same label. Default
ACL uniqueness is scoped to its VirtualNetwork, so a tenant may see one default
ACL per VirtualNetwork.
```yaml
metadata:
  labels:
    osac.openshift.io/default: "true"
```

All auto-created resources (ExternalIP, ExternalIPAttachment created by auto_external_ip_attachment=true) receive label:
```yaml
metadata:
  labels:
    osac.openshift.io/auto-created: "true"
```

#### Operator CRD (osac-operator)

**Tenant status condition:**

```go
type TenantStatus struct {
    // ... existing fields ...
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// New condition type
const (
    TenantConditionDefaultNetworkingReady = "DefaultNetworkingReady"
)
```

Condition values:
- `DefaultNetworkingReady: true` when the tenant default VN, its default
  NetworkACL, IPv4 Subnet and resolved NetworkACL association, and NATGateway
  are all READY
- `DefaultNetworkingReady: false, reason: <FailureReason>` when any default resource or its association failed to provision

**NetworkClass defaults field:**

```go
type NetworkClassSpec struct {
    // ... existing fields ...
    Defaults               *NetworkDefaults `json:"defaults,omitempty"`
    MetalLBVIPPrefixLength int32            `json:"metallbVIPPrefixLength,omitempty"` // reserves sub-range of each subnet for MetalLB VIPs (CaaS)
}

type NetworkDefaults struct {
    VirtualNetworkCIDR string           `json:"virtualNetworkCIDR,omitempty"`
    IPv4SubnetCIDR     string           `json:"ipv4SubnetCIDR,omitempty"`
    // Each rule set includes an ALLOW ALL catch-all for 0.0.0.0/0;
    // more-specific matching rules take precedence.
    IngressRules []NetworkACLRule `json:"ingressRules,omitempty"`
    EgressRules  []NetworkACLRule `json:"egressRules,omitempty"`
}

type NetworkACLRule struct {
    Action      string `json:"action"` // ALLOW or DENY
    Protocol    string `json:"protocol"` // ALL, TCP, UDP, or ICMP; case-sensitive, lowercase values are rejected
    PortFrom    *int32 `json:"portFrom,omitempty"` // optional TCP/UDP destination range
    PortTo      *int32 `json:"portTo,omitempty"`
    IPv4CIDR    string `json:"ipv4CIDR"` // ingress source or egress destination
}
```

**Resource spec fields (ComputeInstance, Cluster, BaremetalInstance):**

```go
type ComputeInstanceSpec struct {
    // ... existing fields ...
    AutoExternalIPAttachment bool `json:"autoExternalIPAttachment,omitempty"`
}

type BareMetalInstanceSpec struct {
    // ... existing fields ...
    AutoExternalIPAttachment bool `json:"autoExternalIPAttachment,omitempty"`
}

type ClusterSpec struct {
    // ... existing fields ...
    AutoExternalIPAttachment bool `json:"autoExternalIPAttachment,omitempty"`
}
```

#### Server Validation (fulfillment-service)

**NetworkClass defaults validation:**
- `virtual_network_cidr` must be valid CIDR notation
- `ipv4_subnet_cidr` must be valid IPv4 CIDR notation and within virtual_network_cidr range
- `virtual_network_cidr` and `ipv4_subnet_cidr` must use canonical IPv4 CIDR notation with host bits zero
- Ingress and egress rule lists are validated independently; priorities are unique within each direction and range from 1 through 32766
- Each rule action is ALLOW or DENY, and each protocol is ALL, TCP, UDP, or ICMP
- Port endpoints are both present or both absent, form an ordered range from 1 through 65535, and are only allowed with TCP or UDP
- Each rule contains a canonical IPv4 CIDR; ingress matches the source and egress matches the destination

**Resource creation with optional network attachment fields:**
- For VMaaS and BMaaS, an omitted or explicitly empty `network_attachments` list is resolved using the shared defaulting matrix. For CaaS, an omitted or explicitly empty `network_attachment` is resolved the same way.
- If one attachment is supplied, a missing Subnet receives the tenant default Subnet; the selected Subnet supplies its associated NetworkACL policy; and BMaaS defaults a missing interface to the first `fabric` port from the selected BareMetalInstanceType. Supplied values are preserved.
- A complete explicit attachment is preserved without applying defaults.
- If a required default is not configured or is not Ready, resource creation fails.

**Auto ExternalIP allocation (when auto_external_ip_attachment: true):**
- Pool selection: pick a READY IPv4 ExternalIPPool with the most available capacity
- If multiple pools have equal capacity: selection is deterministic but implementation-defined (e.g., alphabetical by pool name)
- If no pool has capacity: return error `ExternalIPPool exhaustion: no available capacity in any READY pool for IPv4`
- Pool capacity is checked and decremented synchronously during the API call. If the pool is exhausted, the call fails and no resources are persisted (including the parent resource). "Synchronous" here means the API call validates and creates DB records atomically — actual IP address allocation from the fabric manager and DNAT rule creation happen asynchronously through the operator reconciliation loop. See [Unified Networking — Auto-provisioning lifecycle](/enhancements/OSAC-1433-unified-networking/design.md#external-access-same-for-all-resource-types) for the full two-phase flow.

### Implementation Details/Notes/Constraints

#### Component Responsibility

| Component | Responsibility |
|-----------|---------------|
| fulfillment-service | Validate NetworkClass defaults, create the tenant default VN/IPv4 Subnet/NATGateway at onboarding, ensure each VN has its system-created default ACL, resolve an omitted Subnet ACL to the same-VN default and resolve workload Subnet/interface defaults, auto-provision ExternalIP, track DefaultNetworkingReady, and return errors on capacity exhaustion |
| osac-operator resource controllers | Clean up auto-created ExternalIP and ExternalIPAttachment via finalizer |
| osac-operator networking controllers | Reconcile default networking resources (same as manually created resources) |
| osac-installer | Configure NetworkClass defaults in setup.sh and installation overlays |

#### Default Resource Lifecycle

- **Creation:** Each VirtualNetwork receives one system-created default
  NetworkACL from the NetworkClass rules during VirtualNetwork provisioning.
  The VirtualNetwork is READY only when its network and default ACL are READY.
  At tenant onboarding, fulfillment-service creates the default VirtualNetwork,
  waits for its default ACL to become READY, then creates the default IPv4
  Subnet without `spec.network_acl`; the service resolves and stores the
  same-VN default ACL reference. It also creates the default NATGateway.
  This is the same ACL creation and Subnet defaulting behavior used for every
  tenant-created VirtualNetwork and Subnet.
- **Labeling:** All default resources labeled `osac.openshift.io/default: "true"`
- **Visibility:** Default ACLs appear in list/detail views like any other ACL.
  Each VirtualNetwork has exactly one default ACL; tenants with multiple
  VirtualNetworks therefore see multiple default ACLs, one scoped to each VN.
- **Mutability:** NetworkACL rules and the Subnet's ACL association are fixed at
  creation. To use a different policy, pause network changes and workload
  creation on each affected Subnet; delete dependent workloads and the
  Subnets, then recreate the Subnets with a custom ACL. Default ACLs cannot be
  deleted directly. Changing the default rules for future Subnets requires
  replacing the VirtualNetwork, which also replaces its default ACL.
  VirtualNetwork/Subnet address configuration is also immutable.
- **Deletion protection:** Custom ACLs cannot be deleted while Subnets
  reference them. A default ACL is deleted only with its VirtualNetwork, after
  its Subnets, custom ACLs, and NATGateway have been deleted. Other default
  resources cannot be deleted while they have dependents.
- **Tenant deletion:** Default resources are deleted when tenant is deleted (owner reference cleanup)

#### Auto-Provisioned Resource Lifecycle

- **Creation:** fulfillment-service creates ExternalIP at workload creation
  time when `auto_external_ip_attachment=true` (the ExternalIPPool is already
  Ready, so the readiness gate is satisfied). ExternalIPAttachment is created
  later by the fulfillment-service internal reconciler, only after the
  ExternalIP is Allocated and the target workload is Ready. This follows the
  same creation readiness rules as tenant-initiated operations — see
  [Unified Networking — Auto-provisioning lifecycle](/enhancements/OSAC-1433-unified-networking/design.md#auto-provisioning-lifecycle-auto_external_ip_attachment).
- **Labeling:** All auto-created resources labeled `osac.openshift.io/auto-created: "true"`
- **Cleanup:** Parent resource finalizer deletes auto-created ExternalIP/ExternalIPAttachment on parent deletion
- **Cleanup order:** ExternalIPAttachment → ExternalIP → parent resource removal
- **Cleanup failure:** If cleanup fails permanently (after retries), finalizer is removed, parent deleted, orphaned ExternalIP/ExternalIPAttachment left in cluster (manual cleanup required)
- **Manual resources block deletion:** If tenant created ExternalIP/ExternalIPAttachment explicitly (not labeled auto-created), they block the parent workload's deletion — the tenant must remove them first. See [Unified Networking — Deletion Dependency Guards](/enhancements/OSAC-1433-unified-networking/design.md#deletion-dependency-guards)

#### Prerequisite Ordering for Clusters

For clusters, two separate IP allocations happen from different sources:

- **External IPs** (from ExternalIPPool): allocated by ExternalIP controller via fabric manager (for DNAT front-end)
- **Internal VIPs** (from subnet CIDR): allocated by MetalLB from its IPAddressPool (for API/ingress endpoints)

The DNAT model maps external IPs to internal VIPs:

1. fulfillment-service creates ExternalIP resources at cluster creation time (pool is Ready, readiness gate satisfied). ExternalIPs start Pending.
2. osac-operator ExternalIP controller dispatches to fabric manager → ExternalIPs transition to Allocated (external addresses assigned, e.g., 203.0.113.10)
3. Cluster provisioning proceeds — MetalLB allocates internal VIPs from its IPAddressPool on the hosting cluster (e.g., 10.0.1.200 for API, 10.0.1.201 for ingress)
4. Template discovers VIPs after MetalLB allocation, writes to ClusterOrder status (`apiEndpoint`, `ingressEndpoint`)
5. VIP feedback loop: ClusterOrder status → feedback controller → fulfillment-service syncs to Cluster status
6. Once ExternalIP is Allocated AND Cluster is Ready: fulfillment-service internal reconciler creates ExternalIPAttachments (readiness gate satisfied)
7. ExternalIPAttachment controller creates DNAT: external IP → internal VIP → ExternalIPAttachment transitions to Ready

Note: the external IPs (from ExternalIPPool) and internal VIPs (from MetalLB IPAddressPool) are separate address spaces managed by separate systems. No IPAM coordination needed between them.

#### Tenant Isolation

All default and auto-created resources inherit tenant annotation from parent:
- `osac.openshift.io/tenant` annotation propagated from Tenant to default VN/Subnet/NetworkACL/NATGateway
- `osac.openshift.io/tenant` annotation propagated from ComputeInstance/Cluster/BaremetalInstance to auto-created ExternalIP/ExternalIPAttachment
- OPA policies enforce tenant-scoped read/create/delete operations for
  networking resources; ACL rules and Subnet association are immutable after
  creation, and controller status transitions remain internal

#### CIDR Overlap Across Tenants

All tenants receive the same default CIDR range as configured on the NetworkClass. Tenants are isolated at the fabric level — the unified networking API provides VirtualNetworks with any IP subnet, and the fabric manager enforces isolation regardless of overlapping CIDRs between tenants. This is a fabric-level concern, not an API-level concern.

### Security Considerations

This feature inherits the existing security model:
- Tenant isolation via `osac.openshift.io/tenant` annotation enforced by OPA policies
- Auto-provisioned resources (ExternalIP, ExternalIPAttachment) inherit tenant annotation from parent resource
- Default resources (VN, Subnet, NetworkACL, NATGateway) inherit tenant annotation from Tenant resource
- No new authentication or authorization changes
- NetworkClass default rules are configured by the Cloud Infrastructure Admin
  and materialized whenever a VirtualNetwork receives its default ACL. Later
  NetworkClass changes do not update existing default ACLs.
- NetworkACL rules and Subnet associations are fixed at creation; policy
  changes require the coordinated replacement process in
  [Default Resource Lifecycle](#default-resource-lifecycle). See the
  [default-policy risk](#risk-default-networkacl-too-permissive) for mitigation.

### Failure Handling and Recovery

#### Tenant Onboarding Failures

- **Default VirtualNetwork provisioning fails:** Tenant enters non-READY state with condition `DefaultNetworkingReady: false, reason: VirtualNetworkProvisioningFailed, message: "..."`
- **Default IPv4 Subnet provisioning fails:** Tenant enters non-READY state with condition `DefaultNetworkingReady: false, reason: SubnetProvisioningFailed, message: "..."`
- **Default NetworkACL provisioning fails:** Tenant enters non-READY state with condition `DefaultNetworkingReady: false, reason: NetworkACLProvisioningFailed, message: "..."`
- **Default Subnet-to-NetworkACL association fails:** Tenant remains non-READY with condition `DefaultNetworkingReady: false, reason: NetworkACLAssociationFailed, message: "..."`; the tenant's default Subnet is not used for workload creation until the association is READY
- **A VirtualNetwork's default ACL is not READY:** A Subnet create request
  that omits `spec.network_acl` fails with `FAILED_PRECONDITION` until that
  VirtualNetwork has exactly one READY default ACL. An explicit READY custom
  ACL in the same VirtualNetwork may be used instead.
- **Default NATGateway provisioning fails:** Tenant enters non-READY state with condition `DefaultNetworkingReady: false, reason: NATGatewayProvisioningFailed, message: "..."`
- **Recovery:** Cloud Provider Admin inspects failure (check networking controller logs, AAP job logs), fixes root cause, deletes tenant, re-creates tenant

#### Resource Creation Failures

- **No default networking resources:** Since defaults are mandatory on NetworkClass (rejected at creation time without them), this scenario should not occur. If it does due to data inconsistency, resource creation without an explicit network attachment returns error: `No default networking resources available. Please contact your administrator.`
- **ExternalIPPool capacity exhaustion:** create API call returns error: `ExternalIPPool exhaustion: no available capacity in any READY pool for IPv4`. Resource is NOT persisted.

#### Auto-Provisioned Resource Cleanup Failures

- **Transient failure:** Parent finalizer retries cleanup (exponential backoff)
- **Permanent failure:** After N retries, finalizer is removed, parent resource deleted, orphaned ExternalIP/ExternalIPAttachment left in cluster
- **Manual cleanup required:** Tenant Admin or Cloud Provider Admin manually deletes orphaned resources (identified by label `osac.openshift.io/auto-created: "true"` with no parent reference)

### RBAC / Tenancy

No new tenancy boundary is introduced. Default resources and auto-created ExternalIPs inherit the existing tenant isolation:
- `osac.openshift.io/tenant` annotation propagated from parent to all child resources
- OPA policies enforce tenant-scoped read (List/Get), Create, and Delete for
  networking resources; they expose no Update operation.
- Tenant User can view default and auto-created resources and use the supported
  read, create, and delete operations, subject to dependency protection.
- Cloud Infrastructure Admin configures NetworkClass defaults used during
  tenant onboarding; changes do not revise existing tenant ACLs.

### Observability and Monitoring

New structured log events:
- fulfillment-service: `CreatingDefaultNetworking` (info), `DefaultNetworkingReady` (info), `DefaultNetworkingFailed` (error), `PopulatedNetworkAttachmentsDefaults` (info), `AutoProvisionedExternalIP` (info), `ExternalIPPoolExhausted` (error)

New Kubernetes events on Tenant:
- `DefaultNetworkingCreated`: default VN, its automatic default NetworkACL,
  IPv4 Subnet, and NATGateway creation started
- `DefaultNetworkingReady`: default VN, its default NetworkACL, Subnet and
  resolved ACL association, and NATGateway are READY
- `DefaultNetworkingFailed`: default resource provisioning failed (includes reason and failed resource name)

New Kubernetes events on ComputeInstance/Cluster/BaremetalInstance:
- `NetworkAttachmentsPopulated`: resource-specific network attachment field populated with tenant defaults
- `AutoExternalIPCreated`: ExternalIP and ExternalIPAttachment auto-created

No new metrics or alerts (existing provisioning duration and failure rate metrics apply).

### Risks and Mitigations

#### Risk: ExternalIPPool exhaustion

**Impact:** Auto ExternalIP allocation fails, create API call returns error, tenant cannot create resource with auto_external_ip_attachment=true.

**Mitigation:** Pool capacity visible in status; clear error directs tenant to explicit allocation from another pool or contact admin.

**Reviewed by:** Cloud Provider Admin

#### Risk: Default NetworkACL too permissive

**Impact:** Each VirtualNetwork's default ACL has an `ALLOW ALL` catch-all for
`0.0.0.0/0` in ingress and egress. More-specific matching DENY rules take
precedence over this broad catch-all, so configured rules can restrict traffic.
Workloads may receive unsolicited inbound traffic or initiate outbound connections.
Changing NetworkClass defaults does not update existing default ACLs.

**Mitigation:** Cloud Infrastructure Admin can add more-specific DENY rules in
NetworkClass before tenant onboarding. Tightening an existing tenant's
policy requires the coordinated replacement process in
[Default Resource Lifecycle](#default-resource-lifecycle), including every
Subnet referencing the ACL and its dependent workloads.

**Reviewed by:** Cloud Infrastructure Admin

#### Risk: Auto ExternalIP orphans on partial failure

**Impact:** If parent resource finalizer cleanup fails permanently, orphaned ExternalIP/ExternalIPAttachment resources remain in cluster.

**Mitigation:** Parent resource finalizer handles cleanup; controller retries on transient failures. If cleanup permanently fails, finalizer is removed and parent deleted — orphaned ExternalIPs must be cleaned up manually by Tenant Admin or Cloud Provider Admin.

**Reviewed by:** Platform

#### ~~Risk: Deployment misconfiguration (NetworkClass defaults not configured)~~ — Eliminated

Since defaults are mandatory (a NetworkClass without defaults is rejected at creation time), this scenario cannot occur. The osac-installer setup.sh includes NetworkClass default configuration in installation overlays, and the API validation ensures defaults are always present.

### Drawbacks

#### CIDR overlap across tenants

All tenants receive the same default CIDR range. While fabric-level isolation prevents actual IP conflicts, this may confuse tenants who expect unique CIDR ranges.

**Trade-off:** Simplicity (single default configuration) vs. per-tenant customization. Chosen approach: single default, document fabric-level isolation. Alternative: per-tenant CIDR allocation (more complex, requires IPAM).

#### Capacity exhaustion returns API error, not Failed resource

When ExternalIPPool has no capacity, the create API call returns an error and the resource is NOT persisted. This provides no audit trail.

**Trade-off:** Simplicity vs. auditability. Chosen approach: return error (resource not persisted). Alternative: create Failed resource for audit trail (adds cleanup burden).

## Alternatives (Not Implemented)

### Alternative 1: Per-tenant default CIDR allocation

Instead of all tenants receiving the same default CIDR, allocate unique CIDR ranges per tenant from a global pool.

**Rejected because:** Adds complexity (requires IPAM, CIDR allocation tracking, exhaustion handling). The unified networking API allows overlapping CIDRs between tenants (fabric-level isolation), so unique CIDRs are not required. Single default CIDR is simpler.

### Alternative 2: Capacity exhaustion creates Failed resource instead of returning error

Instead of returning an error when ExternalIPPool has no capacity, create a Failed resource with a status condition.

**Rejected because:** Pool capacity is validated synchronously during the API call — if the pool is exhausted, the call fails atomically and no resources are persisted. Creating a Failed resource adds cleanup burden and audit trail complexity. Clear API error with no persisted state is simpler.


## Open Questions

### ~~1. Should capacity exhaustion return an API error or create a Failed resource?~~ — Resolved

Resolved: Return error, no resource persisted.

## Test Plan

### Unit Tests

- fulfillment-service: NetworkClass defaults validation (valid CIDR and rule
  fields, and include ALLOW ALL catch-all rules for `0.0.0.0/0` for both
  ingress and egress, with more-specific matches taking precedence)
- fulfillment-service: resource-specific attachment resolution (resolve omitted or empty fields, fill partial attachments, preserve complete explicit attachments)
- fulfillment-service: auto ExternalIP pool selection (pick READY pool with most capacity, respect IP family)
- fulfillment-service: capacity exhaustion error (return error, resource not persisted)
- fulfillment-service: default resource creation at tenant onboarding (VN,
  its automatic default NetworkACL, IPv4 Subnet with the resolved ACL
  association, NATGateway, and default labels)
- fulfillment-service: DefaultNetworkingReady condition tracking (true when VN, NetworkACL, Subnet and association, and NATGateway are READY via feedback; false when any fails)
- fulfillment-service: NetworkACL defaults validation (rule actions, duplicate match fields, protocol, optional ports, canonical IPv4 CIDRs)
- fulfillment-service: default NetworkACL includes the configured allow-all catch-all in each direction; more-specific DENY rules take precedence independent of input order
- osac-operator resource controllers: auto-created resource cleanup (delete ExternalIPAttachment → ExternalIP on parent deletion)

### Integration Tests

- E2E: create Tenant, verify default VN, its automatic default NetworkACL,
  IPv4 Subnet association, and NATGateway are READY and labeled
  `osac.openshift.io/default: "true"`
- E2E: create Tenant, default Subnet provisioning fails, verify Tenant remains non-READY with condition
- E2E: create ComputeInstance without network_attachments, verify defaults populated in spec
- E2E: create ComputeInstance with `--external-ip-attachment`, verify auto ExternalIP + ExternalIPAttachment created, DNAT rule functional
- E2E: create Cluster with `--external-ip-attachment`, verify two ExternalIPs created BEFORE provisioning, cluster VIPs match
- E2E: delete ComputeInstance with auto-created resources, verify ExternalIPAttachment and ExternalIP cleaned up
- E2E: create ComputeInstance with a complete explicit attachment and verify its values are preserved
- E2E: create ComputeInstance with a partial attachment and verify only missing fields are defaulted
- E2E: verify workload attachment stores only the resolved Subnet, and the effective ACL is read from that Subnet
- E2E: verify NetworkACL rules and Subnet associations have no Update operation and remain fixed after creation
- E2E: create ComputeInstance with `--external-ip-attachment` when pool exhausted, verify error returned, resource not persisted
- E2E: verify changing NetworkACL rules or Subnet association requires deleting and recreating the affected networking resources
- E2E: create multiple VirtualNetworks and verify each receives exactly one
  READY default NetworkACL, with no extra Subnet created automatically
- E2E: create a Subnet without an ACL and verify it stores the READY default
  ACL from its own VirtualNetwork
- E2E: create a Subnet with one READY custom ACL from the same VirtualNetwork
  and verify the explicit ACL is preserved
- E2E: verify multiple ACL references, cross-VirtualNetwork references, and
  omitted ACLs without a READY default ACL are rejected
- E2E: verify a custom ACL cannot be deleted while referenced, a default ACL
  cannot be deleted directly, and a Subnet cannot become READY without one
  active ACL

### Tricky Test Cases

- Tenant onboarding failure: default Subnet provisioning fails, verify Tenant non-READY, manual retry works
- ExternalIPPool exhaustion: verify error returned, no resource created
- Auto-provisioned resource cleanup failure: verify finalizer retry, eventual orphan cleanup
- Cluster ExternalIP prerequisite ordering: verify ExternalIPs allocated BEFORE provisioning, template receives correct VIPs

## Graduation Criteria

**Note:** This section will be updated when the enhancement is targeted at a release.

Proposed maturity level: **Tech Preview** → **GA**

Tech Preview criteria:
- [ ] NetworkClass defaults field implemented in fulfillment-service and osac-operator
- [ ] fulfillment-service ensures each VirtualNetwork has one system-created
  default NetworkACL and resolves an omitted Subnet ACL to the same-VN default
- [ ] fulfillment-service creates the tenant default VN/IPv4 Subnet/NATGateway
  at onboarding, with the default Subnet storing its resolved ACL reference
- [ ] Tenant DefaultNetworkingReady condition functional
- [ ] Resource-specific network attachment field optional on all three resource types (ComputeInstance, Cluster, BaremetalInstance)
- [ ] Auto ExternalIP attachment (auto_external_ip_attachment) functional for VM and BM
- [ ] Auto ExternalIP attachment for Cluster functional
- [ ] Auto-provisioned resource cleanup via parent finalizer functional
- [ ] Integration tests pass (E2E coverage for default networking, optional attachments, auto ExternalIP, cleanup)
- [ ] Documentation: API reference, user guide for simplified resource creation

GA criteria:
- [ ] Production deployment verified (MOC or other OSAC deployment)
- [ ] User feedback incorporated (usability, error messages, edge cases)
- [ ] osac-installer includes NetworkClass default configuration in setup.sh and overlays
- [ ] No major bugs reported in Tech Preview period
- [ ] Performance validated (tenant onboarding duration, resource creation latency)

## Upgrade / Downgrade Strategy

### Upgrade

The NetworkACL and attachment contract change is a coordinated, breaking
release; the prior release cannot represent NetworkACL resources or Subnet
associations. Tenants must not be asked to create ACLs or associations before
the ACL-aware API is deployed. The cutover follows the [unified networking
upgrade strategy](/enhancements/OSAC-1433-unified-networking/design.md#upgrade--downgrade-strategy)
and adds the default-networking steps below. Existing tenant policy is
tenant-mapped; no automatic or lossless conversion is promised.

#### Pre-upgrade inventory and preparation

- Inventory each affected tenant's VirtualNetworks, Subnets, default-resource
  labels, workload attachments, and existing workload traffic policies.
- Prepare a tenant-approved mapping from existing policies to one intended
  NetworkACL per Subnet, including explicit return-traffic rules where needed.
  If workloads on one Subnet require different policies, plan separate Subnets
  and workload recreation because attachments are immutable. Prepare rule
  specifications and association plans, but do not create ACLs under the old
  release.
- Snapshot the current API/database, CR, NetworkClass, and workload attachment
  state and schedule a maintenance window. Keep the snapshot and reverse plan
  for rollback.

#### Coordinated cutover

1. Freeze tenant network and workload writes that can affect the migration,
   including onboarding, resource creation, deletion, and attachment changes.
2. Deploy the ACL-aware fulfillment-service, operator, networking controllers,
   schemas, and compatible clients as one coordinated release. Do not permit
   old clients to write with the previous attachment contract.
3. Ensure the system has created one default NetworkACL for each existing
   VirtualNetwork from the NetworkClass rules; wait until each is READY. This
   backfill does not change any existing Subnet association. Through the new
   API, create each tenant-approved custom NetworkACL in the same
   VirtualNetwork as its Subnets and wait for it to become READY. Create
   replacement Subnets with either an explicit custom ACL or no ACL to use the
   same-VN default. Existing Subnets cannot be reassociated in place; recreate
   dependent workloads as required by their deletion constraints. A Subnet is
   READY only when its resolved associated policy is active.
4. Keep each affected Subnet and workload creation/default resolution gated
   until that Subnet is READY. Recreate workloads only after their destination
   Subnet's policy is active when tenant policy mapping requires a move.
5. Validate the mapping, readiness, and representative connectivity. Reopen
   network and workload writes only after every affected Subnet has an active
   same-VirtualNetwork ACL association; incomplete tenant migrations remain
   gated.

Existing tenants do not receive a tenant-default VN, Subnet, or NATGateway
retroactively. Existing VirtualNetworks do receive their system-created
default NetworkACL during cutover; existing Subnet associations are not
changed. An existing tenant that wants simplified workload creation must use
the new API to create or select its tenant-default Subnet. Omitting
`spec.network_acl` on that Subnet resolves to the default ACL of the same VN.
New tenant onboarding creates the default VN, its default ACL, and the default
Subnet through the ACL-aware release.

### Downgrade

The prior release cannot represent or manage NetworkACL resources,
`Subnet.spec.network_acl`, or the new attachment contract. A binary-only
rollback after ACL creation or association is unsupported. To roll back, freeze
network/workload writes, restore the pre-upgrade database, CRs, NetworkClass,
and workload attachment state from the coordinated snapshot, and roll back the
fulfillment-service, operator, networking controllers, schemas, and clients
together. ACL resources and migrated associations must be removed or reversed
as part of that tested restore; tenant policy mapping has no automatic reverse
conversion. If the pre-upgrade state cannot be restored, keep the ACL-aware
release and fix forward.

The existing auto-ExternalIP downgrade limit still applies: release `N` does
not recognize `auto_external_ip_attachment`, and auto-created resources may
need manual cleanup if they are no longer required. Do not treat retained
default resources as evidence that rollback is safe; the old release cannot
interpret ACL state.

## Version Skew Strategy

### Control Plane Skew

The ACL-aware fulfillment-service, schemas/CRDs, osac-operator, networking
controllers, and configured networking manager must be deployed and rolled
back together. Mixed `N`/`N+1` control-plane versions are unsupported during
cutover because `N` cannot represent ACL resources or Subnet associations.
Keep affected network and workload writes frozen until the new release is
complete and every migrated Subnet policy is active.

### Client Skew

Clients using the previous attachment contract cannot write safely against the
ACL-aware release, and new clients cannot create NetworkACLs or supply
`spec.network_acl` to release `N`. Upgrade clients with the control plane,
block old-client network/workload writes during the cutover, and do not reopen
writes until clients and APIs use the same contract. The existing limitation
for `--external-ip-attachment` also applies: new clients require the upgraded
server, while older clients cannot request that behavior. Keep clients and the
fulfillment-service within one coordinated minor release.

## Support Procedures

### Symptom: Tenant stuck in non-READY state, condition "DefaultNetworkingReady: false"

**Detection:**
```bash
kubectl describe tenant acme-corp -n <namespace>
# Check status.conditions for DefaultNetworkingReady
```

**Cause:** Default VirtualNetwork, IPv4 Subnet, NetworkACL, or NATGateway provisioning failed

**Resolution:**
1. Check default networking resource status: `kubectl get virtualnetwork -n <namespace> -l osac.openshift.io/default=true`
2. If VirtualNetwork/IPv4 Subnet/NetworkACL/NATGateway is not READY, investigate provisioning failure (check networking controller logs, AAP job logs)
3. Fix root cause (e.g., AAP connectivity issue, fabric manager error)
4. Delete tenant: `osac delete tenant acme-corp`
5. Re-create tenant: `osac create tenant --name acme-corp`

### Symptom: Resource creation fails with "No default networking resources available"

**Detection:** API call returns error: `No default networking resources available. Please contact your administrator.`

**Cause:** Tenant has no default networking resources (data inconsistency — defaults are mandatory on NetworkClass, so this should not occur under normal operation)

**Resolution:**
1. Check NetworkClass configuration: `osac get networkclass -o yaml`
2. Verify NetworkClass.spec.defaults is set (required — NetworkClass creation is rejected without defaults)
3. If tenant has no default resources despite NetworkClass having defaults, delete and re-create tenant

### Symptom: Auto-provisioned ExternalIP not cleaned up after resource deletion

**Detection:** `kubectl get externalip` shows orphaned ExternalIP labeled `osac.openshift.io/auto-created: "true"` with no parent

**Cause:** Finalizer cleanup failed permanently

**Resolution:**
1. Check resource deletion logs (controller logs) for cleanup errors
2. Manually delete orphaned ExternalIPAttachment: `kubectl delete externalipattachment <name> -n <namespace>`
3. Manually delete orphaned ExternalIP: `kubectl delete externalip <name> -n <namespace>`

### Disabling the feature

To disable auto ExternalIP attachment:
- Remove or redact ExternalIPPool CRs (capacity exhaustion prevents auto allocation)
- No API extension to disable NetworkClass defaults (fields are part of CRD, cannot be removed at runtime)

Consequences:
- Auto ExternalIP allocation fails with error (resource not created)
- Manual ExternalIP workflows remain functional
- Default networking at tenant onboarding is partially functional — VN, Subnets, and NetworkACL are created, but NATGateway creation also fails (requires ExternalIP from pool). Auto external access is disabled

## Infrastructure Needed

- osac-installer: NetworkClass default configuration in setup.sh and installation overlays
- fulfillment-service: NetworkClass defaults validation, default VN/IPv4 Subnet/NetworkACL/NATGateway creation at tenant onboarding, resource-specific attachment resolution, auto ExternalIP provisioning, DefaultNetworkingReady condition tracking
- Integration test environment: kind cluster with Tenant, NetworkClass, ExternalIPPool resources

---

## Provenance

Authored: revise @ design 0.11.3 - cc0daa6, workspace main @ 06d340f90 (43 behind origin/main)
Final: revise @ design 0.11.3 - 2bd6607, workspace main @ 06d340f90 (81 behind origin/main)

> Context changed between revise and revise.

> This document's phase history does not include an initial /draft — structure was not verified against the template from origin.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.3","ai_workflows":"2bd6607","source_repo":"06d340f90","source_repo_branch":"main","commits_behind_main":81,"commits_ahead_main":0,"main_ref":"main","phases":["revise","revise","respond","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":true} -->
