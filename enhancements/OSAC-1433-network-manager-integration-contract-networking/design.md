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
  - [4.4 Subnet Orchestration and Handoff Lifecycle](#44-subnet-orchestration-and-handoff-lifecycle)
  - [4.5 Scalability and Performance](#45-scalability-and-performance)
  - [4.6 Security Considerations](#46-security-considerations)
  - [4.7 Failure Handling and Recovery](#47-failure-handling-and-recovery)
  - [4.8 RBAC and Tenancy](#48-rbac-and-tenancy)
  - [4.9 Extensibility and Future-Proofing](#49-extensibility-and-future-proofing)
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

The current operator discovers managers from Kubernetes ConfigMaps and dispatches provider work to Ansible Automation Platform (AAP). A manager registration currently provides a logical name, description, and capabilities, but not a collection role reference, contract version, or peer compatibility declaration. Subnet creation already runs Fabric before Kubernetes and passes Fabric-assigned network-segment identifiers and reserved-address values to the Kubernetes task. The manager resolver does not validate whether a selected Fabric and Kubernetes pair is compatible, AAP dispatch is not yet based on the proposed explicit `implementationRef`, and Subnet teardown currently runs the targets independently. [Codebase: osac-operator/pkg/networkmanager/types.go; osac-operator/pkg/dispatcher/dispatch.go; osac-operator/internal/controller/subnet_controller.go; osac-operator/internal/controller/fabric_output_provider.go; osac-operator/pkg/provisioning/vni_outputs.go]

A provider must be able to declare which manager pairs can consume the same integration data, and reject a pair whose network models cannot interoperate. OSAC needs a declared, checked compatibility boundary and an OSAC-owned handoff protocol, while the managers remain separate implementations. [User]

## 4. Design

### 4.1 Architecture and Manager Roles

A Fabric Manager implements the shared networking resource operations and workload attachment operations assigned to that role. It configures the provider's network fabric for VirtualNetworks, Subnets, security rules, external address pools and attachments, and network address translation (NAT) gateways. It also handles physical workload attachment movement and the supported lease lookup targets.

A Kubernetes Manager implements Kubernetes-side Subnet networking for workloads that need a VM overlay. It does not replace the Fabric Manager and does not implement the shared networking resource operations. A NetworkClass always selects one Fabric Manager. It may select one Kubernetes Manager when the deployment supports workloads that need Kubernetes-side networking.

The OSAC operator resolves the registrations selected by NetworkClass, validates their contract and compatibility, dispatches each operation to its assigned role through Ansible Automation Platform (AAP), and records readiness and job state. Managers remain separate implementations: one manager does not call another or receive its credentials. OSAC owns the integration boundary and orchestration order.

Every networking resource is infrastructure-agnostic: it retains the same meaning for virtual machines, managed clusters, and bare-metal servers. Every manager role is backend-agnostic: any implementation that fulfills its assigned contract can provide that behavior. The [Unified Networking Design](../OSAC-1433-unified-networking/design.md) defines tenant resource semantics; this design defines the provider integration contract.

### 4.2 Registration and Compatibility Data

A **manager registration** is a role-labelled Kubernetes ConfigMap in the operator namespace. The role label identifies a Fabric Manager or Kubernetes Manager. Its `data.name` is the unique logical name selected by NetworkClass. `implementationRef` names the Ansible collection role AAP invokes; the logical name and implementation reference are independent.

Contract v1 requires these registration fields:

| Field | Required | Meaning |
|-------|----------|---------|
| Role label | Yes | Exactly one of `osac.openshift.io/network-fabric-manager: "true"` or `osac.openshift.io/network-k8s-manager: "true"`. |
| `name` | Yes | Unique logical name within the role, selected by NetworkClass. |
| `implementationRef` | Yes | Fully qualified Ansible collection role name, such as `acme.networking.fabric_manager`; it must be installed in the AAP execution environment. |
| `contractVersion` | Yes | Manager contract version implemented by the collection. Contract v1 accepts `v1`. This is a proposed field; current registration parsing does not read it. |
| `capabilities` | Yes | Comma-separated values from the OSAC-defined vocabulary: `ipv4` (Internet Protocol version 4), `ipv6` (Internet Protocol version 6), `dualStack` (both address families), and `dpuSupport` (data processing unit support). Contract v1 requires `ipv4` from each selected role and rejects `ipv6` and `dualStack`. |
| `compatibleManagers` | Yes | Comma-separated logical names in the opposite role that can interoperate with this registration. An empty value means no compatible peer. |
| `description` | No | Human-readable description for provider administration. |

`contractVersion` lets OSAC reject a registration whose task interface or handoff contract it does not understand before dispatch. It is not an API resource version, backend version, or existing registration field.

Compatibility is pair-specific and reciprocal. A Fabric Manager lists compatible Kubernetes Manager names; a Kubernetes Manager lists compatible Fabric Manager names. OSAC accepts a selected pair only when both list each other and both implement the same supported contract version. A manager publisher declares compatibility only when its implementation can consume and produce the shared v1 interface with that peer. No vendor pair is embedded in OSAC code.

Compatibility is based on the data exchanged between a pair, not the manager names. In the example below, Netris publishes a Subnet's Layer 2 segment identifier, its VirtualNetwork's Layer 3 identifier, and the reserved-address range; the [CUDN EVPN Kubernetes Manager](/enhancements/OSAC-4291-cudn-evpn-k8s-manager-phase-1-networking/design.md) consumes those values. They identify a Virtual eXtensible Local Area Network (VXLAN) network and are called VXLAN Network Identifiers (VNIs). The [Agentless VLAN Fabric Manager](/enhancements/OSAC-3664-agentless-vlan-fabric-manager/design.md) uses Virtual Local Area Network (VLAN) identifiers, which the v1 handoff does not carry. In this example, that pair cannot claim compatibility because CUDN EVPN does not consume or translate the Agentless VLAN segment identifier. A future pair may declare compatibility only if both implementations satisfy the same published handoff contract.

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
  contractVersion: "v1"
  capabilities: "ipv4"
  compatibleManagers: "cudn_evpn"
  description: "Fabric integration"
```

#### NetworkClass capability behavior

Manager registrations declare capabilities from a fixed OSAC vocabulary; they cannot define custom capabilities. Contract v1 supports Internet Protocol version 4 (IPv4) only; Internet Protocol version 6 (IPv6) and dual-stack networking (IPv4 and IPv6 together) are unsupported. The operator derives NetworkClass capability output by intersecting the declarations of the selected managers; when no Kubernetes Manager is selected, it uses the Fabric Manager declaration. The resulting API fields and v1 behavior are:

| Registration capability | NetworkClass output | Contract v1 behavior |
|--------------------------|---------------------|-----------------------|
| `ipv4` | `supportsIpv4` | Required from every selected role; IPv4 networking requests are supported. |
| `ipv6` | `supportsIpv6` | Rejected in registrations; output is false and IPv6 requests are unsupported. |
| `dualStack` | `supportsDualStack` | Rejected in registrations; output is false and dual-stack requests are unsupported. |
| `dpuSupport` | `dpuSupport` | Optional; output is true only if every selected role declares it. It reports data processing unit (DPU) networking availability but does not alter v1 operation routing or request validation. |

The NetworkClass `spec.disable_capabilities` control can turn off a capability in the selected-manager intersection; it cannot enable a capability that a selected manager does not declare. The effective NetworkClass capability output is the manager intersection after this provider disable mask, and Unified Networking request validation uses that effective output. Contract v1 requires IPv4 for every selected manager, so a NetworkClass cannot disable IPv4 and remain valid. DPU support can be disabled by the provider; it has no v1 request behavior. The typed east-west capability fields in the Unified Networking API are outside this manager capability vocabulary; see the [Multi-Fabric East-West Networking Design](/enhancements/OSAC-1382-multi-fabric-east-west-networking/design.md). Adding a manager capability that changes request behavior requires a platform contract update.

### 4.3 API and Operation Contract

This contract does not change the tenant-facing networking API. It defines the manager registration, common AAP input, required operations and targets, results, and the stable Fabric-to-Kubernetes Subnet handoff. List and Get operations remain served by OSAC; the operations below cover provider backend reconciliation.

#### Subnet handoff data

When a Kubernetes Manager is selected, the Fabric Manager's `subnet.create` task writes the output ConfigMap `subnet-<subnet-name>-fabric-output` in the Subnet namespace. Contract v1 requires exactly these shared data keys; their ConfigMap values are strings:

The `l2_vni` and `l3_vni` values are 24-bit VXLAN Network Identifiers for the Subnet's Layer 2 network and its parent VirtualNetwork's Layer 3 routing domain. The Fabric Manager assigns them; the Kubernetes Manager consumes the assigned values rather than allocating replacements. The reserved-address value uses Classless Inter-Domain Routing (CIDR) notation where it describes an address range.

| Key | Required value | Consumer behavior |
|-----|----------------|-------------------|
| `l2_vni` | Integer VNI from 1 through 16,777,215. It identifies the Subnet's Layer 2 segment. | The Kubernetes Manager uses the assigned VNI for the Subnet's Layer 2 network; it does not allocate or substitute another VNI. |
| `l3_vni` | Integer VNI from 1 through 16,777,215. It identifies the parent VirtualNetwork's Layer 3 routing domain. | The Kubernetes Manager uses the assigned VNI for the VirtualNetwork's Layer 3 network; it does not allocate or substitute another VNI. |
| `fabric_reserved_range` | A non-empty string representing IPv4 addresses or ranges reserved by the fabric within the Subnet, such as gateway, switch virtual interface, or Dynamic Host Configuration Protocol (DHCP) addresses. | The Kubernetes Manager excludes those addresses from its own workload IP allocation. It validates/translates the value for its native API. |

OSAC reads the ConfigMap only after Fabric provisioning succeeds. It requires all three values, validates both VNI values as integers in the stated range, normalizes them to integers, and passes these values unchanged in meaning to the Kubernetes AAP launch. OSAC checks that `fabric_reserved_range` is non-empty and passes the string through; it does not parse its internal range syntax. A manager pair can declare compatibility only when the Kubernetes implementation understands the Fabric implementation's v1 reserved-range encoding and applies the exclusion correctly.

The handoff is an internal, namespaced ConfigMap and AAP input, not Subnet status or tenant input. Contract v1 defines no VLAN identifier. A pair that needs a VLAN identifier or another shared value cannot claim v1 compatibility; adding that value requires a platform contract update.

#### Common AAP input

OSAC passes a common job envelope in the `osac_job_vars` variable. The task-specific resource is the OSAC resource being reconciled. The manager reference selects the collection role and contract, while the operation names one fixed task from the operation table.

```yaml
osac_job_vars:
  operation: subnet.create
  manager:
    name: netris
    role: fabric
    implementationRef: acme.networking.fabric_manager
    contractVersion: v1
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

#### Fixed operation and target table

| Operation | Assigned role and order | Workload target | Fixed task entry point | Required behavior and result |
|-----------|-------------------------|-----------------|-------------------------|------------------------------|
| `virtual_network.create` / `virtual_network.delete` | Fabric | N/A | `create_virtual_network` / `delete_virtual_network` | Create or remove the isolated VirtualNetwork routing domain and associated allocation. Subnets in the same VirtualNetwork are L3-routable to one another; separate VirtualNetworks remain isolated. |
| `subnet.create` / `subnet.delete` | Create: Fabric, then Kubernetes when configured. Delete: Kubernetes, then Fabric. | N/A | `create_subnet` / `delete_subnet` | Fabric creates/removes one L2 broadcast domain for the Subnet. When a Kubernetes role is selected, Fabric writes the output ConfigMap on create and removes it during its delete task. Workloads in the same Subnet share that L2 domain and are L3-routable within their parent VirtualNetwork. Kubernetes creates/removes its network side using all three exact handoff values. The Subnet CIDR belongs to its parent VirtualNetwork and cannot overlap a sibling Subnet. |
| `security_group.apply` / `security_group.delete` | Fabric | N/A | `create_security_group` / `delete_security_group` | Apply the complete stateful ingress and egress rule set at workload network attachments, remove obsolete rules on apply, and remove rules owned by the group on delete. |
| `external_ip_pool.create` / `external_ip_pool.delete` | Fabric | N/A | `create_external_ip_pool` / `delete_external_ip_pool` | Register or remove the provider address pool. The pool has one canonical IPv4 Classless Inter-Domain Routing (CIDR) prefix; OSAC owns API capacity counters. |
| `external_ip.allocate` / `external_ip.release` | Fabric | N/A | `create_external_ip` / `delete_external_ip` | Reserve or release one address from the selected pool. After durable allocation, the task writes the address to `osac.openshift.io/allocated-address`. Repeated allocation for the same resource unique identifier returns the same address. |
| `external_ip_attachment.create` / `external_ip_attachment.delete` | Fabric | `compute_instance`, `cluster`, `baremetal_instance` | `attach_external_ip` / `detach_external_ip` | Add or remove inbound translation for the resource target or configured Cluster endpoint. Remove the attachment before releasing its ExternalIP. |
| `nat_gateway.create` / `nat_gateway.delete` | Fabric | N/A | `create_nat_gateway` / `delete_nat_gateway` | Add or remove outbound source network address translation (SNAT) for the VirtualNetwork using its ExternalIP. It does not provide inbound access. |
| `workload_attachment.move` | Fabric | `cluster`, `baremetal_instance` | `move_network_attachment` | Move a physical workload port to its selected Subnet when `metadata.deletionTimestamp` is absent; restore the provisioning network when deletion is in progress. Both directions are retry-safe. |
| `dhcp_lease.query` | Fabric | `cluster`, `baremetal_instance` | `query_dhcp_lease` | Return the Dynamic Host Configuration Protocol (DHCP) lease matching each requested network attachment as the AAP artifact `leases`. |

Every implementation must provide the complete operation and workload-target combinations assigned to its role. A registration does not select an operation subset. A manager with a different internal model must translate the fixed OSAC operation into that model; it cannot route the operation to the other manager role.

The `leases` artifact contains entries with `subnet_ref`, `interface`, `ip_address`, and `mac_address`. A missing or ambiguous lease match is a task failure. OSAC owns API resource phase, conditions, and provisioning job history.

#### Operation result envelope

Every successful operation except `dhcp_lease.query` returns an AAP artifact named `osac_result` with this contract-v1 shape. The `resourceUID` field identifies the Kubernetes object's unique identifier (UID); `observedGeneration` identifies the version of its specification that the manager processed.

```yaml
osac_result:
  schemaVersion: v1
  operation: subnet.create
  resourceUID: "<resource UID>"
  observedGeneration: 3
  data: {}
```

`schemaVersion` identifies the result format and is distinct from the manager registration's `contractVersion`. `operation`, `resourceUID`, and `observedGeneration` must match the operation OSAC dispatched and the resource version it dispatched. In contract v1, `data` is an empty object; operation-specific outputs use their separately defined channels. ExternalIP allocation reports its address through the guarded `osac.openshift.io/allocated-address` annotation, not in `data`. Subnet Fabric outputs use the separate namespaced ConfigMap defined above, not this envelope. `dhcp_lease.query` returns the `leases` artifact instead of `osac_result`. Adding fields or result data requires an OSAC contract update.

OSAC validates the artifact name and all required fields before accepting task success. A missing, malformed, unsupported-version, stale, or mismatched envelope is a failed operation and cannot advance resource readiness or release capacity. A manager must not report a successful operation with an envelope for a different operation, resource UID, or generation.

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

The current `origin/main` create lifecycle already waits for Fabric success, reads this ConfigMap, validates/normalizes the values, and passes the required values to the Kubernetes AAP target. The target contract makes this exact three-key exchange normative. Current Subnet deletion runs Fabric and Kubernetes targets independently; generic OSAC lifecycle support must add the reverse dependency so Kubernetes cleanup succeeds before Fabric cleanup removes the ConfigMap. Each task remains independently implemented and idempotent; OSAC owns ordering and retries.

### 4.5 Scalability and Performance

Manager registration and compatibility lists are small and are read during manager discovery or NetworkClass validation. The handoff is bounded to three small ConfigMap values per Subnet and requires no new service or backend database. Subnet provisioning adds an ordered AAP stage when a Kubernetes Manager is configured, so readiness waits for both jobs to finish.

### 4.6 Security Considerations

Manager ConfigMaps contain references and compatibility metadata, not credentials. Existing Kubernetes role-based access control (RBAC) in the operator namespace protects registrations. AAP credentials or provider-managed Secrets supply backend access; Ansible job artifacts and logs must not disclose secrets. The handoff contains only the two VNI identifiers and the fabric-reserved address string. Its ConfigMap is scoped to the Subnet namespace and OSAC reads the exact name derived from that Subnet. Manager tasks act only on the tenant-scoped resources OSAC passes and must preserve existing tenant and owner-reference boundaries.

### 4.7 Failure Handling and Recovery

- **Missing or invalid registration:** NetworkClass validation reports the role, ConfigMap, and invalid field. OSAC does not dispatch resource work.
- **Incompatible manager pair:** NetworkClass validation fails with both selected logical names and the missing reciprocal declaration. OSAC starts no AAP job for resources using that NetworkClass.
- **Unsupported contract version or missing collection role:** OSAC rejects a registration with an unsupported version before dispatch, or AAP fails with the missing fully qualified collection role/task name. The diagnostic names the registration or task.
- **Missing or malformed Fabric outputs:** A missing output ConfigMap, missing required key, empty reserved-range value, or invalid VNI prevents the Kubernetes stage from starting. OSAC reports the missing or invalid output and retries/fails through the existing provisioning lifecycle.
- **Kubernetes create failure after Fabric success:** OSAC retains the Fabric output ConfigMap and retries the Kubernetes stage using the same values. Fabric create is idempotent. The Subnet is not Ready until the Kubernetes stage succeeds.
- **Kubernetes delete failure:** OSAC does not start Fabric deletion, so the output ConfigMap and segment remain available while Kubernetes detachment retries.
- **Fabric delete failure after Kubernetes detach:** OSAC retains job state and retries Fabric cleanup. The Kubernetes target is already detached; repeated Fabric deletion of absent state succeeds.
- **Backend timeout or transient error:** The AAP task returns a diagnostic and a non-success result. Existing OSAC job retry/backoff behavior retries the operation. All create/apply and delete tasks are idempotent by resource UID.
- **Invalid operation result:** A missing, malformed, stale, or mismatched `osac_result` fails the operation; OSAC does not advance readiness or release capacity.
- **Invalid ExternalIP or lease result:** Missing allocated-address output or malformed/ambiguous `leases` output fails the task; OSAC does not report allocation or lease discovery as successful.

### 4.8 RBAC and Tenancy

No tenant-facing RBAC changes are required. The fulfillment service continues to authorize tenant networking requests. Manager registrations and backend credentials remain provider-scoped. OSAC passes only the resource context authorized for the requested tenant; managers must not read or modify resources belonging to another tenant.

### 4.9 Extensibility and Future-Proofing

A new manager is onboarded by installing its collection into the AAP execution environment, registering the role with its contract version, capabilities, and compatible peer names, and selecting it in NetworkClass. No manager-specific Go code or tenant API change is required after OSAC implements contract v1. A new operation, workload target, capability with new behavior, segment encapsulation, handoff field, or incompatible task payload changes the OSAC contract and requires platform support before an implementation can use it.

## 5. Interface Changes

### IC-1: Versioned manager registration

**Requirements:** FR-1, FR-2, FR-3

Manager ConfigMaps require `implementationRef`, `contractVersion`, and `compatibleManagers`; when both roles are selected, the compatibility declarations must be reciprocal. §4.2 specifies the registration fields and validation. `contractVersion` is a proposed new field, not an existing registration field.

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

An opaque vendor map would let each Fabric Manager return its own schema, but the Kubernetes Manager would need vendor-specific parsing. The v1 three-key output contract has a stable common shape and leaves backend-only identifiers inside the Fabric Manager.

## 7. Observability and Monitoring

No new metrics are required. Existing NetworkClass state/message, resource conditions, events, AAP job history, and reconciliation logs report registration, compatibility, operation, and backend errors. Diagnostics identify the manager role and logical name, operation stage, and invalid handoff field. Logs do not include credential values.

## 8. Impact and Compatibility

This is the target contract; the current operator and AAP implementation do not yet enforce all of it. Generic OSAC implementation work is required to parse and validate the proposed registration fields, validate manager compatibility, and invoke the registered collection role. Subnet create already has a generic Fabric-to-Kubernetes output dependency; the contract formalizes its exact data. Generic lifecycle support must add ordered Subnet teardown so Kubernetes cleanup runs before Fabric removes the output ConfigMap. After that support ships, adding a conforming manager requires registration/configuration and Ansible content only; it does not require manager-specific Go changes.

All deployed manager registrations must be updated before contract enforcement is enabled. The current parser accepts only `name`, `description`, and `capabilities`; the proposed `implementationRef`, `contractVersion`, and `compatibleManagers` fields are new. The create-side ConfigMap handoff and validation already exist. Contract v1 adds no tenant-facing resource or API fields; it formalizes the existing output keys and adds ordered teardown. An unsupported version or incompatible manager pair prevents dispatch with a diagnostic.

Changing manager names in NetworkClass does not automatically migrate backend state for existing resources. Providers must follow an explicit migration or resource replacement procedure before switching manager assignments. Contract v1 guarantees common API behavior for newly reconciled resources, not transparent state transfer between different backends.

---

## Provenance

Committed: commit @ design 0.11.3 - 2bd6607, workspace docs/unified-networking-docs-structure @ 09edfec

> Authoring phases not recorded this session (commit-time snapshot only).

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"commit_only","workflow":"design","workflow_version":"0.11.3","ai_workflows":"2bd6607","source_repo":"09edfec","source_repo_branch":"docs/unified-networking-docs-structure","commits_behind_main":0,"commits_ahead_main":17,"main_ref":"main","phases":["commit","commit","commit"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
