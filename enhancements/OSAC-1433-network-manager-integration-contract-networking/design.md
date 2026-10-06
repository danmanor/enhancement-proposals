---
title: network-manager-integration-contract
authors:
  - dmanor@redhat.com
creation-date: 2026-10-04
last-updated: 2026-10-06
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-1433
prd: prd.md
see-also:
  - Unified Networking PRD: /enhancements/OSAC-1433-unified-networking/prd.md
  - Unified Networking Design: /enhancements/OSAC-1433-unified-networking/design.md
  - Agentless VLAN Fabric Manager PRD: /enhancements/OSAC-3664-agentless-vlan-fabric-manager/prd.md
  - CUDN EVPN Kubernetes Manager: /enhancements/OSAC-4291-cudn-evpn-k8s-manager-phase-1-networking/design.md
replaces:
  - N/A
superseded-by:
  - N/A
---

# OSAC Network Manager Integration Contract

| Field | Value |
|-------|-------|
| Author(s) | Dan Manor (dmanor@redhat.com) |
| Jira | https://redhat.atlassian.net/browse/OSAC-1433 |
| PRD | [Network Manager Integration Contract PRD](prd.md) |
| Date | 2026-10-06 |

## Contents

- [1. Overview](#1-overview)
- [2. Goals and Non-Goals](#2-goals-and-non-goals)
- [3. Motivation / Background](#3-motivation--background)
- [4. Design](#4-design)
  - [4.1 Architecture and Manager Roles](#41-architecture-and-manager-roles)
  - [4.2 Registration and Compatibility Data](#42-registration-and-compatibility-data)
  - [4.3 API and Operation Contract](#43-api-and-operation-contract)
    - [Subnet handoff data](#subnet-handoff-data)
    - [Common AAP input](#common-aap-input)
    - [Workload attachment policy input](#workload-attachment-policy-input)
    - [Fixed operation and target table](#fixed-operation-and-target-table)
    - [Operation result envelope](#operation-result-envelope)
  - [4.4 Subnet Orchestration and Handoff Lifecycle](#44-subnet-orchestration-and-handoff-lifecycle)
  - [4.5 Scalability and Performance](#45-scalability-and-performance)
  - [4.6 Security Considerations](#46-security-considerations)
  - [4.7 Failure Handling and Recovery](#47-failure-handling-and-recovery)
  - [4.8 RBAC and Tenancy](#48-rbac-and-tenancy)
  - [4.9 Extensibility and Future-Proofing](#49-extensibility-and-future-proofing)
  - [4.10 Risks and Mitigations](#410-risks-and-mitigations)
  - [4.11 Drawbacks](#411-drawbacks)
- [5. Interface Changes](#5-interface-changes)
- [6. Alternatives Considered](#6-alternatives-considered)
- [7. Observability and Monitoring](#7-observability-and-monitoring)
- [8. Impact and Compatibility](#8-impact-and-compatibility)
- [Test Plan](testplan.md)

## 1. Overview

A **Fabric Manager** is the provider-selected implementation that configures the physical or virtual network fabric for OSAC. A **Kubernetes Manager** is the provider-selected implementation that connects Kubernetes-hosted workloads to that fabric. A **NetworkClass** is the provider-managed OSAC resource that selects one manager for each configured role. A **manager registration** is the role-labelled Kubernetes ConfigMap through which OSAC discovers an implementation. Every networking resource is infrastructure-agnostic: it keeps the same meaning for virtual machine (VM), managed-cluster, and bare-metal workloads. Every manager role is backend-agnostic: any implementation that fulfills this contract can provide its assigned behavior. The [Unified Networking Design](../OSAC-1433-unified-networking/design.md) defines the shared resource semantics.

This design defines the source-neutral registration, compatibility checks, Ansible Automation Platform (AAP) task interface, and cross-manager Subnet handoff. For a Subnet with a Kubernetes Manager, the Fabric Manager writes a fixed output ConfigMap in the Subnet namespace; OSAC validates its values and passes them to the Kubernetes Manager. OSAC coordinates the two manager roles; one manager does not call another directly. Once OSAC implements this generic contract, adding a conforming implementation requires its registration and Ansible collection role, with no implementation-specific Go code. See the [PRD](prd.md) for goals and user outcomes.

## 2. Goals and Non-Goals

### 2.1 Goals

- Specify the complete implementation contract for each manager role, including registration, compatibility, operations, task inputs, results, and retries.
- Let a provider select only manager combinations that mutually declare compatibility.
- Pass the Fabric Manager's Subnet networking output to a compatible Kubernetes Manager through a stable OSAC-owned interface.
- Allow a conforming implementation from any source to use the same tenant networking application programming interface (API) and OSAC dispatch behavior.
- Keep future manager onboarding within registration and Ansible content after generic OSAC contract support is implemented.

### 2.2 Non-Goals

- Change the meaning, fields, or lifecycle of tenant networking resources defined by Unified Networking.
- Require every Fabric Manager to work with every Kubernetes Manager.
- Define product-specific configuration for any backend.
- Let an implementation add resource kinds, operation identifiers, workload targets, or tenant API fields that OSAC does not understand.
- Migrate existing backend resources when a provider changes the manager selected by a NetworkClass.

## 3. Motivation / Background

The current operator discovers managers from Kubernetes ConfigMaps and dispatches provider work to Ansible Automation Platform (AAP). A manager registration currently provides a logical name, description, and capabilities, but not a collection role reference or peer compatibility declaration. Subnet creation already runs Fabric before Kubernetes and passes Fabric-assigned network-segment identifiers and reserved-address values to the Kubernetes task. The manager resolver does not validate whether a selected Fabric and Kubernetes pair is compatible, AAP dispatch is not yet based on the proposed explicit `implementationRef`, and Subnet teardown currently runs the targets independently. [Codebase: osac-operator/pkg/networkmanager/types.go; osac-operator/pkg/dispatcher/dispatch.go; osac-operator/internal/controller/subnet_controller.go; osac-operator/internal/controller/fabric_output_provider.go; osac-operator/pkg/provisioning/vni_outputs.go]

A provider must be able to declare which manager pairs can consume the same integration data, and reject a pair whose network models cannot interoperate. OSAC needs a declared, checked compatibility boundary and an OSAC-owned handoff protocol, while the managers remain separate implementations. [User]

## 4. Design

### 4.1 Architecture and Manager Roles

A Fabric Manager implements the shared networking resource operations and physical workload attachment operations assigned to that role. It configures the provider's network fabric for VirtualNetworks, Subnets, external address pools and attachments, and network address translation (NAT) gateways. It enforces SecurityGroup policy on physical interfaces, attaches bare-metal and CaaS worker interfaces to Subnets, and handles the supported lease lookup targets.

A Kubernetes Manager implements Kubernetes-side Subnet networking and SecurityGroup enforcement for ComputeInstance VM overlays. It does not replace the Fabric Manager and does not implement the shared networking-resource operations. A NetworkClass always selects one Fabric Manager. It may select one Kubernetes Manager when the deployment supports workloads that need Kubernetes-side networking.

The OSAC operator resolves the registrations selected by NetworkClass, validates their required fields and pair compatibility, dispatches each operation to its assigned role through Ansible Automation Platform (AAP), and records readiness and job state. Managers remain separate implementations: one manager does not call another or receive its credentials. OSAC owns the integration boundary and orchestration order.

Every networking resource is infrastructure-agnostic: it retains the same meaning for virtual machines, managed clusters, and bare-metal servers. Every manager role is backend-agnostic: any implementation that fulfills its assigned contract can provide that behavior. The [Unified Networking Design](../OSAC-1433-unified-networking/design.md) defines tenant resource semantics; this design defines the provider integration contract.

### 4.2 Registration and Compatibility Data

A **manager registration** is a role-labelled Kubernetes ConfigMap in the operator namespace. The role label identifies a Fabric Manager or Kubernetes Manager. Its `data.name` is the unique logical name selected by NetworkClass. `implementationRef` names the Ansible collection role AAP invokes; the logical name and implementation reference are independent.

The manager registration requires these fields:

| Field | Required | Meaning |
|-------|----------|---------|
| Role label | Yes | Exactly one of `osac.openshift.io/network-fabric-manager: "true"` or `osac.openshift.io/network-k8s-manager: "true"`. |
| `name` | Yes | Unique logical name within the role, selected by NetworkClass. |
| `implementationRef` | Yes | Fully qualified Ansible collection role name, such as `acme.networking.fabric_manager`; it must be installed in the AAP execution environment. |
| `capabilities` | Yes | Comma-separated values from the OSAC-defined vocabulary. This proposal accepts exactly `ipv4` (Internet Protocol version 4); missing `ipv4`, `ipv6`, `dualStack`, and unknown values are rejected. |
| `compatibleManagers` | Yes | Comma-separated logical names in the opposite role that can interoperate with this registration. An empty value means no compatible peer. |
| `description` | No | Human-readable description for provider administration. |

Compatibility is pair-specific and reciprocal. A Fabric Manager lists compatible Kubernetes Manager names; a Kubernetes Manager lists compatible Fabric Manager names. OSAC accepts a selected pair only when both list each other and both implementations satisfy the shared manager contract. A manager publisher declares compatibility only when its implementation can consume and produce the shared handoff with that peer. No vendor pair is embedded in OSAC code.

Compatibility is based on the data exchanged between a pair, not the manager names. In the example below, Netris publishes a Subnet's Layer 2 segment identifier, its VirtualNetwork's Layer 3 identifier, and the reserved-address range; the [CUDN EVPN Kubernetes Manager](/enhancements/OSAC-4291-cudn-evpn-k8s-manager-phase-1-networking/design.md) consumes those values. They identify a Virtual eXtensible Local Area Network (VXLAN) network and are called VXLAN Network Identifiers (VNIs). The [Agentless VLAN Fabric Manager](/enhancements/OSAC-3664-agentless-vlan-fabric-manager/design.md) uses Virtual Local Area Network (VLAN) identifiers, which this handoff does not carry. In this example, that pair cannot claim compatibility because CUDN EVPN does not consume or translate the Agentless VLAN segment identifier. A pair may declare compatibility only if both implementations satisfy the published handoff and workload attachment contracts.

The following declarations permit Netris with CUDN EVPN and reject Agentless VLAN with CUDN EVPN:

| Registration | `compatibleManagers` |
|--------------|-----------------------|
| Fabric `netris` | `cudn_evpn` |
| Kubernetes `cudn_evpn` | `netris` |
| Fabric `agentless_net` | A list that does not include `cudn_evpn` |

A registration has this shape:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: osac-network-fabric-manager-netris
  namespace: osac
  labels:
    osac.openshift.io/network-fabric-manager: "true"
data:
  name: netris
  implementationRef: acme.networking.fabric_manager
  capabilities: "ipv4"
  compatibleManagers: "cudn_evpn"
  description: "Fabric integration"
```

#### NetworkClass capability behavior

Manager registrations declare capabilities from a fixed OSAC vocabulary; they cannot define custom capabilities. This proposal supports Internet Protocol version 4 (IPv4) only; Internet Protocol version 6 (IPv6) and dual-stack networking (IPv4 and IPv6 together) are unsupported. Every selected manager, including the Fabric Manager and any selected Kubernetes Manager, must declare `ipv4`. A missing declaration or unsupported value makes NetworkClass invalid and prevents provider dispatch. OSAC derives the read-only NetworkClass IP-family output from those declarations. The resulting API fields and behavior are:

| Registration capability | NetworkClass output | Behavior |
|--------------------------|---------------------|-----------------------|
| `ipv4` | `supportsIpv4` | Required from every selected role; every valid NetworkClass reports true and the networking API accepts IPv4 addresses and prefixes. |
| `ipv6` | `supportsIpv6` | Rejected in registrations; output is false and IPv6 inputs are rejected by the networking API. |
| `dualStack` | `supportsDualStack` | Rejected in registrations; output is false and dual-stack inputs are rejected by the networking API. |

These manager-derived fields are distinct from `NetworkClass.spec.east_west_capabilities`, which the provider declares to advertise supported FabricDomain types. East-west capability declaration and validation are defined by the Unified Networking and [Multi-Fabric East-West Networking designs](/enhancements/OSAC-1382-multi-fabric-east-west-networking/design.md); they are not part of manager registration or the manager intersection. Adding a manager capability that changes request behavior requires a platform contract update.

### 4.3 API and Operation Contract

This contract does not change the tenant-facing networking API. It defines the manager registration, common AAP input, required operations and targets, results, and the stable Fabric-to-Kubernetes Subnet handoff. List and Get operations remain served by OSAC; the operations below cover provider backend reconciliation.

#### Subnet handoff data

When a Kubernetes Manager is selected, the Fabric Manager's `subnet.create` task writes the output ConfigMap `subnet-<subnet-name>-fabric-output` in the Subnet namespace. The handoff requires exactly these shared data keys; their ConfigMap values are strings:

The `l2_vni` and `l3_vni` values are 24-bit VXLAN Network Identifiers for the Subnet's Layer 2 network and its parent VirtualNetwork's Layer 3 routing domain. The Fabric Manager assigns them; the Kubernetes Manager consumes the assigned values rather than allocating replacements. The reserved-address value uses Classless Inter-Domain Routing (CIDR) notation where it describes an address range.

| Key | Required value | Consumer behavior |
|-----|----------------|-------------------|
| `l2_vni` | Integer VNI from 1 through 16,777,215. It identifies the Subnet's Layer 2 segment. | The Kubernetes Manager uses the assigned VNI for the Subnet's Layer 2 network; it does not allocate or substitute another VNI. |
| `l3_vni` | Integer VNI from 1 through 16,777,215. It identifies the parent VirtualNetwork's Layer 3 routing domain. | The Kubernetes Manager uses the assigned VNI for the VirtualNetwork's Layer 3 network; it does not allocate or substitute another VNI. |
| `fabric_reserved_range` | A non-empty string representing IPv4 addresses or ranges reserved by the fabric within the Subnet, such as gateway, switch virtual interface, or Dynamic Host Configuration Protocol (DHCP) addresses. | The Kubernetes Manager excludes those addresses from its own workload IP allocation. It validates/translates the value for its native API. |

OSAC reads the ConfigMap only after Fabric provisioning succeeds. It requires all three values, validates both VNI values as integers in the stated range, normalizes them to integers, and passes these values unchanged in meaning to the Kubernetes AAP launch. OSAC checks that `fabric_reserved_range` is non-empty and passes the string through; it does not parse its internal range syntax. A manager pair can declare compatibility only when the Kubernetes implementation understands the shared reserved-range encoding and applies the exclusion correctly.

The handoff is an internal, namespaced ConfigMap and AAP input, not Subnet status or tenant input. The defined handoff includes no VLAN identifier. A pair that needs a VLAN identifier or another shared value cannot claim compatibility; adding that value requires a platform contract update.

#### Common AAP input

OSAC passes a common job envelope in the `osac_job_vars` variable. The task-specific resource is the OSAC resource being reconciled. The manager reference selects the collection role, while the operation names one fixed task from the operation table. `manager.role` is normalized by OSAC to exactly `fabric` or `kubernetes`; a collection cannot choose or override it.

```yaml
osac_job_vars:
  operation: subnet.create
  manager:
    name: netris
    role: fabric
    implementationRef: acme.networking.fabric_manager
  resource:
    apiVersion: osac.openshift.io/v1alpha1
    kind: Subnet
    metadata: {}
    spec: {}
```

For the Kubernetes `subnet.create` and `subnet.delete` tasks, OSAC also passes the three handoff values as top-level AAP extra variables alongside `osac_job_vars`:

```yaml
l2_vni: 40120
l3_vni: 4020
fabric_reserved_range: "192.0.2.1/32,192.0.2.16/28"
```

The Kubernetes task must consume all three values for those operations. It must not read Fabric credentials, call the Fabric Manager directly, or change the OSAC resource semantics. A successful task means that the backend has converged to the requested state unless the operation defines a result artifact below.

#### Workload attachment policy input

SecurityGroup is an immutable OSAC rule resource, not a separately provisioned
backend object. When a workload selects one or more groups, OSAC resolves the
attachment's ready Subnet and groups and sends the complete normalized policy
with `workload_attachment.apply`. A group contains ingress and egress rules;
each rule carries `protocol`, optional TCP/UDP `port_from` and `port_to`, and
an IPv4 `ipv4_cidr`. The payload preserves each group's UID and the complete
rule arrays so managers can remove only policy owned by that workload.
Protocols are `TCP`, `UDP`, `ICMP`, and `ALL`. TCP/UDP require a port range
from 1 through 65535 with `port_to >= port_from`; ICMP and ALL omit port
fields. Ingress CIDRs match sources and egress CIDRs match destinations. The
legacy IPv6 field is omitted because the API rejects non-empty IPv6 values.

The `workload_attachment` object is a sibling of `resource` in
`osac_job_vars`. `resource` remains the owning ComputeInstance, Cluster, or
BaremetalInstance, so the standard result UID and generation identify the
workload. Its normalized fields are:

| Field | Meaning |
|-------|---------|
| `subnet` | Resolved Subnet UID, name, IPv4 CIDR, and its parent VirtualNetwork UID and name. |
| `interfaces` | One or more resolved interface descriptors with `type` and `name`. ComputeInstance uses `{type: overlay, name: primary}`; BaremetalInstance uses the selected physical port; Cluster provides each node-set name and its resolved `fabric_interface` as a physical interface. |
| `security_groups` | Entries containing group UID, name, and complete immutable `ingress` and `egress` rules. Rule order does not affect behavior; attached groups' allow rules combine as a union. |

OSAC validates that the Subnet and all selected SecurityGroups are Ready and
belong to the same VirtualNetwork before dispatch. `workload_attachment.apply`
must establish the Subnet connection and install the effective stateful rules
before workload traffic is enabled. Unmatched new traffic is denied and
established reply traffic is allowed. `workload_attachment.delete` removes the
workload-owned policy and detaches the interface (restoring the provisioning
network for physical ports). Both operations are idempotent by workload UID
and interface. Policy failure keeps the workload non-ready and traffic
unavailable; delete failure retains the workload finalizer and retries before
teardown. The task receives the same normalized attachment on apply and delete.

For ComputeInstance, OSAC routes these operations to the Kubernetes Manager.
For BaremetalInstance and Cluster workers, OSAC routes them to the Fabric
Manager. Managers do not call one another; the cluster's resolved physical
interfaces and policy are part of the Fabric Manager input.

```yaml
osac_job_vars:
  operation: workload_attachment.apply
  manager:
    name: cudn_evpn
    role: kubernetes
    implementationRef: acme.networking.vm_overlay
  resource:
    apiVersion: osac.openshift.io/v1alpha1
    kind: ComputeInstance
    metadata: {uid: "<workload UID>", generation: 3}
    spec: {}
  workload_attachment:
    subnet:
      uid: "<subnet UID>"
      name: app-subnet
      ipv4_cidr: 192.0.2.0/24
      virtual_network: {uid: "<VirtualNetwork UID>", name: app-network}
    interfaces:
      - {type: overlay, name: primary}
    security_groups:
      - uid: "<group UID>"
        name: web
        ingress:
          - {protocol: TCP, port_from: 443, port_to: 443, ipv4_cidr: 0.0.0.0/0}
        egress: []
```

#### Fixed operation and target table

| Operation | Assigned role and order | Workload target | Fixed task entry point | Required behavior and result |
|-----------|-------------------------|-----------------|-------------------------|------------------------------|
| `virtual_network.create` / `virtual_network.delete` | Fabric | N/A | `create_virtual_network` / `delete_virtual_network` | Create or remove the isolated VirtualNetwork routing domain and associated allocation. Subnets in the same VirtualNetwork are L3-routable to one another; separate VirtualNetworks remain isolated. |
| `subnet.create` / `subnet.delete` | Create: Fabric, then Kubernetes when configured. Delete: Kubernetes, then Fabric. | N/A | `create_subnet` / `delete_subnet` | Fabric creates/removes one L2 broadcast domain for the Subnet. When a Kubernetes role is selected, Fabric writes the output ConfigMap on create and removes it during its delete task. Workloads in the same Subnet share that L2 domain and are L3-routable within their parent VirtualNetwork. Kubernetes creates/removes its network side using all three exact handoff values. The Subnet CIDR belongs to its parent VirtualNetwork and cannot overlap a sibling Subnet. |
| `workload_attachment.apply` / `workload_attachment.delete` | Kubernetes Manager for `compute_instance`; Fabric Manager for `cluster` and `baremetal_instance` | `compute_instance`, `cluster`, `baremetal_instance` | `apply_workload_attachment` / `delete_workload_attachment` | Connect the resolved interface to its Subnet and enforce the complete effective stateful SecurityGroup rules before enabling traffic. On delete, remove policy and detach the interface before workload teardown. For CaaS, Fabric receives each node-set interface. |
| `external_ip_pool.create` / `external_ip_pool.delete` | Fabric | N/A | `create_external_ip_pool` / `delete_external_ip_pool` | Register or remove the provider address pool. The pool has one canonical IPv4 Classless Inter-Domain Routing (CIDR) prefix; OSAC owns API capacity counters. |
| `external_ip.allocate` / `external_ip.release` | Fabric | N/A | `create_external_ip` / `delete_external_ip` | Reserve or release one address from the selected pool. After durable allocation, the task writes the address to `osac.openshift.io/allocated-address`. Repeated allocation for the same resource unique identifier returns the same address. |
| `external_ip_attachment.create` / `external_ip_attachment.delete` | Fabric | `compute_instance`, `cluster`, `baremetal_instance` | `attach_external_ip` / `detach_external_ip` | Add or remove inbound translation for the resource target or configured Cluster endpoint. Remove the attachment before releasing its ExternalIP. |
| `nat_gateway.create` / `nat_gateway.delete` | Fabric | N/A | `create_nat_gateway` / `delete_nat_gateway` | Add or remove outbound source network address translation (SNAT) for the VirtualNetwork using its ExternalIP. It does not provide inbound access. |
| `dhcp_lease.query` | Fabric | `cluster`, `baremetal_instance` | `query_dhcp_lease` | Return the Dynamic Host Configuration Protocol (DHCP) lease matching each requested network attachment as the AAP artifact `leases`. |

Every implementation must provide the complete operation and workload-target combinations assigned to its role. A registration does not select an operation subset. A manager with a different internal model must translate the fixed OSAC operation into that model; it cannot route the operation to the other manager role.

The `leases` artifact contains entries with `subnet_ref`, `interface`, `ip_address`, and `mac_address`. A missing or ambiguous lease match is a task failure. OSAC owns API resource phase, conditions, and provisioning job history.

#### Operation result envelope

Every successful operation except `dhcp_lease.query` returns an AAP artifact named `osac_result` with this fixed shape. The `resourceUID` field identifies the Kubernetes object's unique identifier (UID); `observedGeneration` identifies the version of its specification that the manager processed.

```yaml
osac_result:
  operation: subnet.create
  resourceUID: "<resource UID>"
  observedGeneration: 3
  data: {}
```

`operation`, `resourceUID`, and `observedGeneration` must match the operation OSAC dispatched and the resource version it dispatched. `data` is an empty object; operation-specific outputs use their separately defined channels. ExternalIP allocation reports its address through the guarded `osac.openshift.io/allocated-address` annotation, not in `data`. Subnet Fabric outputs use the separate namespaced ConfigMap defined above, not this envelope. `dhcp_lease.query` returns the `leases` artifact instead of `osac_result`. The envelope has no version field; changes to its fields or result data require a coordinated OSAC and manager contract update.

OSAC validates the artifact name and all required fields before accepting task success. A missing, malformed, stale, or mismatched envelope is a failed operation and cannot advance resource readiness or release capacity. A manager must not report a successful operation with an envelope for a different operation, resource UID, or generation.

Ansible supports role inclusion by a variable role name and the `tasks_from` selector. The fully qualified collection role must be installed in the AAP execution environment. See [Ansible Core include_role documentation](https://docs.ansible.com/projects/ansible-core/2.17/collections/ansible/builtin/include_role_module.html) and [using collection roles by FQCN](https://docs.ansible.com/projects/ansible/latest/collections_guide/collections_using_playbooks.html). [Research: §1]

### 4.4 Subnet Orchestration and Handoff Lifecycle

The operation table and handoff values above define what each manager receives. This sequence shows when OSAC invokes each role and how the shared ConfigMap remains available through cleanup:

```mermaid
sequenceDiagram
    participant NC as NetworkClass
    participant Operator as OSAC operator
    participant AAP as AAP
    participant Fabric as Fabric Manager
    participant K8s as Kubernetes Manager

    NC->>Operator: Select mutually compatible registrations
    Operator->>Operator: Validate roles, capabilities, contract, and compatibility
    Operator->>AAP: subnet.create for Fabric
    AAP->>Fabric: create_subnet(resource)
    Fabric->>Fabric: Write namespaced fabric-output ConfigMap
    AAP-->>Operator: Successful Fabric job
    Operator->>Operator: Read ConfigMap; validate and normalize required values
    Operator->>AAP: subnet.create for Kubernetes with three top-level extra vars
    AAP->>K8s: create_subnet(resource, l2_vni, l3_vni, fabric_reserved_range)
    K8s-->>AAP: Converged result
    AAP-->>Operator: Successful Kubernetes job
    Operator->>Operator: Mark Subnet Ready after both roles succeed

    Operator->>Operator: Preserve output ConfigMap through K8s cleanup
    Operator->>AAP: subnet.delete for Kubernetes with the same three values
    AAP->>K8s: delete_subnet(resource, l2_vni, l3_vni, fabric_reserved_range)
    K8s-->>AAP: Detached result
    AAP-->>Operator: Successful Kubernetes job
    Operator->>AAP: subnet.delete for Fabric
    AAP->>Fabric: delete_subnet(resource)
    Fabric->>Fabric: Remove fabric resources and output ConfigMap
    Fabric-->>AAP: Removed result
    AAP-->>Operator: Successful Fabric job
    Operator->>Operator: Complete deletion after both roles succeed
```

#### Workload attachment policy lifecycle

The workload attachment operation carries the complete normalized attachment
to the manager that owns its interface. That manager connects the interface
only after it has installed the selected stateful SecurityGroup rules. During
deletion, OSAC waits for policy removal and interface detachment before the
workload controller removes the VM, server, or cluster attachment.

```mermaid
sequenceDiagram
    participant Workload as Workload provisioning controller
    participant OSAC as OSAC dispatcher
    participant Manager as Interface owner manager
    participant Target as Workload interface

    Workload->>OSAC: Ready Subnet and resolved SecurityGroups
    OSAC->>Manager: workload_attachment.apply(resource, normalized attachment)
    Manager->>Target: Install deny-by-default stateful policy
    Manager->>Target: Connect interface to Subnet
    Manager-->>OSAC: Successful osac_result
    OSAC-->>Workload: Attachment complete; workload may become Ready
    Workload->>OSAC: Delete workload
    OSAC->>Manager: workload_attachment.delete(same attachment)
    Manager->>Target: Remove policy and detach or restore provisioning network
    Manager-->>OSAC: Successful osac_result
    OSAC-->>Workload: Attachment cleanup complete; continue teardown
```

OSAC routes ComputeInstance attachments to the Kubernetes Manager and routes
BaremetalInstance and CaaS worker attachments to the Fabric Manager. A failed
apply leaves the workload non-ready with no workload traffic; a failed delete
retains its finalizer and blocks teardown until retry succeeds.

The current `origin/main` create lifecycle already waits for Fabric success, reads this ConfigMap, validates/normalizes the values, and passes the required values to the Kubernetes AAP target. The target contract makes this exact three-key exchange normative. Current Subnet deletion runs Fabric and Kubernetes targets independently; generic OSAC lifecycle support must add the reverse dependency so Kubernetes cleanup succeeds before Fabric cleanup removes the ConfigMap. Each task remains independently implemented and idempotent; OSAC owns ordering and retries.

### 4.5 Scalability and Performance

Manager registration and compatibility lists are small and are read during manager discovery or NetworkClass validation. The handoff is bounded to three small ConfigMap values per Subnet and requires no new service or backend database. Subnet provisioning adds an ordered AAP stage when a Kubernetes Manager is configured, so readiness waits for both jobs to finish.

### 4.6 Security Considerations

Manager ConfigMaps contain references and compatibility metadata, not credentials. Existing Kubernetes role-based access control (RBAC) in the operator namespace protects registrations. AAP credentials or provider-managed Secrets supply backend access; Ansible job artifacts and logs must not disclose secrets. The handoff contains only the two VNI identifiers and the fabric-reserved address string. Its ConfigMap is scoped to the Subnet namespace and OSAC reads the exact name derived from that Subnet. Manager tasks act only on the tenant-scoped resources OSAC passes and must preserve existing tenant and owner-reference boundaries.

### 4.7 Failure Handling and Recovery

- **Missing or invalid registration:** NetworkClass validation reports the role, ConfigMap, and invalid field. OSAC does not dispatch resource work.
- **Incompatible manager pair:** NetworkClass validation fails with both selected logical names and the missing reciprocal declaration. OSAC starts no AAP job for resources using that NetworkClass.
- **Missing collection role or task:** AAP fails with the missing fully qualified collection role or task name. The diagnostic names the registration or task.
- **Missing or malformed Fabric outputs:** A missing output ConfigMap, missing required key, empty reserved-range value, or invalid VNI prevents the Kubernetes stage from starting. OSAC reports the missing or invalid output and retries/fails through the existing provisioning lifecycle.
- **Kubernetes create failure after Fabric success:** OSAC retains the Fabric output ConfigMap and retries the Kubernetes stage using the same values. Fabric create is idempotent. The Subnet is not Ready until the Kubernetes stage succeeds.
- **Kubernetes delete failure:** OSAC does not start Fabric deletion, so the output ConfigMap and segment remain available while Kubernetes detachment retries.
- **Fabric delete failure after Kubernetes detach:** OSAC retains job state and retries Fabric cleanup. The Kubernetes target is already detached; repeated Fabric deletion of absent state succeeds.
- **Backend timeout or transient error:** The AAP task returns a diagnostic and a non-success result. Existing OSAC job retry/backoff behavior retries the operation. All create/apply and delete tasks are idempotent by resource UID.
- **Workload attachment policy apply failure:** The manager keeps traffic unavailable; OSAC does not report the workload Ready and retries the same normalized attachment. **Policy delete failure:** OSAC retains the workload finalizer and does not detach the Subnet or continue workload teardown until cleanup succeeds.
- **Invalid operation result:** A missing, malformed, stale, or mismatched `osac_result` fails the operation; OSAC does not advance readiness or release capacity.
- **Invalid ExternalIP or lease result:** Missing allocated-address output or malformed/ambiguous `leases` output fails the task; OSAC does not report allocation or lease discovery as successful.

### 4.8 RBAC and Tenancy

No tenant-facing RBAC changes are required. The fulfillment service continues to authorize tenant networking requests. Manager registrations and backend credentials remain provider-scoped. OSAC passes only the resource context authorized for the requested tenant; managers must not read or modify resources belonging to another tenant.

### 4.9 Extensibility and Future-Proofing

A new manager is onboarded by installing its collection into the AAP execution environment, registering the role with its capabilities and compatible peer names, and selecting it in NetworkClass. No manager-specific Go code or tenant API change is required after OSAC implements the manager contract. A new operation, workload target, capability with new behavior, segment encapsulation, handoff field, or incompatible task payload changes the OSAC contract and requires platform support before an implementation can use it.

### 4.10 Risks and Mitigations

- **A declared manager pair may not preserve the shared network and policy semantics.** Require pair-level conformance for the Subnet handoff and workload attachment policy before a provider selects the pair.
- **Policy installation failure could expose a workload without its requested rules.** Require deny-by-default behavior until policy is installed, keep the workload non-ready on failure, and retain the deletion finalizer until policy cleanup and detachment succeed.
- **A manager may receive rules for a different tenant or attachment.** OSAC validates tenant ownership and the Subnet/SecurityGroup relationship before dispatch; the task is scoped to the workload UID and interface and receives no cross-tenant lookup authority.
- **The normalized operation may grow as OSAC adds rule types or attachment targets.** Treat new rule fields and targets as contract changes and require both OSAC and manager conformance updates before dispatching them.

### 4.11 Drawbacks

Each manager must implement OSAC's complete role-specific operation set and
match its SecurityGroup semantics, which increases provider integration and
conformance work. Workload readiness and teardown also depend on manager
availability while an attachment is applied or removed. The fixed normalized
rule model does not expose backend-specific ACL features to tenants. These
costs keep the tenant API consistent and allow providers to replace manager
implementations without changing workload attachment behavior.

## 5. Interface Changes

### IC-1: Manager registration and compatibility

**Requirements:** FR-1, FR-2, FR-3

Manager ConfigMaps require `implementationRef` and `compatibleManagers`; when both roles are selected, the compatibility declarations must be reciprocal. §4.2 specifies the registration fields and validation.

### IC-2: Generic AAP task invocation

**Requirements:** FR-1, FR-2, FR-3

AAP playbooks resolve the role from the registered fully qualified `implementationRef` and pass the fixed operation envelope in §4.3. Each manager collection provides the complete task set for its role.

### IC-3: Manager-pair validation and fixed role dispatch

**Requirements:** FR-2, FR-3, FR-4

NetworkClass selection requires a Fabric Manager and checks mutual compatibility whenever it selects a Kubernetes Manager. OSAC marks an invalid NetworkClass failed with a diagnostic, rejects it before AAP, and routes each operation only to its contract-assigned role.

### IC-4: Operation results and retry behavior

**Requirements:** FR-2

Manager tasks follow the operation, target, artifact, idempotency, and error rules in §4.3 and §4.7, including ExternalIP allocation and DHCP lease results.

### IC-5: Fabric-to-Kubernetes Subnet output handoff

**Requirements:** FR-1, FR-2, FR-3, FR-4

The Fabric Manager writes the three contract-defined output keys, OSAC validates them, and the Kubernetes Manager consumes the same values on Subnet create and delete. §4.3 defines the ConfigMap schema and §4.4 defines the required ordering.

### IC-6: Workload attachment and SecurityGroup enforcement

**Requirements:** FR-2, FR-3

OSAC sends the resolved workload attachment and complete SecurityGroup rules to the manager that owns its interface. The Kubernetes Manager handles ComputeInstance overlays; the Fabric Manager handles BaremetalInstance and CaaS worker interfaces. Apply and delete ordering, payload fields, stateful rule semantics, and failure behavior are specified in §4.3 and §4.4.

## 6. Alternatives Considered

### Keep compatibility as a hard-coded manager-pair table in Go

A central pair table gives OSAC direct control over every combination. It also requires a Go change whenever a provider adds a manager or a new compatible pair, which prevents configuration-only onboarding. Registration declarations let the generic validator check new pairs without vendor-specific branches.

### Let managers call each other directly

Direct calls can pass segment data without OSAC persistence. They couple managers to one another's APIs and credentials, bypass OSAC job tracking, and make retries and partial failure depend on vendor-specific coordination. OSAC-owned sequencing preserves separate manager implementations and one audited handoff.

### Keep the Subnet manager jobs independent

Independent jobs allow either manager to reconcile without waiting for the other. They cannot guarantee that the Kubernetes Manager consumes the values published by Fabric, and they can report partial readiness. The ordered create and delete stages provide the required dependency and retain the output ConfigMap for retries and cleanup.

### Derive the implementation from the logical manager name

Name-derived collection roles match the current built-in role convention. An independently maintained implementation would need to extend or replace the OSAC collection. An explicit fully qualified role reference keeps the selection source-neutral while operation names and payloads remain OSAC-defined.

### Pass an opaque vendor-specific artifact

An opaque vendor map would let each Fabric Manager return its own schema, but the Kubernetes Manager would need vendor-specific parsing. The three-key output contract has a stable common shape and leaves backend-only identifiers inside the Fabric Manager.

## 7. Observability and Monitoring

No new metrics are required. Existing NetworkClass state/message, resource conditions, events, AAP job history, and reconciliation logs report registration, compatibility, operation, and backend errors. Diagnostics identify the manager role and logical name, operation stage, and invalid handoff field. Logs do not include credential values.

## 8. Impact and Compatibility

This is the target contract; the current operator and AAP implementation do not yet enforce all of it. Generic OSAC implementation work is required to parse and validate the proposed registration fields, validate manager compatibility, and invoke the registered collection role. Subnet create already has a generic Fabric-to-Kubernetes output dependency; the contract formalizes its exact data. Generic lifecycle support must add ordered Subnet teardown so Kubernetes cleanup runs before Fabric removes the output ConfigMap. After that support ships, adding a conforming manager requires registration/configuration and Ansible content only; it does not require manager-specific Go changes.

All deployed manager registrations must be updated before contract enforcement is enabled. The current parser accepts only `name`, `description`, and `capabilities`; the proposed `implementationRef` and `compatibleManagers` fields are new. The create-side ConfigMap handoff and validation already exist. The manager contract adds no tenant-facing resource or API fields; it formalizes the existing output keys and adds ordered teardown. An incompatible manager pair prevents dispatch with a diagnostic.

Changing manager names in NetworkClass does not automatically migrate backend state for existing resources. Providers must follow an explicit migration or resource replacement procedure before switching manager assignments. This contract guarantees common API behavior for newly reconciled resources, not transparent state transfer between different backends.

---

## Provenance

Committed: commit @ design 0.11.3 - 2bd6607, workspace docs/unified-networking-docs-structure @ 3d0f65e

> Authoring phases not recorded this session (commit-time snapshot only).

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"commit_only","workflow":"design","workflow_version":"0.11.3","ai_workflows":"2bd6607","source_repo":"3d0f65e","source_repo_branch":"docs/unified-networking-docs-structure","commits_behind_main":0,"commits_ahead_main":19,"main_ref":"main","phases":["commit","commit","commit","commit"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
