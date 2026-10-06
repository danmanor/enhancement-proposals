---
title: unified-networking-api
authors:
  - dmanor@redhat.com
creation-date: 2026-06-03
last-updated: 2026-10-06
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-1433
prd: "prd.md"
see-also:
  - BareMetal Instance API: /enhancements/OSAC-1118-baremetal-instance-api
  - Three-Layer Networking Model: https://docs.google.com/document/d/1MwBjpmYoZoUN3PVjeIRZ2Y6mBuf0lu1uvTtN6XXPPTM
  - VMaaS Networking: /enhancements/OSAC-1435-vmaas-networking
  - CaaS Networking: /enhancements/OSAC-1436-caas-networking
  - BMaaS Networking: /enhancements/OSAC-1437-bmaas-networking
  - Default Networking: /enhancements/OSAC-1433-default-networking
replaces:
  - OSAC-356 Networking API (legacy)
superseded-by:
  - N/A
---

# Unified Networking API for VMaaS, CaaS, and BMaaS

| Field | Value |
|-------|-------|
| Jira | https://redhat.atlassian.net/browse/OSAC-1433 |
| Target release | OSAC 0.2 |
| PRD | [Unified Networking PRD](prd.md) |

## Contents

- [1. Summary](#1-summary)
- [2. Motivation: Goals and Non-Goals](#2-motivation-goals-and-non-goals)
  - [Goals](#21-goals)
  - [Non-Goals](#22-non-goals)
- [3. Background and Rationale](#3-background-and-rationale)
- [4. Proposal](#4-proposal)
  - [4.1 Resource Model](#41-resource-model)
    - [Provider Profile and Manager Roles](#provider-profile-and-manager-roles)
    - [Resource API Meaning](#resource-api-meaning)
  - [4.2 Data Model and Schema Changes](#42-data-model-and-schema-changes)
    - [API Extensions](#api-extensions)
      - [NetworkClass](#networkclass)
        - [Capabilities](#capabilities)
        - [Manager Registration (ConfigMap)](#manager-registration-configmap)
      - [ExternalIPPool](#externalippool-1)
      - [VirtualNetwork](#virtualnetwork)
      - [Subnet](#subnet)
      - [SecurityGroup](#securitygroup)
      - [ExternalIP](#externalip)
      - [Network Attachment Types](#network-attachment-types)
      - [Resource Status: Discovered IPs](#resource-status-discovered-ips)
      - [ExternalIPAttachment: Inbound Traffic (DNAT)](#externalipattachment-inbound-traffic-dnat)
      - [NATGateway: Outbound Traffic (SNAT)](#natgateway-outbound-traffic-snat)
    - [ExternalIPPool](#externalippool)
    - [Resource Hierarchy](#resource-hierarchy)
  - [4.3 Architecture and Manager Integration](#43-architecture-and-manager-integration)
    - [Manager Roles and Selection](#manager-roles-and-selection)
      - [Why Two Managers?](#why-two-managers)
    - [How VMs Join the Fabric](#how-vms-join-the-fabric)
    - [Dispatcher](#dispatcher-operator-composition-logic)
    - [Backend Effects and Completion Contract](#backend-effects-and-completion-contract)
    - [Manager Operation Contract](#manager-operation-contract)
  - [4.4 API Changes](#44-api-changes)
    - [Resource lifecycle enforcement](#resource-lifecycle-enforcement)
    - [API operation constraint](#api-operation-constraint)
    - [Implementation Details](#implementation-details)
    - [NetworkClass Examples](#networkclass-examples)
    - [UX Alignment](#ux-alignment)
    - [Workflow Description: End-to-End Flows](#workflow-description-end-to-end-flows)
    - [Auto-provisioning lifecycle](#auto-provisioning-lifecycle)
  - [4.5 Scalability and Performance](#45-scalability-and-performance)
  - [4.6 Security Considerations](#46-security-considerations)
  - [4.7 Failure Handling and Recovery](#47-failure-handling-and-recovery)
    - [Risks and Mitigations](#risks-and-mitigations)
    - [Provider Networking Control](#provider-networking-control)
  - [4.8 RBAC and Tenancy](#48-rbac-and-tenancy)
  - [4.9 Extensibility and Future-Proofing](#49-extensibility-and-future-proofing)
- [5. Interface Changes](#5-interface-changes)
  - [IC-1: Shared networking resource API](#ic-1-shared-networking-resource-api)
  - [IC-2: Workload network attachments](#ic-2-workload-network-attachments)
  - [IC-3: External access](#ic-3-external-access)
  - [IC-4: Provider manager configuration](#ic-4-provider-manager-configuration)
  - [IC-5: Provider networking control](#ic-5-provider-networking-control)
  - [IC-6: Unified networking UI and documentation](#ic-6-unified-networking-ui-and-documentation)
- [6. Alternatives Considered](#6-alternatives-considered)
  - [Drawbacks](#drawbacks)
- [7. Observability and Monitoring](#7-observability-and-monitoring)
- [8. Impact and Compatibility](#8-impact-and-compatibility)
  - [Current implementation alignment](#current-implementation-alignment)
  - [Upgrade and Downgrade Strategy](#upgrade-and-downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
  - [Support Procedures](#support-procedures)
  - [Infrastructure Needed](#infrastructure-needed)
- [Test Plan](#test-plan)
  - [Unit and component tests (DEV)](#unit-and-component-tests-dev)
  - [Integration tests (DEV)](#integration-tests-dev)
  - [End-to-end release qualification (QE)](#end-to-end-release-qualification-qe)
- [Graduation Criteria](#graduation-criteria)
- [Support Boundaries](#support-boundaries)
  - [Deployment Support Boundary](#deployment-support-boundary)
  - [Networking Hub Support Boundary](#networking-hub-support-boundary)

## 1. Summary

This design defines one networking application programming interface (API) and resource model for virtual machines (VMs), managed Kubernetes clusters, and bare-metal servers, delivered through Virtual-Machine-as-a-Service (VMaaS), Cluster-as-a-Service (CaaS), and bare-metal-as-a-service (BMaaS). Provider-selected manager roles supply backend behavior without changing resource meanings across workload types; see the [PRD](prd.md) for user needs and acceptance criteria.

## 2. Motivation: Goals and Non-Goals

### 2.1 Goals

- Use the same tenant networking resource schemas across VM, cluster, and bare-metal workloads; keep workload-specific connection fields on each workload resource.
- Resolve provider implementations through the deployment's NetworkClass registrations so tenant requests contain no backend selector.
- Assign each manager role a fixed operation and workload-target set, and require each assigned implementation to fulfill that complete contract.
- Limit this contract to Internet Protocol version 4 (IPv4) and one tenant network attachment per workload.

### 2.2 Non-Goals

- Internet Protocol version 6 (IPv6), dual-stack networking (IPv4 and IPv6 together), or multiple tenant network attachments per workload.
- Tenant-managed Domain Name System (DNS) zones, virtual private cloud (VPC) peering, load balancers, or Internet gateways.
- Advanced bare-metal interface configuration such as bonding or virtual local area network (VLAN) trunking.

## 3. Background and Rationale

OSAC previously exposed different networking workflows for virtual machines, managed Kubernetes clusters, and bare-metal servers. This design gives those workloads one resource model; the Resource Model section below defines each resource and provider role before the design describes their schemas and interactions. Per-service provisioning details remain in the VM, cluster, and bare-metal networking proposals linked in the metadata.

## 4. Proposal

### 4.1 Resource Model

This section defines the provider configuration, manager roles, and networking resources used throughout the design.

#### Provider Profile and Manager Roles

A **NetworkClass** is a provider-managed configuration container for one deployment. It holds manager-role selections, provider defaults, capability controls (including support for data processing unit (DPU) networking), and readiness status; it is not a tenant network and tenants do not select it. The active NetworkClass and its role assignments form the deployment's **networking profile**.

Each deployment's **networking hub** is the provider-designated OSAC management cluster where networking resource objects are stored and reconciled. Resource status records that hub so reconciliation stays in the same cluster.

A **manager role** is a stable OSAC responsibility boundary. Providers may use implementations from any source as long as each implementation fulfills the complete contract for its role:

- **Fabric Manager:** connects OSAC to the provider's physical network and applies network isolation, traffic policy, address allocation, inbound and outbound address translation, and physical-port placement.
- **Kubernetes (K8s) Manager:** handles Kubernetes-side networking. When configured alongside the Fabric Manager, it creates a **VM overlay**, the software network that carries VM traffic inside a hosting cluster (the OpenShift cluster that runs the VMs), and connects that overlay to the provider network.

A **manager registration** is a Kubernetes ConfigMap that identifies one implementation and role. Its required fields are a role label, a unique logical name, an implementation reference, a contract version, and declared networking capabilities. The implementation reference names the provider's collection role; the contract version identifies the required OSAC interface. A registration does not define an implementation-specific subset of operations or workload targets.

The **fixed dispatch contract** assigns each operation and its allowed workload targets to a manager role. Every conforming implementation supplies the complete set assigned to its role. A Fabric Manager is required for every deployment and handles the shared network, policy, address, NAT, and physical-port operations. A deployment that supports VMs also assigns a K8s Manager to create and remove VM overlay resources for each Subnet; deployments without VMs may omit that role. The operator rejects requests when a required role is missing before starting provider work; a registered implementation missing a required task is nonconforming and its provider job fails. Registration fields are defined here; operation entry points, target sets, and results are detailed in [Manager Operation Contract](#manager-operation-contract).

The **fulfillment service** stores and serves networking API resources. The `osac-operator` observes those resources and reconciles provider changes. Provider work runs through Ansible Automation Platform (AAP), which invokes the selected manager implementation.

#### Resource API Meaning

VMaaS provides the `ComputeInstance` workload resource for a VM, CaaS provides the `Cluster` workload resource for a managed Kubernetes cluster, and BMaaS provides the `BaremetalInstance` workload resource for a bare-metal server. All three services manage their workload lifecycle. In the table, “Provider” and “Tenant” identify who manages the networking resource: provider-managed resources configure the deployment, while tenant-managed resources express tenant network intent. Field names, cardinality, defaults, and status are specified in [Data Model and Schema Changes](#42-data-model-and-schema-changes).

| Resource | Owner | Meaning |
|---|---|---|
| **NetworkClass** | Provider | The deployment's active networking profile: manager-role selections, defaults, capability controls, and status. |
| **VirtualNetwork** | Tenant | An isolated tenant IP network and the parent of its Subnets, SecurityGroups, and optional NATGateway. Its Subnets are separate Layer 2 (L2) broadcast domains and are connected by Layer 3 (L3) routing, subject to SecurityGroup policy. |
| **Subnet** | Tenant | An IP range and network segment within one VirtualNetwork; workloads attached to the same Subnet share an L2 broadcast domain and have L3 connectivity through their parent VirtualNetwork. |
| **Workload network attachment** | Workload API | Connects one ComputeInstance, Cluster, or BaremetalInstance interface to a Subnet and selects its SecurityGroups. A bare-metal attachment may also name one exposed network interface; it is not a separate provider network resource. Traffic policy is enforced on this attachment. |
| **SecurityGroup** | Tenant | Stateful incoming and outgoing traffic rules selected on a workload network attachment. Matching rules allow traffic, established connections allow their reply traffic, and unmatched new traffic is denied. Groups can be attached only to workloads in their parent VirtualNetwork. |
| **ExternalIPPool** | Provider | Deployment-wide capacity of IPv4 addresses outside tenant VirtualNetworks. |
| **ExternalIP** | Tenant | One external IPv4 address allocated, or pending allocation, from an ExternalIPPool. “External” means outside the tenant VirtualNetwork, not necessarily reachable from the public Internet. |
| **ExternalIPAttachment** | Tenant | Associates one allocated ExternalIP with a VM, bare-metal server, or cluster endpoint for inbound access using destination network address translation (DNAT). |
| **NATGateway** | Tenant | Associates one ExternalIP with one VirtualNetwork for outbound source network address translation (SNAT); it does not provide inbound access. |

The resource model is **infrastructure-agnostic**: every resource keeps the same meaning across virtual machines, managed clusters, and bare-metal workloads. Workload-specific connection details remain on the workload API. It is **backend-agnostic**: provider choice does not change tenant resources, and any implementation that fulfills the manager-role contract can supply the backend behavior.

**Tenant defaults** are the default VirtualNetwork, Subnet, and SecurityGroup created for each tenant; when provider networking is enabled, onboarding also creates a default NATGateway associated with an ExternalIP. The [Default Networking design](/enhancements/OSAC-1433-default-networking/design.md) defines their creation, readiness, and disabled-mode behavior. Workload attachment defaulting uses the default Subnet and SecurityGroup as described in [Attachment Presence and Defaulting](#attachment-presence-and-defaulting).

An address marked **Allocated** has been confirmed by the Fabric Manager. A resource marked **Ready** has met the completion gate defined for that resource; in disabled provisioning mode, logical readiness does not by itself prove provider connectivity. The status schemas and disabled-mode behavior below define these distinctions for each resource.

### 4.2 Data Model and Schema Changes

This section gives the fields and status shapes for the resources defined above. Network address ranges use Classless Inter-Domain Routing (CIDR) notation. The Protocol Buffers (protobuf) messages below are the normative target for field names, cardinality, defaults, and status; the PRD describes their user-visible effects.

#### API Extensions

##### NetworkClass

As defined in [Provider Profile and Manager Roles](#provider-profile-and-manager-roles),
NetworkClass is the provider-managed configuration container for the active
deployment profile. It is not a tenant network; tenant VirtualNetworks refer
to the service-resolved profile. This schema carries the provider's manager
role selections, defaults, capability controls, and status.

```protobuf
message NetworkClass {
  string id = 1;
  Metadata metadata = 2;
  string title = 3;
  string description = 4;
  NetworkClassConstraints constraints = 6;
  NetworkClassCapabilities capabilities = 7; // derived from manager registrations
  NetworkClassStatus status = 8;             // read-only readiness and hub
  string fabric_manager = 10;          // required role for shared network operations
  optional string k8s_manager = 11;    // Kubernetes-side networking role
  NetworkClassSpec spec = 12;
}

message NetworkClassSpec {
  NetworkDefaults defaults = 1;
  NetworkClassCapabilities disable_capabilities = 2;
  optional int32 vip_prefix_length = 3; // virtual IP (VIP) reservation prefix; valid range 1 through 32
  EastWestConfig east_west_config = 4;
}

message NetworkClassStatus {
  NetworkClassState state = 1; // system-reported readiness
  optional string message = 2;
  string hub = 3; // provider-configured networking hub for resource placement
}

enum NetworkClassState {
  NETWORK_CLASS_STATE_UNSPECIFIED = 0;
  NETWORK_CLASS_STATE_PENDING = 1;
  NETWORK_CLASS_STATE_READY = 2;
  NETWORK_CLASS_STATE_FAILED = 3;
}

message NetworkClassConstraints {} // reserved for future provider constraints

message NetworkClassCapabilities {
  bool supports_ipv4 = 1;
  bool supports_ipv6 = 2;        // false for this proposal
  bool supports_dual_stack = 3;  // false for this proposal
  bool dpu_support = 4; // DPU-accelerated networking available
  bool supports_east_west_ethernet = 5;
  bool supports_east_west_infiniband = 6;
  bool supports_nvlink = 7;
}
```

###### Capabilities

Capabilities are **inferred from the assigned managers** and published in
the NetworkClass `capabilities` field — the provider does not set them
manually. The operator computes the intersection of capabilities declared by
the assigned manager ConfigMaps and populates `capabilities` automatically. For
a NetworkClass without a `k8s_manager`, the absent manager is excluded
from this intersection; the required `fabricManager` contributes
capabilities in every NetworkClass.

The deployment supports IPv4 only. Manager registrations declare the IPv4
capability; IPv6 and dual-stack declarations are rejected. NetworkClass
capability output is `supportsIpv4: true`, `supportsIpv6: false`, and
`supportsDualStack: false`.

| Capability | Type | Meaning |
|-----------|------|---------|
| `supportsIpv4` | bool | IPv4 addressing is available; `true` for OSAC networking |
| `supportsIpv6` | bool | IPv6 addressing; always `false` |
| `supportsDualStack` | bool | IPv4 + IPv6 addressing; always `false` |
| `dpuSupport` | bool | DPU-accelerated networking available |

The set of capabilities is defined by the operator and is fixed — adding a
new capability requires an operator update. Managers declare which
capabilities they support; they cannot define custom capabilities. These
capability flags describe networking features; the fixed manager contract
defines operations and workload targets.

###### Manager Registration (ConfigMap)

Each implementation registers one manager role through a ConfigMap installed
in the OSAC operator namespace. The role label determines whether it is a
Fabric Manager or K8s Manager. The proposed registration adds a fully qualified
`implementationRef` and required `contractVersion` to the current fields: a
unique `name`, declared `capabilities`, and optional description. `implementationRef`
resolves the provider implementation. `contractVersion` lets OSAC reject an
implementation before dispatch if its manager input, entry points, or
operation semantics do not match the contract. The first supported value is
`v1`; any other or missing value is rejected until OSAC supports it. This
field is new: the current manager ConfigMap parser does not read it.
It is distinct from the result envelope's `schemaVersion`, which validates a
response after a task runs; the registration version check lets OSAC fail
before provider work starts. The registration does not
list operations or targets; the manager contract defines the complete required
set for each role. Credentials remain in provider-managed AAP credentials or
Secrets, not in the registration.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: osac-network-fabric-manager-example
  namespace: osac
  labels:
    osac.openshift.io/network-fabric-manager: "true"
data:
  name: example
  implementationRef: acme.networking.fabric_manager
  description: "Example Fabric Manager implementation"
  contractVersion: "v1"
  capabilities: "ipv4"
```

The operator validates each NetworkClass role assignment against the matching
registration before dispatch. The registration resolves the implementation;
the fixed manager contract supplies its required operations and targets. A
conforming implementation from any source can be registered without changing
tenant APIs or operation dispatch. Adding an OSAC operation, target type, or
incompatible input/output contract requires an OSAC contract update. Legacy
implementation-strategy fields or annotations on networking resources do not
select an implementation and must not alter fixed role dispatch.


##### ExternalIPPool

ExternalIPPool is provider-managed address capacity available to all workload
types. Its CIDR is provider input and is not a tenant-selected backend or
workload setting. The provider-facing create API accepts one canonical IPv4
CIDR and the IPv4 family; capacity and readiness are system-reported.

```protobuf
message ExternalIPPool {
  string id = 1;
  Metadata metadata = 2;
  ExternalIPPoolSpec spec = 3;
  ExternalIPPoolStatus status = 4;
}

message ExternalIPPoolSpec {
  repeated string cidrs = 2; // private/provider input: exactly one canonical IPv4 CIDR
  IPFamily ip_family = 3;    // required provider input: IP_FAMILY_IPV4
}

message ExternalIPPoolStatus {
  ExternalIPPoolState state = 1;
  optional string message = 2;
  string hub = 3; // provider-configured networking hub for resource placement
  int64 total = 4;
  int64 allocated = 5;
  int64 available = 6;
}
```

The public tenant projection exposes pool identity, metadata, family, and
available capacity. CIDR configuration and detailed provider status remain
provider-managed.

##### VirtualNetwork

```protobuf
message VirtualNetworkSpec {
  NetworkClassReference network_class = 1; // required; service-resolved provider profile, immutable
  string ipv4_cidr = 2;                    // required canonical IPv4 CIDR, immutable
}
```

The fulfillment service resolves the deployment's provider-managed
NetworkClass; tenants do not choose a manager or provider profile. The
NetworkClass selects registered manager implementations. Tenant-facing
networking specs do not select a workload type or concrete backend. The same
resource model applies across virtual-machine, cluster, and bare-metal workloads.
Each Subnet in the VirtualNetwork is a separate L2 broadcast domain. The
VirtualNetwork routes traffic between its Subnets at L3, subject to the
SecurityGroups selected on each workload attachment; it does not join those
Subnets into one broadcast domain.

##### Subnet

```protobuf
message SubnetSpec {
  VirtualNetworkLocalReference virtual_network = 1; // required, immutable
  string ipv4_cidr = 2; // required canonical IPv4, within parent CIDR, immutable
}
```

The Subnet's CIDR range must be contained by its VirtualNetwork range and must not
overlap another Subnet in that VirtualNetwork. Workloads attached to one Subnet
share that Subnet's L2 broadcast domain and have L3 connectivity through the
parent VirtualNetwork. A different Subnet remains a separate L2 broadcast
domain, with inter-Subnet traffic routed by the VirtualNetwork subject to
SecurityGroup policy. A Subnet is the workload attachment point; it does not
select a workload type or a manager.

##### SecurityGroup

```protobuf
message SecurityGroupSpec {
  VirtualNetworkLocalReference virtual_network = 1; // required, immutable
  repeated SecurityRule ingress = 2;
  repeated SecurityRule egress = 3;
}
```

Each `SecurityRule` matches a protocol, optional TCP/UDP port range, and an
IPv4 CIDR (source for ingress, destination for egress). A SecurityGroup is a
reusable rule set scoped to one VirtualNetwork. The workload API selects
SecurityGroups on each network attachment, and the backend enforces their
combined rules at that Subnet-to-interface attachment. A matching rule allows
traffic; unmatched new traffic is denied; return traffic for an established
connection is allowed automatically, so the rules are stateful. Rules contain
match criteria, not an explicit allow/deny action, so list order does not
change their meaning. When multiple SecurityGroups are attached to one
workload interface, their allow rules combine as a union. Every selected
SecurityGroup must belong to the VirtualNetwork that owns the attachment's
Subnet.

```protobuf
message SecurityRule {
  Protocol protocol = 1;         // TCP, UDP, ICMP, or ALL
  optional int32 port_from = 2;  // required for TCP/UDP, 1 through 65535
  optional int32 port_to = 3;    // required for TCP/UDP, >= port_from
  optional string ipv4_cidr = 4; // required canonical IPv4 CIDR
  optional string ipv6_cidr = 5; // legacy wire field; non-empty values rejected
}

enum Protocol {
  PROTOCOL_UNSPECIFIED = 0;
  PROTOCOL_TCP = 1;
  PROTOCOL_UDP = 2;
  PROTOCOL_ICMP = 3;
  PROTOCOL_ALL = 4;
}
```

`PROTOCOL_UNSPECIFIED` is invalid. Port bounds apply only to TCP and UDP;
ICMP and ALL ignore them. The IPv6 field remains only for wire compatibility
and must be empty.

##### ExternalIP

```protobuf
message ExternalIPSpec {
  ExternalIPPoolReference pool = 1; // required, immutable
}
```

The allocated address is system-provided in `ExternalIP.status.address`; a
tenant requests a pool, not a specific address. See
[ExternalIP Address Selection and Ownership](#externalip-address-selection-and-ownership)
for the manager allocation and release contract.

##### HostType and BareMetalInstanceType

**HostType** is a legacy system-level inventory resource. New BMaaS and CaaS
network attachment resolution uses the tenant-facing `BareMetalInstanceType`
and its `network_ports`; the workload networking contract does not use
`HostType` as a second source of truth.

```protobuf
message NetworkInterface {
  string name = 1;        // e.g., "data-0", "data-1", "mgmt-0" — unique within the type
  string role = 2;        // e.g., "fabric", "management", "storage", "lifecycle"
  string description = 3; // e.g., "100GbE fabric interface"
}
```

Existing HostType records may still expose `interfaces` for inventory and
legacy consumers, but that list does not expand the tenant attachment
cardinality or override `BareMetalInstanceType.network_ports`.

Interfaces are ordered. When multiple interfaces share the same role
(e.g., two `fabric` interfaces), the first one in the list is the default
for that role — used by CaaS for automatic interface resolution.

**BareMetalInstanceType** is a tenant-facing catalog resource defined in
the [BareMetalInstanceType EP](/enhancements/OSAC-1201-baremetal-instance-types).
It provides a richer hardware discovery catalog for BMaaS, including
structured network ports with additional type and speed information:

```protobuf
message BareMetalNetworkPortSpec {
  string name = 1;        // e.g., "data-0", "data-1", "mgmt-0" — unique within the type
  string role = 2;        // e.g., "fabric", "management", "storage", "lifecycle"
  string type = 3;        // e.g., Ethernet, InfiniBand
  string speed = 4;       // e.g., 1Gbps, 100Gbps
  string description = 5; // e.g., "100GbE fabric interface"
}
```

`BareMetalInstanceType` is the authoritative tenant-facing hardware and
network-port catalog. Its `BareMetalNetworkPortSpec` entries provide the
names, roles, types, and speeds used for BMaaS validation and CaaS interface
resolution; no HostType reverse lookup is required for the workload contract.

| Role | Meaning |
|------|---------|
| `fabric` | Primary fabric traffic (east-west, tenant workloads) |
| `management` | In-band management/control plane traffic |
| `storage` | Storage fabric traffic |
| `lifecycle` | Out-of-band lifecycle management (network boot, remote hardware management) — not tenant-attachable |

Roles are conventions, not enforced enums. Ports/interfaces with role
`lifecycle` are used by the provisioning system (Ironic, Metal3) and
should not appear in `network_attachments`.

**CaaS** uses BareMetalInstanceType: the fulfillment service resolves the
interface automatically (first `fabric`-role port → stored as immutable
`fabric_interface` on the node set definition).

**BMaaS** uses BareMetalInstanceType: the tenant discovers interfaces
from BareMetalInstanceType and specifies port names directly on
`BareMetalNetworkAttachment.interface`, validated against the
`BareMetalInstanceType.network_ports` list. The `interface` field references
a port name from that list.

##### Network Attachment Types

Each resource type has its own network attachment message. The core fields
(`subnet`, `security_groups`) are shared, but each type adds
resource-specific fields. `network_attachments` are immutable after
resource creation — changing network attachment requires recreating the
resource. VMaaS and BMaaS keep repeated fields for wire/API compatibility but
enforce a maximum of one entry. CaaS uses its existing singular field.

**ComputeNetworkAttachment** (for ComputeInstance):

```protobuf
message ComputeNetworkAttachment {
  SubnetLocalReference subnet = 1;                         // Optional on input; immutable after resolution
  repeated SecurityGroupLocalReference security_groups = 2; // Optional on input; immutable after resolution
}
```

The repeated field is retained for compatibility, but at most one entry is
accepted. The sole entry is the VM's default route/primary attachment; the
VMaaS attachment message has no primary field.

**BareMetalNetworkAttachment** (for BaremetalInstance):

```protobuf
message BareMetalNetworkAttachment {
  SubnetLocalReference subnet = 1;                         // Optional on input; immutable after resolution
  repeated SecurityGroupLocalReference security_groups = 2; // Optional on input; immutable after resolution
  string interface = 3;                 // optional, immutable: physical port name from BareMetalInstanceType
  optional bool primary = 4;            // omitted or true: implicit primary; false is rejected
}
```

The repeated field is retained for compatibility, but at most one entry is
accepted. The `interface` field, when supplied, references a port name from
the BareMetalInstanceType's network ports list; if omitted, the system picks
the default fabric interface. Omitted or empty attachment lists receive
defaults, and a supplied entry receives defaults only for missing fields. The
sole entry is the default route/primary attachment. A `primary: true` value is
accepted for compatibility and is redundant; `primary: false` is rejected.

**ClusterNetworkAttachment** (for Cluster):

```protobuf
message ClusterNetworkAttachment {
  SubnetLocalReference subnet = 1;                         // Required after resolution; immutable after creation
  repeated SecurityGroupLocalReference security_groups = 2; // Optional on input; immutable after resolution
}
```

A single attachment applies to the whole cluster — all node sets share the same subnet.
The `fabric_interface` is resolved by the fulfillment service at creation time for each
node set from its BareMetalInstanceType (first port with role `fabric`)
and stored on the node set definition. The tenant does not set this field.

##### Attachment Presence and Defaulting

The API distinguishes an omitted attachment from a supplied attachment, but
both an omitted attachment and an empty attachment list/message mean that the
caller requested the normal tenant defaults. Defaulting is field-level for a
single supplied attachment:

| Input | Resolution |
|---|---|
| VMaaS attachment omitted or empty | Add the tenant's default Subnet and default SecurityGroup. |
| BMaaS attachment list omitted or empty | Add the tenant's default Subnet, default SecurityGroup, and the first `fabric` port from `BareMetalInstanceType.network_ports`. |
| CaaS attachment omitted or empty | Add the tenant's default Subnet and default SecurityGroup; resolve the first `fabric` port from each node set's `BareMetalInstanceType` for the bare-metal worker handoff. |
| One attachment with no Subnet | Default the Subnet; preserve supplied SecurityGroups and, for BMaaS, the supplied interface. Every supplied SecurityGroup must belong to the default Subnet's VirtualNetwork or the create is rejected. |
| One attachment with no SecurityGroups | Default only the SecurityGroup list, but only when the resolved Subnet belongs to the tenant's default VirtualNetwork. Otherwise the caller must provide SecurityGroups from the resolved Subnet's VirtualNetwork. |
| One BMaaS attachment with no interface | Default only the interface to the first `fabric` port from `BareMetalInstanceType.network_ports`. |
| One complete attachment | Preserve all supplied values and validate readiness, tenant scope, and VirtualNetwork relationships. |

An explicitly empty `security_groups` list is treated as a missing
SecurityGroup value for this defaulting rule. If a required default is absent
or not Ready, creation fails with a validation or precondition error. The
fully resolved attachment is stored with the workload and is immutable after
creation. If the caller omits the Subnet but supplies a SecurityGroup from a
different VirtualNetwork than the tenant's default Subnet, OSAC rejects the
request; the caller must supply a Subnet from that SecurityGroup's
VirtualNetwork or choose SecurityGroups from the default Subnet's
VirtualNetwork. OSAC always validates the resolved Subnet and every selected
SecurityGroup against the same parent VirtualNetwork before accepting the
workload.

##### Resource Specs

**ComputeInstance**:

```protobuf
message ComputeInstanceSpec {
  // ... existing fields ...
  repeated ComputeNetworkAttachment network_attachments = 14; // max 1 for compatibility
  optional bool auto_external_ip_attachment = 18;
}
```

The repeated `network_attachments` field is retained for API compatibility, but the
fulfillment service and operator accept at most one entry. VMaaS has no separate
primary field; the sole entry is implicitly the default route.

**BaremetalInstance** (new — defined in the
[BareMetal Instance API enhancement](/enhancements/OSAC-1118-baremetal-instance-api)):

```protobuf
message BareMetalInstanceSpec {
  string catalog_item = 1;
  optional string ssh_public_key = 2;
  optional string user_data = 3;
  optional BareMetalInstanceRunStrategy run_strategy = 4;
  int64 restart_trigger = 5;
  map<string, google.protobuf.Any> template_parameters = 6;
  optional BareMetalInstanceImage image = 7;

  // NEW: OSAC networking; repeated for compatibility, max 1
  repeated BareMetalNetworkAttachment network_attachments = 8;
}
```

**Cluster** (new):

```protobuf
message ClusterSpec {
  string template = 1;
  map<string, google.protobuf.Any> template_parameters = 2;
  map<string, ClusterNodeSet> node_sets = 3;

  // NEW: networking
  ClusterNetworkAttachment network_attachment = 9;  // singular, one per cluster
}
```

- Cluster-internal pod and service network ranges use platform defaults.
- The cluster's template determines whether nodes are VMs or bare metal. Both
  types are placed on the same subnet — VMs via the K8s overlay (already
  bridged to the fabric), bare-metal nodes directly on the fabric.

The design uses two existing Cluster status fields populated by the system
during provisioning:

```protobuf
message ClusterStatus {
  string api_endpoint = 6;      // existing; set by CaaS template, internal API server VIP
  string ingress_endpoint = 7;  // existing; set by CaaS template, internal ingress VIP
}
```

These are used by the ExternalIPAttachment controller as the DNAT backend
IP when the target is a cluster (see
[Cluster ExternalIPAttachment flow](#cluster-externalipattachment-flow)).

##### Resource Status: Discovered IPs

After provisioning, resources receive IP addresses through Dynamic Host
Configuration Protocol (DHCP). Feedback controllers discover these addresses
and write them to status for tenant visibility and ExternalIPAttachment DNAT
target resolution.

**ComputeInstanceStatus:**

```protobuf
message ComputeNetworkAttachmentStatus {
  string subnet_ref = 1;               // Subnet ID (echoed from spec)
  string ip_address = 2;               // Discovered from KubeVirt virtual machine instance (VMI) network status
}

message ComputeInstanceStatus {
  // ... existing fields ...
  repeated ComputeNetworkAttachmentStatus compute_network_attachment_statuses = 7; // proposed
}
```

The feedback controller watches KubeVirt virtual machine instance
`status.interfaces[].ipAddress`, maps each interface to the corresponding
attachment using its Kubernetes network attachment reference, and sends the
Signal remote procedure call (RPC) to the fulfillment service.

**BaremetalInstanceStatus:**

```protobuf
message BareMetalNetworkAttachmentStatus {
  string interface = 1;                 // Physical interface name (echoed from spec)
  string subnet_ref = 2;               // Subnet ID (echoed from spec)
  string ip_address = 3;               // Discovered via query_dhcp_lease after provisioning (matches the interface address to its lease)
  bool primary = 4;                     // true for the sole resolved attachment; normalized from spec
}

message BareMetalInstanceStatus {
  // ... existing fields ...
  repeated BareMetalNetworkAttachmentStatus network_attachment_statuses = 4; // existing
}
```

The IP address is discovered after DHCP assigns it on the tenant network. After
`reconcileProvisioning` completes, the operator dispatches
`query_dhcp_lease` to the Fabric Manager's DHCP lease API, matching
the server's interface address to find the assigned IP (see
[BMaaS OQ#4 — Resolved](/enhancements/OSAC-1437-bmaas-networking/design.md#4-how-is-the-hosts-runtime-ip-discovered-after-network-reconfiguration-resolved)).
The operator writes the discovered IP to resource status, and the feedback
controller syncs to fulfillment service.

**ClusterStatus** does not have per-attachment IP status — CaaS uses
service-level virtual IPs (VIPs; `api_endpoint`, `ingress_endpoint`) rather than
per-node IPs. Per-agent IPs are tracked on the ClusterOrder resource's
`NodeSetStatus.AgentStatus.IPAddress` (operator-internal, not surfaced
to tenant).

##### ExternalIPAttachment: Inbound Traffic (DNAT)

Handles **inbound traffic only**. Does not affect egress (that is
NATGateway's job).

```protobuf
enum ExternalIPAttachmentEndpoint {
  EXTERNAL_IP_ATTACHMENT_ENDPOINT_UNSPECIFIED = 0;
  EXTERNAL_IP_ATTACHMENT_ENDPOINT_API         = 1;  // Cluster API server
  EXTERNAL_IP_ATTACHMENT_ENDPOINT_INGRESS     = 2;  // Cluster ingress wildcard
}

message ExternalIPAttachmentSpec {
  ExternalIPLocalReference external_ip = 1; // required, immutable

  oneof target {
    ComputeInstanceLocalReference compute_instance = 2;
    ClusterLocalReference cluster = 3;
    BareMetalInstanceLocalReference baremetal_instance = 4;
  }
  ExternalIPAttachmentEndpoint target_endpoint = 5;
  // Required when target=cluster; must be UNSPECIFIED otherwise.
}
```

All fields are immutable after creation.

##### NATGateway: Outbound Traffic (SNAT)

Handles **outbound traffic only**.

```protobuf
message NATGatewaySpec {
  VirtualNetworkLocalReference virtual_network = 1; // required, immutable
  ExternalIPLocalReference external_ip = 2;         // required, immutable
}
```

An ExternalIP can only be used by one consumer (either an
ExternalIPAttachment or a NATGateway, not both). One NATGateway per
VirtualNetwork. A NATGateway associates its ExternalIP with its
VirtualNetwork and uses that address as the network's outbound SNAT identity;
it does not provide inbound access. NATGateway is optional. Without one,
resources may still have default egress but without a controlled source IP.

All fields are immutable after creation.

**Direction summary:**

| Resource | Direction | Mechanism |
|----------|-----------|-----------|
| ExternalIPAttachment | Inbound (DNAT) | External IP → resource |
| NATGateway | Outbound (SNAT) | Resource → external IP |

#### ExternalIPPool

"External" in ExternalIPPool/ExternalIP means **outside the tenant's
VirtualNetwork**. The provider controls where the address is reachable; the
API does not promise Internet reachability. Deployment reachability
requirements are defined in [Support Boundaries](#support-boundaries).

ExternalIPPools are provider-managed and deployment-scoped. The required
Fabric Manager handles ExternalIP pool registration, allocation, and release.
One provider-managed pool serves all workload types in the deployment.
Each pool uses exactly one canonical IPv4 CIDR. The API's repeated `cidrs`
field is retained for compatibility, but validation rejects an empty list or
more than one entry; IPv6 and dual-stack pools are not supported.
Pool creation requires `spec.ipFamily` to be `IP_FAMILY_IPV4`;
`IP_FAMILY_UNSPECIFIED`, IPv6, and dual-stack values are rejected before
persistence. The provider create API supplies `cidrs` and `ipFamily`; the
provider configures that range in the Fabric Manager before
the pool becomes Ready. The pool does not carry a tenant-selectable
implementation field.

##### Address-Family and CIDR Contract

All user-supplied network CIDRs use canonical dotted-decimal IPv4 notation
(`a.b.c.d/prefix`) with host bits zero. A Subnet CIDR must be contained by its
parent VirtualNetwork and sibling Subnet CIDRs must not overlap. Provider and
controller-produced addresses are canonical IPv4 addresses without a CIDR
suffix. Any IPv6, dual-stack, malformed, or non-canonical value is rejected
before persistence or backend dispatch.
All explicit and automatic ExternalIP allocation paths, including per-service
auto-provisioning, must request `IP_FAMILY_IPV4`; `IP_FAMILY_UNSPECIFIED` is
not a valid default for this contract.

##### ExternalIP Address Selection and Ownership

The ExternalIPPool defines the eligible range; it does not choose the concrete
address. For `external_ip.allocate`, OSAC supplies the ExternalIP unique identifier (UID) and the
resolved pool UID and canonical IPv4 CIDR to the Fabric Manager. The manager chooses a free address in that pool and
durably reserves it under the ExternalIP UID. Allocations from the same pool
must be unique, and retrying the same UID must return the same reservation.
The selection order is implementation-specific; the contract does not require
first-fit or any other particular algorithm.

After confirming the reservation, the manager writes the address to the
`osac.openshift.io/allocated-address` annotation on the same ExternalIP resource.
The patch must be guarded by the supplied resource UID and generation, and
must not change the spec, status, or other annotations. The manager reports
success only after the provider reservation and annotation write succeed. The
common `osac_result` envelope identifies the operation, resource UID, and
generation; it carries no address payload.

After AAP reports success, OSAC validates the result envelope against the
current ExternalIP, reads the annotation, and validates canonical IPv4 form
and membership in the selected pool. Only then does OSAC write
`ExternalIP.status.address` and report the ExternalIP as **Allocated**. OSAC
does not write the allocated-address annotation. A missing or invalid
annotation leaves the ExternalIP non-ready with no accepted address. A retry
for the same UID reuses the provider reservation and retries the annotation
write. If the pool has no free address, the manager returns a failure with a
diagnostic and no success result.

The fulfillment service reserves API-side pool capacity in the transaction
that creates the ExternalIP. On deletion while provider networking is enabled,
OSAC first requires dependent ExternalIPAttachments and NATGateways to be
removed, then invokes `external_ip.release`. The manager removes the UID-owned
provider reservation and reports success only after the address is absent.
OSAC returns API-side pool capacity only after successful AAP completion and
validation of an `osac_result` with `schemaVersion: "v1"`,
`operation: external_ip.release`, the current ExternalIP UID in `resourceUID`,
the dispatched generation in `observedGeneration`, and empty `data`. The
successful result asserts that the UID-owned provider reservation is absent;
no separate `RELEASED` data field is required. A failed job or missing,
malformed, stale, or mismatched envelope keeps capacity held for
reconciliation.

While provider networking is disabled, deleting an ExternalIP completes its
OSAC object deletion without dispatching `external_ip.release`, and releases
its API-side pool-capacity reservation when the logical object is deleted. If
the manager had confirmed an allocation before disablement, its provider
reservation may remain after OSAC deletion and require manual or provider-side
cleanup. Releasing the OSAC capacity slot does not release that address in the
provider or guarantee that the provider can allocate it again. [User]

#### Resource Hierarchy

The diagram summarizes the resource relationships defined in the API extensions above. Manager roles shown on provider operations follow the dispatcher contract.

```text
NetworkClass (per deployment, provider-only; Fabric Manager required)

VirtualNetwork (tenant-managed, shared across workload types)
  ├── Subnet              → Fabric Manager; K8s Manager also creates VM overlays when VMs are supported
  ├── SecurityGroup       → Fabric Manager
  └── NATGateway          → Fabric Manager

ExternalIPPool (deployment-scoped, provider-managed)
  └── ExternalIP (tenant-managed) → Fabric Manager

ExternalIPAttachment (tenant-managed)
                          → Fabric Manager
                            references an ExternalIP and a target resource
```

### 4.3 Architecture and Manager Integration

The fulfillment service stores tenant API resources; the `osac-operator` observes and reconciles them. Ansible Automation Platform (AAP) runs provider jobs and invokes the selected manager implementation. OSAC uses the active networking profile and the fixed dispatch contract defined above to route each request.

```mermaid
flowchart LR
    Tenant --> API["Fulfillment API and database"]
    API --> Reconcile["Fulfillment reconciliation"]
    Reconcile --> Resources["Networking resources"]
    Resources --> Operator["osac-operator dispatcher"]
    Operator --> AAP["AAP manager role"]
    AAP --> Fabric["Fabric manager"]
    AAP -. "when VMs are supported: create the VM overlay and connect it to each Subnet" .-> K8s["K8s manager role"]
    Fabric --> Result["Manager result and resource annotations"]
    K8s --> Result
    Result --> Operator
    Operator --> Status["Observed resource status"]
    Status --> Reconcile
```


The BMaaS integration uses the per-server `BaremetalInstance` API, aligned with `ComputeInstance`; see the [BareMetal Instance API enhancement](/enhancements/OSAC-1118-baremetal-instance-api). The networking schemas support Internet Protocol version 4 (IPv4) only; IPv6 and dual-stack networking are outside scope. Per-service provisioning details remain in the linked VM, cluster, and bare-metal designs.

#### Manager Roles and Selection

The active NetworkClass always selects a Fabric Manager. If the deployment supports VMs, it also selects a K8s Manager to connect VM overlays to each Subnet. Deployments without VM workloads may omit the K8s Manager. All networking-resource operations use the fixed role assignments below; the K8s Manager does not replace or fall back for the Fabric Manager.

The complete operation and target set for each role is defined by the [dispatcher](#dispatcher-operator-composition-logic) and [Manager Operation Contract](#manager-operation-contract). A registration identifies an implementation but does not advertise an operation subset. Every implementation must provide every task assigned to its role. OSAC rejects a request when a required manager role is missing before starting AAP; a missing role-required task is implementation nonconformance and fails its provider job.

##### Why Two Managers?

The Fabric Manager handles changes to the provider's shared physical network, including the tenant networking resources and policy. The K8s Manager handles Kubernetes-side networking for VM overlays when VMs are supported. Keeping the roles separate lets a provider replace either implementation without changing tenant resources or operation routing.

#### How VMs Join the Fabric

OpenShift runs VMs through KubeVirt. Each VM is encapsulated in a pod whose
networking is managed by Open Virtual Network (OVN)-Kubernetes. VM addresses
start inside that overlay and are not visible on the physical network. When
VMs are supported, the K8s Manager connects the overlay to the provider
network while preserving the Subnet contract defined in [Resource API
Meaning](#resource-api-meaning): VMs attached to one Subnet share its L2
broadcast domain with other workloads on that Subnet, while different
Subnets remain separate L2 domains and communicate through their parent
VirtualNetwork's L3 routing, subject to SecurityGroup policy.

The K8s Manager is pluggable, but an implementation must provide that same
per-Subnet Layer 2 behavior regardless of the underlying mechanism. CUDN with
LocalNet can bridge a Subnet's OVN network to the corresponding physical
VLAN. DPU bridging can also satisfy the contract when it provides a shared
L2 domain for each Subnet. By contrast, OVN EVPN route advertisement and
VRF-lite as described in their L3-only forms provide routed reachability but
not a shared broadcast domain; those forms do not conform unless extended to
provide the required per-Subnet L2 behavior. This distinction keeps backend
choice transparent without weakening the VirtualNetwork and Subnet API
contract.

#### Dispatcher (Operator Composition Logic)

The `osac-operator` is the fixed **dispatcher**: it resolves the active
[NetworkClass](#networkclass), checks whether the configured manager roles can
handle the requested operation and workload target, then invokes the role
assigned by the matrix below. A registration cannot change that matrix, and OSAC does not
silently fall back to another manager. AAP receives the resource and operation
context through the common input described in
[Manager Operation Contract](#manager-operation-contract). This table and the [Manager Operation Contract](#manager-operation-contract) below define the complete target set for each operation.

| Operation identifier | Assigned role and profile behavior |
|----------------------|-----------------------------------|
| `virtual_network.create`, `virtual_network.delete` | Fabric Manager. |
| `subnet.create`, `subnet.delete` | Fabric Manager; the K8s Manager also creates or removes VM overlay resources when VMs are supported. |
| `security_group.apply`, `security_group.delete` | Fabric Manager. |
| `external_ip_pool.create`, `external_ip_pool.delete` | Fabric Manager. |
| `external_ip.allocate`, `external_ip.release` | Fabric Manager. |
| `external_ip_attachment.create`, `external_ip_attachment.delete` | Fabric Manager for `compute_instance`, `cluster`, and `baremetal_instance` targets. |
| `nat_gateway.create`, `nat_gateway.delete` | Fabric Manager. |
| `workload_attachment.move` | Fabric Manager for `cluster` and `baremetal_instance` targets. |
| `dhcp_lease.query` | Fabric Manager for `cluster` and `baremetal_instance` lease discovery. VM addresses come from Kubernetes status. |

When VMs are supported, the K8s Manager receives the Subnet operation needed
to create or remove each applicable hosting cluster's VM overlay. It is not
called for deployments without VM workloads. CaaS VIP address-pool work
follows the Subnet workflow and does not imply a VM overlay.

The dispatch table above covers **networking resources only**. Compute
resources (ComputeInstance, BaremetalInstance, Cluster) handle per-instance
network attachment through their provisioning operators — see per-service
designs at [VMaaS](/enhancements/OSAC-1435-vmaas-networking),
[CaaS](/enhancements/OSAC-1436-caas-networking),
[BMaaS](/enhancements/OSAC-1437-bmaas-networking).

#### Backend Effects and Completion Contract

The operator dispatches provider operations through the manager roles in the
[dispatcher table](#dispatcher-operator-composition-logic). The expected
provider effect for each object is:

| Object or operation | Manager responsibility |
|---|---|
| `virtual_network.create` / `.delete` | The Fabric Manager creates/removes an isolated L3 routing domain. Different VirtualNetworks remain isolated; Subnets in one VirtualNetwork are routed according to policy and remain separate L2 broadcast domains. |
| `subnet.create` / `.delete` | The Fabric Manager creates/removes the L2 segment, gateway, and address service. When VMs are supported, the K8s Manager also creates/removes a VM overlay that shares the Subnet's L2 broadcast domain on each applicable hosting cluster. |
| `security_group.apply` / `.delete` | The Fabric Manager applies/removes the complete stateful, default-deny rule set. Workload provisioning applies selected groups to each network attachment; multiple groups combine as a union of their allow rules. |
| `external_ip_pool.create` / `.delete` | The Fabric Manager registers/removes the address range in its allocation system. Fulfillment-service owns API-side capacity accounting. |
| `external_ip.allocate` / `.release` | The Fabric Manager allocates/releases a unique address under the ExternalIP UID. OSAC accepts an address only after validating the manager result and resource annotation. |
| `external_ip_attachment.create` / `.delete` | The Fabric Manager creates/removes DNAT from the ExternalIP to a workload address or selected Cluster endpoint VIP. Readiness requires a known target and installed mapping. |
| `nat_gateway.create` / `.delete` | The Fabric Manager associates the allocated ExternalIP with the VirtualNetwork and creates/removes outbound SNAT for that VirtualNetwork's IPv4 range. |
| `workload_attachment.move` | The Fabric Manager moves a physical workload port onto the selected Subnet and restores provisioning placement on detach. |
| `dhcp_lease.query` | The Fabric Manager resolves leases for its declared workload target types and returns the lease artifact. |

The manager contract defines each role's required operations and targets,
operation input, success result, retry behavior, and failure diagnostics.
Implementations scope side
effects to the OSAC resource UID, make create/apply and delete retry-safe, and
report success only after requested provider state is present or absent. OSAC
owns API validation, dependency ordering, and resource status; failed or
incomplete provider work leaves the resource non-ready for reconciliation.
Vendor-specific configuration remains inside the implementation and does not
change the API.

#### Manager Operation Contract

Every provider operation is dispatched using the same `osac_job_vars` input.
The dispatcher selects the registered implementation from the NetworkClass
role assignment, confirms its `contractVersion`, checks that the assigned role
handles the requested operation and target, then invokes the collection
task required by the manager contract. The registration does not contain
credentials or an operation list.

```yaml
osac_job_vars:
  operation: external_ip_attachment.create
  manager:
    name: example
    role: fabric
    implementationRef: acme.networking.fabric_manager
    contractVersion: v1
  resource:
    apiVersion: osac.openshift.io/v1alpha1
    kind: ExternalIPAttachment
    metadata: {}
    spec: {}
```

The full resource object supplies the resource UID, generation, metadata, and
desired spec. For target-scoped operations, the resource identifies the
workload target. The manager contract fixes the target set for each operation
and role; registrations do not enumerate it. OSAC rejects requests when the
configured roles cannot route them. A K8s Manager's presence never provides an
implicit fallback for an operation assigned to the required Fabric Manager.

Contract v1 uses one task entry point per operation. Collection task names
are shown below; the generic AAP playbook resolves
`manager.implementationRef` and selects the matching task. Implementations
may be written by any provider or vendor, but each implementation must provide
every entry point assigned to its role and meet the stated behavior.

| Operation identifier | Collection task entry point | Required backend behavior |
|---|---|---|
| `virtual_network.create` / `.delete` | `create_virtual_network` / `delete_virtual_network` | Create or remove the isolated L3 routing domain and associated allocation; keep different VirtualNetworks isolated and route between this VirtualNetwork's Subnets subject to attachment policy. |
| `subnet.create` / `.delete` | `create_subnet` / `delete_subnet` | Create or remove one L2 broadcast domain for workloads attached to this Subnet, plus any assigned K8s network resources; honor parent CIDR containment and sibling non-overlap. |
| `security_group.apply` / `.delete` | `create_security_group` / `delete_security_group` | Register or remove the complete stateful rule set scoped to the SecurityGroup's VirtualNetwork. Workload provisioning binds selected groups to each interface attachment, where their combined rules are enforced. API spec is immutable after creation. |
| `external_ip_pool.create` / `.delete` | `create_external_ip_pool` / `delete_external_ip_pool` | Register or remove the one-CIDR IPv4 allocation pool. |
| `external_ip.allocate` / `.release` | `create_external_ip` / `delete_external_ip` | Reserve or release one address under the ExternalIP UID; allocation writes the guarded OSAC annotation before success. |
| `external_ip_attachment.create` / `.delete` | `attach_external_ip` / `detach_external_ip` | Create or remove inbound translation for a target allowed by the fixed manager contract. |
| `nat_gateway.create` / `.delete` | `create_nat_gateway` / `delete_nat_gateway` | Associate the allocated ExternalIP with the VirtualNetwork and create or remove outbound SNAT for that VirtualNetwork. |
| `workload_attachment.move` | `move_network_attachment` | For `cluster` and `baremetal_instance` targets, attach the physical port when no deletion timestamp exists and restore provisioning placement when one exists. |
| `dhcp_lease.query` | `query_dhcp_lease` | For `cluster` and `baremetal_instance` targets, return an unambiguous lease for every requested attachment in the `leases` artifact. VM addresses come from Kubernetes status. |

Successful create/apply operations mean the implementation has converged the
provider state for that resource UID; delete success means the UID-owned state
is absent. Repeating an operation is safe. `external_ip.allocate` additionally
persists the confirmed IPv4 address in
`osac.openshift.io/allocated-address`, guarded by UID and generation, before
reporting success. `dhcp_lease.query` returns a `leases` AAP artifact with
`subnet_ref`, `interface`, `ip_address`, and `mac_address` for each requested
attachment. Other operations return the common `osac_result` envelope with
the schema version, operation, resource UID, observed generation, and any
operation-specific data. A failed task returns a diagnostic and no success
result; OSAC retains non-ready state and retries according to its controller
policy. There is no silent retry through another manager.

### 4.4 API Changes

The networking contract exposes create, read, and delete operations, with readiness and dependency checks performed before accepting invalid references. Workload attachment fields are create-time inputs and follow the same one-attachment limit. These technical rules implement [FR-7 through FR-9](prd.md#3-requirements).

#### Resource lifecycle enforcement

The fulfillment service enforces strict dependency constraints on both
creation and deletion of networking resources at the API layer. Invalid
operations are rejected immediately — the system never accepts a request and
defers validation to asynchronous operator reconciliation.

- **Creation:** A resource referencing another resource can only be created
  when every referenced resource is in its terminal ready state. See
  [Creation Readiness Gates](#creation-readiness-gates) for the full table.
  There are no exceptions — internal flows (auto-provisioning and default
  networking) follow the same rules by creating resources in dependency
  order and waiting for each to reach its ready state before creating the
  next.
- **Deletion:** A resource can only be deleted when no other active resource
  references it (only dependency-graph leaves are deletable). See
  [Deletion Dependency Guards](#deletion-dependency-guards) for the full
  table and dependency chain. Auto-provisioned resources are the only
  resources subject to cascade deletion on parent removal.

#### API operation constraint

The unified networking API supports only create, read, and delete operations
for networking resources. Read means `List` and `Get`; there is no tenant or
provider `Update`/`Patch` operation for a networking resource's specification
or metadata. The affected resources are `NetworkClass`, `VirtualNetwork`,
`Subnet`, `SecurityGroup`, `ExternalIPPool`, `ExternalIP`,
`ExternalIPAttachment`, and `NATGateway`.

All networking resource specification and metadata fields are immutable after
creation. A change requires deleting the resource and creating a replacement,
subject to the normal
dependency guards. The network attachment fields on `ComputeInstance`,
`Cluster`, and `BaremetalInstance` are create-time-only as well; changing a
network attachment requires replacing the parent workload. Controllers may
update status, conditions, readiness, and IP-discovery fields during
reconciliation, but those internal writes are not additional API operations.
This is the normative contract for the VMaaS, CaaS, and BMaaS designs that
reference this document; those designs inherit it and do not redefine
networking operations.

#### Implementation Details

##### Deletion Dependency Guards

The fulfillment service enforces resource dependency constraints at the API
layer. A delete request is rejected immediately with a `FailedPrecondition`
error if any active resource still references the target. Only
dependency-graph leaves — resources with no active dependents — are
deletable. The error response includes the blocking resource type so the
caller knows what to remove first.

**API-layer deletion guards (fulfillment service):**

| Resource | Reject delete if active … exist |
|---|---|
| VirtualNetwork | Subnets, SecurityGroups, NATGateways, or FabricDomains referencing this VirtualNetwork |
| Subnet | ComputeInstances, Clusters, or BaremetalInstances with network attachments referencing this Subnet |
| SecurityGroup | ComputeInstances, Clusters, or BaremetalInstances with network attachments referencing this SecurityGroup |
| ExternalIP | ExternalIPAttachments or NATGateways referencing this ExternalIP |
| ExternalIPPool | ExternalIPs referencing this pool |
| ExternalIPAttachment | (leaf — no dependents, always deletable) |
| NATGateway | (leaf — no dependents, always deletable) |
| ComputeInstance / Cluster / BaremetalInstance | Manually-created ExternalIPAttachments targeting this resource |
| NetworkClass | VirtualNetworks referencing this NetworkClass |

"Active" means the resource exists and has not been fully deleted (i.e., is
not archived). A resource that is itself being deleted (has
`deletion_timestamp` set but is still being deprovisioned) counts as active
for the purpose of these guards — the parent cannot be deleted until the
child is fully gone, not merely marked for deletion.

**Exception — auto-provisioned resources:** Resources created by the system
via `auto_external_ip_attachment` (labeled
`osac.openshift.io/auto-created`) are cascade-deleted when their parent
workload is deleted. The parent workload's delete handler in
fulfillment service initiates the cascade, and the operator-side finalizer
executes it in dependency order (ExternalIPAttachment first, then
ExternalIP). Because the system created these resources and controls the
full dependency chain, cascade deletion is safe. Manually-created
ExternalIPAttachments targeting the same workload are NOT cascade-deleted —
they block the workload's deletion until the tenant removes them.

**Operator-side guards (defense in depth):** The operator controllers retain
their existing child-resource gates as a safety net. Each parent controller lists
child Kubernetes resources before triggering the AAP deprovision job; if any children still
exist, the controller requeues instead of dispatching. This is defense in
depth — the fulfillment service API-layer rejection is the primary
enforcement point.

The full dependency chain (delete order, leaf first):

```text
ExternalIPAttachment (leaf)
  must be gone before --> ExternalIP
  must be gone before --> target ComputeInstance / Cluster / BaremetalInstance
                          (only manually-created attachments block target deletion;
                           auto-created attachments are cascade-deleted)

NATGateway (leaf)
  must be gone before --> ExternalIP
  must be gone before --> VirtualNetwork

FabricDomain (leaf)
  must be gone before --> VirtualNetwork

SecurityGroup
  must be gone before --> VirtualNetwork
  (blocked by ComputeInstances / Clusters / BaremetalInstances referencing it)

ComputeInstance / Cluster / BaremetalInstance
  must be gone before --> Subnet
  (blocked by manually-created ExternalIPAttachments targeting it)

Subnet
  must be gone before --> VirtualNetwork

ExternalIP
  must be gone before --> ExternalIPPool

VirtualNetwork
  must be gone before --> NetworkClass

NetworkClass (provider-managed, delete last)
```

##### Creation Readiness Gates

The fulfillment service enforces that every referenced resource is in its
terminal ready state before allowing creation. A create request is rejected
immediately with a `FailedPrecondition` error if any referenced resource
does not exist, is not ready, or is being deleted.

**API-layer creation gates (fulfillment service):**

| Created resource | Referenced resource | Required state |
|---|---|---|
| VirtualNetwork | NetworkClass | Ready |
| Subnet | VirtualNetwork | Ready |
| SecurityGroup | VirtualNetwork | Ready |
| NATGateway | VirtualNetwork | Ready |
| NATGateway | ExternalIP | Allocated |
| ExternalIP | ExternalIPPool | Ready |
| ExternalIPAttachment | ExternalIP | Allocated |
| ExternalIPAttachment | Target (ComputeInstance / Cluster / BaremetalInstance) | Ready |
| ComputeInstance | Subnet | Ready |
| ComputeInstance | SecurityGroup(s) | Ready |
| Cluster | Subnet | Ready |
| Cluster | SecurityGroup(s) | Ready |
| BaremetalInstance | Subnet | Ready |
| BaremetalInstance | SecurityGroup(s) | Ready |
| FabricDomain | VirtualNetwork | Ready |

The fulfillment service checks these conditions synchronously during the
create API call. If any referenced resource is in Pending, Failed, or
Deleting state, the request is rejected before persistence. The error
response includes the referenced resource and its current state.

There are no exceptions to the creation readiness rule. Internal
fulfillment service flows — `auto_external_ip_attachment` and default
networking tenant onboarding — follow the same readiness gates by
creating resources in dependency order and waiting for each to reach its
ready state before creating the next. See
[Auto-provisioning lifecycle](#auto-provisioning-lifecycle)
and [Default Resource Lifecycle](/enhancements/OSAC-1433-default-networking/design.md#default-resource-lifecycle)
for the stepped creation flows.

##### NATGateway Scope

One NATGateway per VirtualNetwork. All subnets in that VirtualNetwork use the gateway.
Per-subnet NAT association is a future enhancement.

##### Single Network Interface Attachment Constraint

VMaaS, BMaaS, and CaaS currently support at most one tenant network
attachment per workload. VMaaS and BMaaS retain repeated attachment fields
for wire/API compatibility, while CaaS retains its singular field. Requests with more than one ComputeInstance or bare-metal attachment are
rejected by API validation and the corresponding operator schema. Multiple
network interfaces per workload are future scope.

With exactly one attachment, the attachment is the **primary** attachment by
default and determines:

- Which subnet provides the **default gateway** for the resource
- Which subnet IP is used as the **DNAT target** for ExternalIPAttachment
- Which subnet IP is used as the **source** for NATGateway SNAT

**Validation and compatibility:**
- Zero or one attachment is valid
- If one BMaaS attachment exists, its existing `primary` field may be omitted
  or set to `true`; both mean the same default-route behavior. `primary: false`
  is rejected. VMaaS has no primary field, and CaaS has no primary concept.
- Omitted or empty attachment lists receive the resource-specific defaults
  described in [Attachment Presence and Defaulting](#attachment-presence-and-defaulting);
  a supplied attachment receives defaults only for missing fields.
- The complete resolved attachment list and every network-owned field are
  immutable after creation; changing them requires deleting and recreating the
  workload.

**IP assignment:** All resource types receive IPs via DHCP. For VMs,
OVN provides DHCP on the CUDN overlay. For bare-metal servers and CaaS agents,
the fabric's DHCP server assigns IPs on the network segment. The provisioning
template does NOT configure host-side networking (no static IP, gateway,
or DNS configuration) — DHCP handles it automatically.

| Subnet role | IP assignment provides (via DHCP) |
|-------------|---------------------------------------------|
| Sole/primary attachment | IP address + default gateway + DNS |

This ensures the resource has exactly one default route. Additional workload
attachments are not supported in the current API contract.

**ExternalIPAttachment:** The Fabric Manager creates a DNAT rule to the
resource's sole/primary subnet IP. The tenant does not need to specify an
interface; the single-attachment contract determines the target.

**Cluster networking:** `ClusterNetworkAttachment` is a single attachment
(one subnet for the whole cluster). More than one network interface for individual cluster nodes is
not supported by the current CaaS contract. The `primary` field does not
apply to `ClusterNetworkAttachment`.

##### Multiple Hosting Clusters Per Deployment

Multiple hosting clusters are supported per deployment. At subnet creation, the
K8s Manager creates a Kubernetes overlay on each hosting cluster and bridges it to
the fabric segment. VMs on different hosting clusters share the same subnet
via the fabric.

##### Hub Selection (Resource Placement)

The fulfillment-controller creates Kubernetes resources on the provider-configured
networking hub. Networking resources (VirtualNetwork, Subnet, SecurityGroup,
ExternalIPPool, ExternalIP, ExternalIPAttachment, NATGateway) use that hub;
the assignment remains sticky through `status.hub` for resource lifecycle and
reconciliation. The supported hub count and cross-hub behavior are defined in
[Networking Hub Support Boundary](#networking-hub-support-boundary).

The fabric can still span multiple hosting clusters where the relevant
networking feature supports that topology.

##### Cross-VirtualNetwork Communication

VirtualNetworks are isolated. Communication between them (VirtualNetwork
peering) is a separate enhancement.

##### DNS

DNS is a service-integration concern, not part of the networking API. CaaS
template roles create DNS records. A DNS API is a separate enhancement.

##### Deployments Without VM Support

If a NetworkClass has no K8s Manager, the deployment does not support VMaaS.
ComputeInstance creation is rejected because there is no K8s overlay to place
the VM on.

CaaS clusters work without a K8s Manager. MetalLB IPAddressPool creation
is handled by the Subnet controller (gated on
`NetworkClass.spec.vip_prefix_length`), not by the K8s Manager. CaaS clusters
with bare-metal nodes use fabric networking and MetalLB VIP allocation; no VM
overlay is involved.

##### CIDR Overlap

The operator validates that Subnet CIDRs do not overlap within a
VirtualNetwork at creation time.

#### NetworkClass Examples

The following fulfillment API JSON examples use deployment-selected manager
registration names, not vendor fields in tenant resources. `capabilities` is
system-derived from the registered managers. Each example is an alternative
profile; a deployment uses one active NetworkClass at a time.

**Fabric-backed profile with VM support:**

```json
{
  "id": "connected-region-a",
  "metadata": {"name": "connected-region-a"},
  "fabricManager": "fabric-manager-a",
  "k8sManager": "vm-overlay-manager-a",
  "spec": {},
  "capabilities": {
    "supportsIpv4": true,
    "supportsIpv6": false,
    "supportsDualStack": false
  }
}
```

**Fabric-backed profile without VM support:**

```json
{
  "id": "baremetal-region-a",
  "metadata": {"name": "baremetal-region-a"},
  "fabricManager": "fabric-manager-b",
  "spec": {},
  "capabilities": {
    "supportsIpv4": true,
    "supportsIpv6": false,
    "supportsDualStack": false
  }
}
```

#### UX Alignment

The PRD puts the unified networking UI and user documentation in scope for
OSAC 0.2; screen-level interaction design and implementation are tracked by
[OSAC-2226](https://redhat.atlassian.net/browse/OSAC-2226). The UI must use the
API contract in this design: create/read/delete resource operations, shared
workload pickers, and readiness/dependency errors.

| UI/API surface | Current UI client coverage | Target alignment |
|---|---|---|
| VirtualNetwork, Subnet, SecurityGroup | `osac-ui/libs/ui-components/src/api/v1/networking.ts` provides list/get and create/delete hooks. | Keep these flows and show shared resource relationships. Remove in-place configuration editing; replace a resource after its dependents are removed. |
| SecurityGroup rules | The current client has a `useUpdateSecurityGroup` placeholder that does not call the API, while the UI presents rule editing. | Do not expose rule updates under the create/read/delete contract. Rule changes require replacement. |
| ExternalIP and NATGateway | The current networking client reads ExternalIP state and joins NATGateway references to addresses. | Complete create/read/delete and show allocation state; describe ExternalIP as outside the VirtualNetwork without promising Internet reachability. |
| ExternalIPAttachment | No create/delete hook is defined in the current networking client. | Add the shared inbound-access workflow and target-specific endpoint selection through OSAC-2226. |
| NetworkClass and ExternalIPPool | Provider details are not managed by the current tenant networking client. | Give Cloud Infrastructure Admins a provider view for manager roles, supported capabilities, and pool capacity; keep backend selection provider-only. |
| Workload attachment pickers | Existing API helpers filter ready Subnets and SecurityGroups by VirtualNetwork. | Use the same authorized, ready-resource choices in VMaaS, CaaS, and BMaaS workflows. |

The detailed pages and interaction patterns remain in OSAC-2226. This
alignment records the API-visible behavior that those screens must preserve.

#### Workflow Description: End-to-End Flows

This section shows how the unified networking API works from the tenant's
perspective. The flows are the same regardless of which Fabric Manager or
K8s manager the provider has deployed.

These provider setup, networking setup, attachment, and external access flows
describe `global.networking.provisioningEnabled=true`. The disabled branch is
defined in [Provider Networking Control](#provider-networking-control); API
readiness, validation, and deletion constraints apply in both modes. [User]

##### Provider Setup

1. Provider deploys hosting cluster(s) and fabric controller
2. Provider creates NetworkClass for the deployment (provider-only,
   tenants never see it)
3. Provider creates ExternalIPPool:

```bash
osac admin create externalippool \
  --network-class connected-region-a \
  --cidrs 203.0.113.0/24 \
  --ip-family ipv4 \
  --name external-pool-1
```

The Fabric Manager registers the IP range in its IP address management (IPAM) system for allocation.

##### Networking Setup (Same for All Resource Types)

The tenant creates networking resources. This workflow is identical
regardless of whether the tenant plans to run VMs, clusters, or bare-metal
servers.

**Create VirtualNetwork:**

```bash
osac create virtualnetwork --network-class connected-region-a --cidr 10.0.0.0/16 \
  --name my-net
```

The Fabric Manager creates an isolated tenant segment on the fabric.

**Create Subnet:**

```bash
osac create subnet --virtual-network my-net --cidr 10.0.1.0/24 \
  --name my-subnet
```

The Fabric Manager creates a fabric segment (e.g., VLAN) for the subnet.
If the NetworkClass has a K8s manager, it also creates a K8s overlay on each
hosting cluster and bridges it to the fabric segment. After this step, VMs placed in the
overlay and bare-metal servers with switch ports on the fabric segment are in the
same Layer 2 domain.

**Create SecurityGroup:**

```bash
osac create security-group --virtual-network my-net --name my-sg \
  --ingress "protocol:tcp,port:443,source:0.0.0.0/0"
```

The Fabric Manager creates ACL rules on the fabric.

##### Resource Creation (Differs by Type)

The networking setup above is shared. Only the workload placement step
differs internally — the tenant-facing resource model and CLI choices are the
same for all types.

**ComputeInstance (VM):**

```bash
osac create computeinstance --template ocp_virt_vm \
  --network-attachment subnet=my-subnet,security-groups=my-sg \
  --name my-vm
```

VM is placed in the K8s overlay namespace on a hosting cluster. Because the
overlay is bridged to the fabric, the VM is directly on the fabric segment
and gets an IP from the subnet CIDR.

**BaremetalInstance:**

Bare-metal servers have multiple physical interfaces. The tenant discovers
available network ports via the BareMetalInstanceType API — each
BareMetalInstanceType lists its network ports with name, role, type, speed,
and description (see
[HostType and BareMetalInstanceType](#hosttype-and-baremetalinstancetype)). Given the port identifiers, the tenant specifies which
interface to attach to the subnet. The current BMaaS contract accepts one
entry in the repeated `network_attachments` field, mapping one physical
interface to one subnet. If `interface` is omitted, fulfillment defaults to
the first `fabric` port from `BareMetalInstanceType.network_ports`.

Single interface (simple case):

```bash
osac create baremetalinstance --template bcm_h100 \
  --network-attachment interface=data-0,subnet=my-subnet,security-groups=my-sg \
  --name my-server
```

The API retains the repeated field for compatibility, but only one attachment
is supported. The Fabric Manager configures the selected host switch port on
the corresponding fabric segment and the interface gets an IP from the
subnet's CIDR.

Validation rules:
- At most one attachment is accepted
- All referenced subnets must belong to the same VirtualNetwork
- The `interface` must reference a valid port name from the BareMetalInstanceType's
  network ports list

**Cluster:**

```bash
osac create cluster --template ocp_4_17_small \
  --network-attachment subnet=my-subnet,security-groups=my-sg \
  --node-set workers=large,size=3 --name my-cluster
```

For v0.2, **CaaS supports bare-metal node sets only**. VM-based cluster node sets
are architecturally possible but deferred. The fulfillment service resolves
the interface from the BareMetalInstanceType (`fabric_interface` — first port
with role `fabric`) and stores it on the node set. The worker controller passes
that stored value to BMaaS; BMaaS handles the host's network attachment as part
of its provisioning lifecycle.
See [CaaS Networking](/enhancements/OSAC-1436-caas-networking) for the detailed flow.

Cluster nodes have multiple physical interfaces. Unlike BaremetalInstance
(where the tenant specifies interfaces directly), for clusters the
**system** resolves the interface from each node set's BareMetalInstanceType
`network_ports` list.
The tenant specifies which subnet to use (one per cluster); the system maps it to the
correct physical interfaces based on each node set's BareMetalInstanceType.

Where the selected profile supports a workload, its provisioning flow joins
that workload to the selected Subnet. The assigned manager implementations
apply the shared resource semantics; the API does not distinguish virtual-machine,
bare-metal, or cluster networking resources.

##### External Access (Same for All Resource Types)

External access uses the same ExternalIPAttachment and NATGateway resources
for every supported workload type. The assigned manager implementation
handles the required DNAT or SNAT operation according to its role contract.

**Allocate ExternalIP:**

```bash
osac create externalip --pool external-pool-1 --name my-ip
```

The Fabric Manager allocates a free address from the selected pool and writes
it to the ExternalIP annotation. OSAC validates
that annotation and writes status as defined in [ExternalIP Address Selection
and Ownership](#externalip-address-selection-and-ownership) (e.g.,
203.0.113.45).

**Attach for inbound access (DNAT):**

```bash
# Attach to a VM
osac create externalipattachment --externalip my-ip \
  --compute-instance my-vm --name vm-att

# Attach to a bare-metal server (new target type)
osac create externalipattachment --externalip my-ip \
  --baremetal-instance my-server --name bm-att

# Attach to a cluster API server (new target type + endpoint)
osac create externalipattachment --externalip my-ip \
  --cluster my-cluster --target-endpoint api --name api-att
```

The Fabric Manager creates a DNAT rule: external IP → resource's subnet IP.
Each resource (ComputeInstance, BaremetalInstance) is associated with one
tenant subnet and has one fabric IP — the DNAT targets that IP directly. The
ExternalIP is attached to the resource, not to a specific interface; the
Fabric Manager routes to the resource's sole/primary subnet IP.

##### Cluster ExternalIPAttachment flow

For VMs and bare-metal instances, the DNAT target is the resource's fabric IP —
straightforward. For clusters, the DNAT target is a service-level VIP
(API server or ingress) that is discovered during cluster provisioning.
The VIP allocation is decoupled from the networking layer:

1. CaaS template creates MetalLB LoadBalancer Services for API server
   and ingress. MetalLB allocates VIPs from its IPAddressPool (created
   by k8s_manager at subnet creation).
2. Template discovers the allocated VIPs and writes them to ClusterOrder
   resource status (`apiEndpoint`, `ingressEndpoint`)
3. Feedback controller syncs VIPs to the Cluster object in the
   fulfillment service as `api_endpoint` and `ingress_endpoint` fields
4. ExternalIPAttachment controller reads the VIP from ClusterOrder
   status → calls Fabric Manager to create DNAT: external IP →
   internal VIP
5. ExternalIPAttachment transitions to Ready

The tenant can inspect the allocated VIPs:

```bash
osac get cluster my-cluster -o yaml
# api_endpoint: 10.0.5.20
# ingress_endpoint: 10.0.1.50
```

ExternalIPAttachments for clusters follow the same creation readiness
rules as all other resources: the cluster must be in Ready state before
an ExternalIPAttachment targeting it can be created. Auto-provisioned
ExternalIPAttachments (via `auto_external_ip_attachment`) also follow
the readiness rules — they are created by the fulfillment service
internal reconciler only after both the ExternalIP is Allocated and the
cluster is Ready (see below).

#### Auto-provisioning lifecycle

Auto ExternalIP attachment provisioning (described in per-service
EPs and [Default Networking](/enhancements/OSAC-1433-default-networking)) is a
multi-step process that follows the same creation readiness rules as
tenant-initiated operations. The fulfillment service controls the
timing and creates each resource only after its dependencies are ready.

*Step 1 — synchronous (during the create API call):*

The fulfillment service validates pool capacity, creates ExternalIP
records in PostgreSQL, and decrements pool capacity — within the same
API transaction as the workload creation. If the pool is exhausted, the
call fails and no resources are persisted (including the parent
workload). The ExternalIP starts in **Pending** state. For clusters,
two ExternalIPs are created (one for API, one for ingress). For
ComputeInstances and BaremetalInstances, one ExternalIP is created.

ExternalIPAttachments are **not** created at this point — their
dependencies (ExternalIP Allocated + target Ready) are not yet met.

*Step 2 — asynchronous (ExternalIP reconciliation):*

The fulfillment service reconciler pushes ExternalIP resources to the hub
cluster. The osac-operator dispatches `external_ip.allocate` to the assigned
Fabric Manager. The manager durably allocates an address
and writes the standard allocated-address annotation. OSAC validates the job
result and annotation, then writes status under [ExternalIP Address Selection
and Ownership](#externalip-address-selection-and-ownership). The ExternalIP
then transitions to **Allocated**, and the fulfillment service receives the
status update via the Signal remote procedure call (RPC).

*Step 3 — asynchronous (deferred ExternalIPAttachment creation):*

Once both prerequisites are met — the ExternalIP is **Allocated** and
the target workload is **Ready** — a fulfillment service parent-resource
reconciler creates the ExternalIPAttachment. This new reconciler is separate
from the existing ExternalIP and ExternalIPAttachment synchronization
controllers. It follows the standard creation readiness gate: the attachment
is only persisted when
its ExternalIP is Allocated and its target is Ready. The
ExternalIPAttachment starts in **Pending** state and is pushed to the
hub cluster by the reconciler.

*Step 4 — asynchronous (ExternalIPAttachment reconciliation):*

The osac-operator ExternalIPAttachment controller verifies its
preconditions (ExternalIP Allocated + target has a known IP) and
dispatches to AAP → Fabric Manager creates the DNAT rule →
ExternalIPAttachment transitions to **Ready**.

*ExternalIPAttachment controller preconditions per target type:*

| Target type | Required precondition | Source of target IP |
|-------------|----------------------|---------------------|
| ComputeInstance | `compute_network_attachment_statuses` populated with primary attachment's `ip_address` | Feedback controller reads KubeVirt VMI network status, writes `ComputeNetworkAttachmentStatus` per attachment |
| Cluster | `status.apiEndpoint` or `status.ingressEndpoint` populated on ClusterOrder resource | MetalLB allocates VIP from IPAddressPool, template discovers and writes to ClusterOrder status |
| BaremetalInstance | `status.networkAttachmentStatuses[].ipAddress` populated for the primary interface | Operator queries Fabric Manager's DHCP lease API via dispatcher (`query_dhcp_lease` role) after provisioning completes; matches network-interface address (from the BareMetalHost `osac.openshift.io/interface-macs` annotation) to assigned IP; operator writes to resource status |

The controller uses the existing requeue pattern: if the precondition
is not met, it returns `ctrl.Result{RequeueAfter: interval}` and
retries until the target IP appears. This is the same pattern used
today for the `VirtualMachineReference` check on ComputeInstance
targets.

*IP discovery — DHCP-based host networking:*

All host-side IP assignment uses DHCP. The fabric's DHCP server (managed
by the Fabric Manager as part of the network segment infrastructure) assigns IPs
to hosts when they boot on the subnet. OSAC does not pre-allocate IPs
or configure host-side networking — DHCP handles IP address, gateway,
prefix, and DNS automatically.

After the host receives its IP address, the operator writes it to resource
status for ExternalIPAttachment target resolution and tenant visibility.

IP discovery mechanism per service type:

| Service | Discovery source | Who writes status | Status field |
|---------|-----------------|-------------------|-------------|
| VMaaS | KubeVirt virtual machine instance (VMI) `status.interfaces[].ipAddress` | osac-operator feedback controller → Signal RPC → fulfillment service | `ComputeInstanceStatus.compute_network_attachment_statuses[].ip_address` |
| CaaS | Cluster agent network status | osac-operator feedback controller → Signal RPC → fulfillment service | `ClusterOrderStatus.nodeSets[].agents[].ipAddress` (operator-internal) |
| BMaaS | Operator queries Fabric Manager's DHCP lease API via dispatcher (`query_dhcp_lease` role) after provisioning completes; matches the network-interface address — from the BareMetalHost `osac.openshift.io/interface-macs` annotation — to the DHCP-assigned IP, falling back to server name for named fabric servers (see [BMaaS OQ#4 — Resolved](/enhancements/OSAC-1437-bmaas-networking/design.md#4-how-is-the-hosts-runtime-ip-discovered-after-network-reconfiguration-resolved)) | bare-metal-fulfillment-operator dispatches `query_dhcp_lease` → writes to resource status → feedback controller → Signal RPC → fulfillment service | `BareMetalInstanceStatus.network_attachment_statuses[].ip_address` |

The Fabric Manager's `move_network_attachment` role is switch-side
only — it moves a host's fabric port from one network segment to another
(`from_vnet_name` → `to_vnet_name`, either side optional). Attach and
detach are the **same primitive**: on provision the port moves from a
**provisioning network** to the tenant subnet's network segment; on deletion it
moves back to the provisioning network. The role operates purely against the
fabric (without looking up a Subnet resource) and is keyed on plain segment names, so the caller
resolves a `subnetRef` → tenant segment name and supplies the provisioning
network name from configuration. Detach is a no-op if the port is not on the
named segment, so re-runs and unexpected states are safe.

One role handles two cases: a bare-metal server uses a fabric network interface
on the provisioning network while idle for Metal3 inspection, and a cluster
agent moves from the provisioning network to the tenant network. The **timing**
of the move differs by workload:

- **Bare-metal instance:** Move happens **POST-provisioning** (provision on the provisioning
  network → move to tenant network → reboot so the host requests a new address through DHCP on the tenant
  network). This achieves isolation-until-ready: the tenant cannot reach the
  server during imaging/first-boot.
- **Cluster worker:** the BMaaS provisioning flow moves the port **POST-OS-provisioning** (the host is provisioned
  on the provisioning network, then the port moves to the tenant network and the
  host reboots before it joins the cluster installation flow).

Once on the tenant network, the host receives an IP from the fabric's DHCP server
automatically. A single AAP job template serves both directions, deriving onboard
(provisioning network → tenant) vs. offboard (tenant → provisioning network) from
the resource's `deletionTimestamp`. See [BMaaS — Provisioning Network and Port
Moves](/enhancements/OSAC-1437-bmaas-networking/design.md#provisioning-network-and-port-moves).

IP discovery for BMaaS is a separate dispatcher call. After
`reconcileProvisioning` completes and the host has received a DHCP
lease, the operator dispatches `query_dhcp_lease` — this role queries
the Fabric Manager's DHCP lease API for the subnet and matches the
server's network-interface address to find the corresponding DHCP-assigned IP.
Bare-metal hosts are not named fabric servers, so the lease is matched
by network-interface address, which the operator supplies from the host's
`osac.openshift.io/interface-macs` BareMetalHost resource annotation; named
fabric servers such as CaaS agents fall back to matching by server name.

*NATGateway controller preconditions:*

The NATGateway controller has two preconditions before dispatching the
SNAT rule creation:

| Precondition | Source |
|-------------|--------|
| Referenced VirtualNetwork must be Ready (fabric segment provisioned) | VirtualNetwork resource status |
| Referenced ExternalIP must be Allocated (have an allocated address) | ExternalIP resource status |

If either precondition is not met, the NATGateway controller requeues.
This prevents dispatching to AAP before the VirtualNetwork's fabric segment exists
(no segment to attach the SNAT rule to) or without a valid SNAT source
address.

*Auto-provisioned resource labeling:*

All auto-created resources receive the label
`osac.openshift.io/auto-created: "true"`. Auto-provisioned
ExternalIPs also receive a parent-resource label
`osac.openshift.io/auto-created-for: <resource-id>` so that the
cleanup logic can find orphaned ExternalIPs directly, even if the
intermediate ExternalIPAttachment has already been deleted.

*Auto-provisioned resource cleanup on parent deletion:*

The parent resource's finalizer uses a phased requeue approach to
ensure correct ordering:

1. Query ExternalIPAttachments labeled `auto-created` targeting
   this resource. Issue delete for each. Requeue.
2. On next reconcile: check if all ExternalIPAttachments are fully
   deleted (including their own finalizers completing the DNAT rule
   removal). If not, requeue.
3. Once all ExternalIPAttachments are gone: query ExternalIPs labeled
   `auto-created-for: <this-resource>`. Issue delete for each.
   Requeue.
4. On next reconcile: check if all ExternalIPs are fully deleted. If
   not, requeue.
5. Once all ExternalIPs are gone: proceed with parent resource
   deletion.

If cleanup fails permanently (after N retries): finalizer is removed,
parent resource deleted, orphaned resources left in cluster. Orphaned
resources are identifiable by the `auto-created-for` label.

**Enable outbound NAT (SNAT):**

```bash
osac create externalip --pool external-pool-1 --name nat-ip
osac create natgateway --virtual-network my-net --externalip nat-ip \
  --name my-nat
```

The Fabric Manager creates a SNAT rule for the VirtualNetwork: all egress
traffic from its CIDR is source-NATted to the ExternalIP. This applies to all
resources in the VirtualNetwork — VMs, bare-metal servers, and cluster nodes —
because they share the provider network.


### 4.5 Scalability and Performance

Provider-side work is asynchronous and dispatched per networking resource. In VM-enabled profiles, Subnet provisioning fans out to each applicable hosting cluster for VM overlay creation; the amount of that work therefore grows with the number of hosting clusters. API-side work consists of resource validation, dependency checks, and persistence. The design sets no throughput or latency target, so release capacity must be assessed against the deployment's resource counts and hosting-cluster topology.

### 4.6 Security Considerations

VirtualNetworks define tenant isolation, and SecurityGroups define permitted traffic across workload types. The implementation assigned to the SecurityGroup operation enforces the same policy semantics for each supported workload, including VM traffic after the Kubernetes manager connects its overlay. Input validation rejects unsupported address families, non-canonical or out-of-range CIDRs, invalid references, and invalid attachment shapes before provider dispatch. ExternalIP allocation is associated with the owning resource UID so retries cannot silently transfer an address reservation to another object.

### 4.7 Failure Handling and Recovery

Fulfillment rejects creates whose referenced resources are missing, deleting, or not ready, and rejects deletes while active dependents remain. The detailed gates and dependency tables are in [Creation Readiness Gates](#creation-readiness-gates) and [Deletion Dependency Guards](#deletion-dependency-guards). Manager failures leave resources non-ready for reconciliation; an ExternalIP allocation is accepted only after the manager result and the provider-owned address annotation pass validation.

#### Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Fabric manager complexity | One Ansible role handles all networking concerns | Clear interface contract per operation; tested independently per manager |
| K8s-to-fabric bridge failure | VMs unreachable from fabric | K8s Manager validates bridge connectivity at subnet creation; subnet stays Pending until bridge is confirmed |
| CaaS endpoint readiness | Cluster endpoint VIPs may not be available when provisioning starts | CaaS records the API and ingress endpoints before the Cluster becomes Ready; create is rejected until the Cluster is Ready and the requested endpoint exists. |
| ExternalIPAttachment target validation | Target may not exist or may be deleting | Fulfillment rejects missing, deleting, or non-Ready targets at create time. No forward-reference Pending state is supported. |
| CIDR overlap | Overlapping subnets cause routing ambiguity | Operator validates at creation time; rejected with clear error |

#### Provider Networking Control

`global.networking.provisioningEnabled` is the shared Helm boolean and
defaults to `true`. Enclave Wizard presents the same setting during
installation and upgrade. Both operator areas use that value; a change takes
effect after the coordinated rollout completes. This is an installation or
upgrade setting, not a live OSAC console toggle.
[PRD: FR-10] [User]

The setting controls provider-network operations only. Networking APIs and
OSAC object lifecycle remain available in both modes, with the same
authorization, tenant isolation, validation, defaulting, available API actions,
and dependency rules. Logical resource status continues to reflect the OSAC
object lifecycle; it does not claim a provider change occurred. The setting
does not change the enabled-services list. [PRD: FR-10] [User]

When disabled, OSAC submits no provider-network operations for any supported
network resource, regardless of the selected manager profile. No network
configuration, address allocation, routing, cleanup, DHCP discovery, or port
movement is performed, and OSAC submits no substitute or no-op work. Ordinary
VM and cluster provisioning, host provisioning, inventory, hardware, and power
management remain available. [User]

##### Resource operation behavior

API create/read/delete semantics, immutable fields, readiness preconditions,
and deletion dependency guards apply in both modes. All networking specification
and metadata updates remain rejected, including SecurityGroup rule changes; the
switch does not add an Update/Patch operation. OSAC object status continues to
reflect logical lifecycle and unmet prerequisites. [PRD: FR-8, FR-9, FR-10]

| Resource | Enabled provider create / rejected update / delete | Disabled provider create / rejected update / delete |
|----------|---------------------------------------------------|----------------------------------------------------|
| VirtualNetwork | Create and remove the manager's tenant network; specification updates rejected | OSAC object create/delete remains available; no provider network is created or removed; specification updates rejected |
| Subnet | Create/remove the selected managers' subnet/network resources; specification updates rejected | No provider segment, overlay, namespace, or pool is provisioned or removed; specification updates rejected |
| SecurityGroup | Create/delete the manager policy; specification and metadata updates rejected | OSAC create/delete remains available; no provider policy operation; existing backend rules may remain effective until provider-side cleanup; updates rejected |
| ExternalIPPool | Create/remove provider pool integration; specification updates rejected | Logical create/delete remains available without creating or removing a provider pool; specification updates rejected |
| ExternalIP | The Fabric Manager durably allocates an address and writes the allocated-address annotation; OSAC validates the result and annotation before recording the address. On deletion, OSAC returns pool capacity only after confirmed provider release; specification updates rejected | No provider allocation or release occurs. Without a confirmed allocation it remains Pending with an empty address; a previously confirmed allocation retains its real validated address and Allocated state, marked last-known while disabled. Logical deletion releases OSAC capacity but may leave a provider reservation for manual cleanup. Specification updates rejected |
| ExternalIPAttachment | Create/remove inbound routing for an Allocated IP and Ready target; specification updates rejected | No inbound routing is configured; creation still requires the existing API prerequisites, so a newly unallocated ExternalIP cannot satisfy them; specification updates rejected |
| NATGateway | Create/remove outbound routing for the supported profile; specification updates rejected | No outbound routing is configured; creation retains the VirtualNetwork Ready and ExternalIP Allocated gates; specification updates rejected |

Read (List/Get) continues to expose persisted desired state and conditions.
While provider networking is disabled, VirtualNetwork, Subnet, SecurityGroup,
ExternalIPPool, ExternalIPAttachment, and NATGateway report `Ready=True`, reason
`ProvisioningDisabled`, and a message identifying the unavailable provider
operation, but only after the existing logical
preconditions are satisfied. Dependency checks remain specific to the
existing API gates: an ExternalIPAttachment or NATGateway requires ExternalIP
`state=Allocated`; references such as VirtualNetwork, ExternalIPPool, and
workload targets require `Ready`. A backend-confirmed ExternalIP that remains
`Allocated` during disablement satisfies an `Allocated` reference gate even
though its own condition is `Ready=False`/`ProvisioningDisabled`; its last-known
address does not imply provider reachability. If an existing resource's logical
prerequisite is unmet, it remains in its ordinary waiting state, and new create
requests retain their existing API precondition errors. This is logical OSAC
readiness only; it does not claim provider connectivity, policy enforcement, or
routing. `Skipped` describes the message/result and is not a Kubernetes
condition status.

If a legacy ExternalIPAttachment points to an ExternalIP whose fake
`0.0.0.0` address is cleared during migration, it remains
`phase=Progressing` with `Ready=False`, reason `ExternalIPNotAllocated`, and a
message that routing is waiting for a real allocation. It launches no routing
job while disabled. This waiting case does not weaken API validation: new
ExternalIPAttachments still require an Allocated ExternalIP and a Ready target.

ExternalIP is the allocation exception. If no real manager-confirmed
allocation has completed, it remains `state=Pending`, `phase=Progressing`, with an empty
`address` and `Ready=False`, reason `ProvisioningDisabled`; the message says
allocation is waiting for provider networking to be enabled. The backend remains
the address allocator when enabled; OSAC does not select an address from the
pool CIDR. If a real allocation completed before the setting was disabled,
retain its last manager-confirmed `state=Allocated` and address from the
validated allocated-address annotation, set
`phase=Progressing` while provider networking is disabled, and set
`Ready=False`, reason `ProvisioningDisabled`. The message identifies the
address as last-known information that is not being reconciled or guaranteed
reachable. If an allocation already in progress finishes while networking is
being disabled, OSAC waits for terminal job state and records the allocation
only after validating the successful result envelope and the real IPv4 address
in the manager-written annotation. It does not dispatch a new allocation or
compensating operation. Never write `0.0.0.0` or another placeholder. On rollout
to this behavior, convert
existing disabled-mode `0.0.0.0` records to Pending with an empty address;
attachments that depended on the placeholder remain waiting until a real
allocation is confirmed. [User]

##### In-Flight Provider Work and Deletion

Any provider-network operation already in progress must reach a terminal
state before OSAC reports work skipped or completes deletion. If an operation
cannot be confirmed terminal, the resource remains pending and deletion stays
incomplete so the operation can be checked again. If the operation completes
before disablement takes effect, OSAC records that confirmed result and does not
start a compensating cleanup operation. This applies to work spanning multiple
manager targets, network configuration, address allocation, host port moves,
and DHCP discovery. [User]

Once in-progress provider work is terminal, live resources report provider
work skipped. Live resources with unmet logical prerequisites remain waiting as
specified above. Deletes still respect dependency guards and auto-created child
deletion order, then complete OSAC object deletion without provider cleanup.
Logical cascade deletion does not imply provider cleanup. Turning the setting
off also does not withdraw existing provider state or public exposure. Existing
ExternalIPAttachment DNAT and NATGateway SNAT routes, allocated addresses,
segments, security rules, overlays, and port placements may remain effective
until manual/provider-side cleanup. Disabled reconciliation does not remove
them. Existing hosts are not moved back to provisioning connectivity. [User]

When the setting is enabled again after rollout, Pending ExternalIPs may proceed
to provider allocation. An automatic attachment is created only after a real
allocation and workload readiness are confirmed. Deleted OSAC objects are not recreated to
clean up provider leftovers; those leftovers require provider/manual cleanup.

##### Workload flow boundary

The manager-backed flows below apply when the setting is enabled. When it is
disabled, ordinary workload provisioning remains available for API-valid
requests:

- VMaaS provisions VMs on platform default networking; disabled mode does not
  require or apply tenant subnet placement.
- BMaaS provisions new hosts on baseline provisioning connectivity, skips
  tenant port movement and networking handoff reboots, and performs no tenant
  DHCP queries or tenant-IP feedback. Each incomplete network phase skipped
  after disablement uses `Status=Unknown`, reason `ProvisioningDisabled`, and a
  message identifying the skipped operation. Confirmed phases from before
  disablement retain `True`; legacy `True`/`Skipped` conditions from the current
  disabled path are normalized to `Unknown`/`ProvisioningDisabled`. A skipped
  network phase counts as complete only when its condition is `True` or
  `Unknown` with that exact reason;
  normal power control remains active, so `--auto-up` still powers
  the host on and it remains on provisioning connectivity. Workload Ready means
  host provisioning completed, not tenant connectivity.
- CaaS continues cluster and worker provisioning on baseline platform/
  provisioning connectivity. That connectivity must already support
  assisted-service/control-plane access, required DNS and address services,
  and installation/image dependencies. Tenant-network routing, OSAC-managed
  tenant VIP pools, and public ExternalIP routing are not supplied by OSAC in
  disabled mode; an environment that relies on those resources must provide
  adequate baseline connectivity before cluster installation can succeed.

Tenant defaulting and all API validation remain in force. Disabled mode is
not an exemption for missing defaults, invalid interfaces, unsupported
workload types, or allocation prerequisites. Automatically created
ExternalIPs retain pool/capacity validation; an ExternalIP that stays
unallocated does not create an ExternalIPAttachment; one is created only after its ExternalIP
is Allocated and the workload is Ready. Ordinary workload provisioning does
not wait for skipped provider work to produce an address. [User]

For default tenant networking, onboarding creates the logical default
VirtualNetwork, Subnet, and SecurityGroup through their normal API paths. It
does not create the default ExternalIP or NATGateway while provider networking
is disabled: no ExternalIP can be allocated, and NATGateway creation retains
the existing `Allocated` prerequisite. Once those logical defaults are ready,
`DefaultNetworkingReady` is true with reason `ProvisioningDisabled`, allowing
virtual-machine, bare-metal, and cluster resources to use the same default attachment resolution
and API validation. This readiness does not assert provider connectivity or
outbound NAT. When provider networking is enabled again, default networking
creates the missing ExternalIP and NATGateway through the normal allocation
and readiness gates. [User]

### 4.8 RBAC and Tenancy

Role-based access control (RBAC) requires no new authorization role. Provider-owned NetworkClass and ExternalIPPool resources remain provider-managed. Every tenant-scoped networking resource and workload attachment carries the `osac.openshift.io/tenant` annotation on its Kubernetes representation. Controller-created child resources also preserve the applicable `osac.openshift.io/owner-reference` annotation when OSAC owns them; references between resources do not by themselves establish ownership. Existing fulfillment service authorization continues to enforce tenant access. A conforming manager must preserve these annotations and must not act on resources outside its authorized tenant scope. The assigned manager implementation enforces policy uniformly for each supported workload.

### 4.9 Extensibility and Future-Proofing

Manager registrations separate provider integrations from the tenant resource model. A deployment can use any implementation that meets the required Fabric Manager contract and declared capability requirements. When VM workloads are supported, it can also use any conforming K8s Manager implementation for VM overlay operations. Neither choice adds a tenant-facing backend selector or workload-specific networking resources. Internal IP pools remain manager-managed with manager-provided defaults; they are not tenant API resources or NetworkClass settings. The capability vocabulary remains operator-defined, so adding a new capability still requires an operator update.

## 5. Interface Changes

The following interface changes map the technical design to the stable PRD requirements. Schemas, manager integration, and API lifecycle behavior are defined in Sections 4.2 through 4.4; failure handling is in Section 4.7.

### IC-1: Shared networking resource API

**Requirements:** FR-1, FR-2, FR-3, FR-4, FR-5, FR-8, FR-9

The fulfillment service public API and operator resource surfaces cover NetworkClass, VirtualNetwork, Subnet, SecurityGroup, ExternalIPPool, ExternalIP, ExternalIPAttachment, and NATGateway. The API actions and resource lifecycle constraints are defined in [API Changes](#44-api-changes).

### IC-2: Workload network attachments

**Requirements:** FR-2, FR-3, FR-7, FR-9

ComputeInstance, Cluster, and BaremetalInstance accept their resource-specific network attachment at creation. Each accepts at most one tenant attachment; the bare-metal attachment may identify one exposed physical interface.

### IC-3: External access

**Requirements:** FR-4, FR-5

ExternalIPAttachment exposes inbound access to supported workload endpoints. NATGateway exposes optional outbound source identity. The resource and endpoint shapes are defined in [API Extensions](#api-extensions).

### IC-4: Provider manager configuration

**Requirements:** FR-6

Provider configuration selects a Fabric Manager and an optional Kubernetes manager through NetworkClass and manager registrations. Tenant APIs do not expose those implementation choices.

### IC-5: Provider networking control

**Requirements:** FR-10

The shared `global.networking.provisioningEnabled` installation setting and Enclave Wizard control select whether OSAC submits provider networking operations. The default is enabled; the setting takes effect through rollout.

### IC-6: Unified networking UI and documentation

**Requirements:** FR-11

The unified UI supports the shared create/read/delete resource lifecycle,
authorized workload pickers, provider views for NetworkClass and ExternalIP
pool capacity, and accurate readiness and reachability information. User
documentation explains those workflows and the disabled-provider behavior.
Screen-level design and implementation are tracked by OSAC-2226.

## 6. Alternatives Considered

**Keep service-specific networking.** This avoids shared-model changes in the
short term, but preserves separate tenant workflows and provider integrations
for the VM, cluster, and bare-metal services. It does not meet the shared requirements and is
rejected.

**Tenant-selected NetworkClass.** This lets tenants choose a NetworkClass per
VirtualNetwork, but exposes provider implementation choices and makes tenant
networking depend on infrastructure details. Provider-owned selection keeps
the tenant contract consistent, so this option is rejected.

**Per-operation manager drivers.** Separate drivers for networks, ACLs,
address allocation, ingress, and egress could allow different products for
each task. It does not match the single physical fabric that applies those
operations and adds composition and validation complexity, so one fabric
manager is preferred.

**Separate Kubernetes ACL manager.** A Kubernetes NetworkPolicy layer could
add enforcement inside the overlay, but would duplicate fabric SecurityGroup
policy for VMs already bridged to the fabric. The design keeps one enforcement
point and rejects this additional layer.

**Workload-scoped VirtualNetworks.** A virtual-machine/bare-metal/cluster scope field could make
service-specific provisioning explicit, but it would prevent mixed workload
subnets and couple tenant resources to placement details. A shared,
infrastructure-agnostic networking model is preferred.

**Lazy subnet provisioning.** Waiting until workload placement to select a
manager could defer provider setup, but leaves subnet readiness and manager
selection ambiguous. The design provisions the selected fabric and optional
Kubernetes overlay when the Subnet is created.

#### Drawbacks

This design requires K8s-to-fabric connectivity in every deployment that
hosts VMs. The K8s Manager must bridge the OVN overlay to the physical
fabric while preserving the per-Subnet L2 contract. A deployment without VM
workloads, including bare-metal deployments with or without CaaS, does not
need a K8s Manager; every deployment still requires a Fabric Manager. MetalLB
IPAddressPool creation for CaaS VIP allocation is handled by the Subnet
controller, not by the VM-overlay operation.

The trade-off is justified by infrastructure-agnostic networking resources:
the same tenant resources serve virtual machines, clusters, and bare-metal
instances; the role contracts provide uniform policy enforcement; and tenant
resources do not need per-workload variants. When VM workloads are supported,
the K8s Manager bridges overlays to the selected Subnet while preserving its
shared L2 broadcast domain.

## 7. Observability and Monitoring

No new standalone metrics, alerts, or tracing spans are specified. Existing resource status and conditions expose logical readiness, provider-operation progress, and disabled-mode skips; the exact status contract is defined in [API Extensions](#api-extensions) and [Provider Networking Control](#provider-networking-control).

## 8. Impact and Compatibility

### Current implementation alignment

The API and manager contracts in this proposal are the target. The current OSAC code has the following gaps that implementation must close; this section records them so the proposal is not mistaken for a description of already-delivered behavior:

- Current networking controllers can use `k8s_manager` when `fabric_manager` is absent for some shared network operations. The target requires a Fabric Manager in every NetworkClass and routes all shared networking-resource operations to it; the K8s Manager is assigned only the VM overlay work. The current fallback behavior must be removed (see [`virtualnetwork_controller.go`](https://github.com/osac-project/osac/blob/main/osac-operator/internal/controller/virtualnetwork_controller.go), [`securitygroup_controller.go`](https://github.com/osac-project/osac/blob/main/osac-operator/internal/controller/securitygroup_controller.go), [`externalip_controller.go`](https://github.com/osac-project/osac/blob/main/osac-operator/internal/controller/externalip_controller.go), and [`externalippool_controller.go`](https://github.com/osac-project/osac/blob/main/osac-operator/internal/controller/externalippool_controller.go)).
- Manager registrations currently select `fabric_manager` and `k8s_manager`, but the operator parser and Helm template do not yet validate `implementationRef` or `contractVersion`. The target contract validates requests against the fixed profile dispatch matrix in Sections 4.1 and 4.3; required operation sets and target types are defined by the Manager Operation Contract, not by registration fields. See [`networkmanager/types.go`](https://github.com/osac-project/osac/blob/main/osac-operator/pkg/networkmanager/types.go) and [`network-managers.yaml`](https://github.com/osac-project/osac/blob/main/osac-operator/charts/operator/templates/network-managers.yaml).
- Current networking service protos expose Update RPCs, and some resources allow metadata updates. This proposal's create/read/delete contract rejects resource specification and metadata updates. Current Cluster attachment validation also documents that an ExternalIPAttachment may be created before the Cluster is Ready; the target contract requires the target and selected endpoint to be Ready before the create request is accepted.
- Current `VirtualNetworkSpec` does not carry the proposed immutable `network_class` reference. The target schema adds it so the VirtualNetwork's selected provider profile is explicit. The current code must add and validate the reference and keep the role implementations hidden from tenant selection.
- Current manager capability handling accepts IPv6 and dual-stack declarations, while this design requires IPv4-only registration and output. The disabled ExternalIP path currently reports a placeholder address (`0.0.0.0`) as allocated and ready; the target requires an empty address and Pending state until a real provider allocation exists. See [`externalip_controller.go`](https://github.com/osac-project/osac/blob/main/osac-operator/internal/controller/externalip_controller.go).
- The current private ExternalIPPool schema retains an output-only `implementation_strategy` field. That legacy field is not part of the target API and must not select a backend; manager role registration is authoritative. Existing values or annotations must be ignored for dispatch and may be removed through the API compatibility work.
- Current `NetworkClass.spec.vip_prefix_length` validation permits values through 128. The IPv4-only target limits this prefix to 1 through 32 so the VIP range is valid for the supported address family.
- Cluster endpoint fields `api_endpoint=6` and `ingress_endpoint=7`, and the BaremetalInstance network status at field 4, already exist. ComputeInstance has no equivalent network-attachment status today; this proposal adds `compute_network_attachment_statuses=7`.
- The current networking UI has list/create/delete support for VirtualNetwork, Subnet, and SecurityGroup, reads ExternalIP/NATGateway data, and includes a SecurityGroup rule-editing placeholder. The target removes in-place rule changes and completes the remaining provider and ExternalIPAttachment workflows under OSAC-2226, as summarized in [UX Alignment](#ux-alignment).

The published API must converge on one readiness rule, one immutable-resource rule, and one manager operation contract before OSAC 0.2 release qualification.

### Upgrade and Downgrade Strategy

The provider gate defaults to enabled, preserving the current osac-operator
umbrella default. BMF deployments that previously inherited the standalone
chart's disabled default change behavior unless their existing profile or
upgrade values select and set the desired combined state. When the previous
operator settings differ, the shared setting necessarily changes one of them.
Changing the gate requires a
Helm/Enclave upgrade and rollout of both operators; it is read at startup.
Disabled behavior is established after both operators use the new
value and tracked network jobs are terminal. Older pods may still submit work
during a rolling upgrade, so the disabled contract must not be claimed before
rollout completes. API availability is independent of this rollout. Previously active provider
network operations must reach terminal state before disabled status or deletion
completion is reported. [User]

A downgrade to an operator/chart that does not support the gate restores its
older provider behavior. Provider resources left by disabled cleanup require
manual/provider-side reconciliation before re-enabling or downgrading; the
setting does not reverse previous provider changes. No schema migration is
introduced by this setting. [User]

### Version Skew Strategy

Both operator versions and their charts must support the same global startup
setting. Mixed operator versions/settings do not provide the disabled-mode
guarantee; complete the coordinated rollout before treating provider work as
disabled. The fulfillment API has no new field or registration dependency.
[User]

### Support Procedures

Inspect the global Helm value, both operators' startup environments, resource
conditions, and tracked AAP job states. `Ready=True` with reason
`ProvisioningDisabled` means only that a non-allocating OSAC object completed
its logical lifecycle. Bare-metal networking-phase conditions with `Status=Unknown` and
reason `ProvisioningDisabled` identify work that was skipped; neither form proves
provider connectivity, allocation, or cleanup. Cancellation errors require restoring AAP access and
waiting for terminal state. Delete provider leftovers or restore host port
placement using provider/manual procedures before assuming cleanup or baseline
connectivity. Change the setting through Helm/Enclave rollout. [User]

### Infrastructure Needed

No additional infrastructure beyond existing OSAC components and managers.

---

## Test Plan

### Unit and component tests (DEV)

- **FR-1 through FR-9, IC-1 through IC-3:** validate canonical IPv4 ranges,
  isolation boundaries, rule semantics, resource immutability, one attachment
  per workload, readiness gates, and dependency-guard error details.
- **FR-2, FR-3, FR-6, IC-4:** validate registration identity, role, contract
  version, and capability fields; verify that every NetworkClass requires a
  Fabric Manager, the fixed operation and target matrix, rejection before
  dispatch when a required role is absent, and no implicit manager fallback.
- **FR-4, FR-5:** validate ExternalIP allocation result and annotation against
  UID, generation, address family, and pool; ensure retry returns the same
  reservation; ensure release does not free API capacity before confirmed
  manager success.
- **FR-7, FR-9:** validate workload attachment schemas and resolved status for
  ComputeInstance, Cluster, and BaremetalInstance, including target-specific
  endpoint selection and the tenant/owner annotations.
- **FR-10, IC-5:** with the provider gate disabled, verify zero provider jobs,
  logical API lifecycle and dependency rules, in-flight job draining, correct
  `ProvisioningDisabled` state, no placeholder ExternalIP address, and no
  cleanup claim for provider resources that may remain.
- **FR-11, IC-6:** verify the UI client uses only create/read/delete for
  immutable networking resources, exposes only authorized Ready resources in
  workload pickers, and reports allocation and readiness accurately.

### Integration tests (DEV)

- Run the contract suite against at least two independent manager
  implementations for each role used by the deployment profiles. Verify that
  each implements every operation and target required for its role, and that
  tenant requests, operation payloads, results, retries, and resource status
  remain consistent across implementations.
- In a deployment with VM workloads, create and delete a VirtualNetwork and
  Subnet; verify Fabric Manager segment work and K8s Manager overlay work on
  each applicable hosting cluster. In a deployment without VM workloads,
  verify no K8s overlay operation is submitted. Verify that a NetworkClass
  without a Fabric Manager is rejected and no networking-resource operation
  falls back to the K8s Manager.
- Exercise SecurityGroup apply/delete, pool registration, ExternalIP
  allocation/release, and ExternalIPAttachment create/delete end to end. Verify
  tenant annotations and owner references on controller-created resources.
- Provision a VM, CaaS cluster, and bare-metal instance on the same supported
  Subnet, then verify discovered address status, ExternalIP target resolution,
  and SecurityGroup behavior.
- Verify CaaS endpoint status is populated before Cluster Ready and that
  ExternalIPAttachment creation is rejected until the target and endpoint
  prerequisites are Ready.
- Verify enabled and disabled paths across both operator areas, including
  API/UI availability and the ordinary workload-provisioning behavior defined
  for disabled mode.

### End-to-end release qualification (QE)

- In a Fabric-backed OSAC 0.2 deployment, use the unified UI and API to create
  a VirtualNetwork, Subnet, SecurityGroup, and ExternalIP; attach a VM, a CaaS
  cluster, and a bare-metal instance; confirm address visibility and permitted
  traffic; attach an ExternalIP to each supported target type; and delete
  resources in dependency order.
- Repeat the shared workload journey with two conforming manager
  implementations and verify tenant-visible APIs and behavior remain the same.
- Disable provider networking through the supported install/upgrade setting;
  verify resource APIs and ordinary VM, CaaS, and bare-metal provisioning remain
  available, provider operations are not submitted, and status and UI do not
  imply connectivity or cleanup. Re-enable through rollout and confirm normal
  reconciliation resumes for remaining resources.
- Verify lifecycle error journeys for a not-Ready reference, an unsupported
  manager target, an active dependent on delete, and an unavailable cluster
  endpoint; each must return an actionable error and leave no accepted
  forward-reference resource.
- Verify Cloud Infrastructure Admin provider views and the user documentation
  cover manager capability, pool capacity, resource lifecycle, ExternalIP
  reachability, and disabled-mode behavior.

## Graduation Criteria

The feature is eligible for the OSAC 0.2 release when all of the following are
true:

- The PRD acceptance criteria for FR-1 through FR-11 pass for the supported
  VM, CaaS, and BMaaS journeys, including create/read/delete and lifecycle
  errors.
- Every manager implements the complete operation and target set assigned to
  its role; conformance tests pass, and profile combinations with no dispatch
  route are rejected before provider work starts.
- Unit/component and integration coverage above passes in DEV qualification;
  the mixed-workload, manager-agnostic, and disabled-mode journeys pass QE
  release qualification.
- The unified UI work tracked by OSAC-2226 and user documentation deliver the
  API behaviors and support statements defined by the PRD and this design.
- All documented incompatibilities with current API/update behavior and
  manager registrations are resolved or explicitly accepted for the OSAC 0.2
  release; there are no unresolved schema placeholders or conflicting
  readiness rules.

## Support Boundaries

The provider networking control described in [Provider Networking Control](#provider-networking-control) gates provider operations while keeping the OSAC resource APIs available. These support limits define the deployment and workload conditions for this proposal.

### Deployment Support Boundary

The current OSAC networking contract supports connected deployments only.
Air-gapped and disconnected networking deployments are outside the supported
boundary and must not be advertised as supported profiles. A connected
deployment has reachability among the provider-owned hub, selected network
managers, provider-controlled networking services, and provider-controlled
address infrastructure. The provider owns this configuration; connectivity is
not tenant selectable, and these reachability prerequisites must hold before
the deployment's NetworkClass is accepted. Every supported NetworkClass
requires a Fabric Manager and may also assign a K8s Manager for VM overlay
operations.

### Networking Hub Support Boundary

OSAC networking supports exactly one provider-owned hub per deployment.
Multi-hub networking placement, cross-hub resource coordination, and
cross-hub network connectivity are unsupported. This boundary applies only to
the networking area and does not define hub behavior for other OSAC areas.
Multiple hosting/workload clusters remain supported where a networking feature
explicitly specifies them.

---

## Provenance

Authored: revise @ design 0.11.3 - 2bd6607, workspace main @ 1f3b63b82 (58 behind origin/main)
Final: revise @ design 0.11.3 - 2bd6607, workspace main @ 1f3b63b82 (81 behind origin/main, dirty)

> Context changed between revise and revise.

> This document's phase history does not include an initial /draft — structure was not verified against the template from origin.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.3","ai_workflows":"2bd6607","source_repo":"1f3b63b82 (dirty)","source_repo_branch":"main","commits_behind_main":81,"commits_ahead_main":0,"main_ref":"main","phases":["revise","manual-edit","revise","revise","revise","revise","revise","revise"],"authoring_modes":["manual","skill"],"context_changed":true,"origin_untracked":true} -->
