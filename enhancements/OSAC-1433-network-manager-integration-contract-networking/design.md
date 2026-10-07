---
title: network-manager-integration-contract
authors:
  - dmanor@redhat.com
creation-date: 2026-10-04
last-updated: 2026-10-07
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
| Date | 2026-10-07 |

## Contents

- [1. Overview](#1-overview)
- [2. Goals and Non-Goals](#2-goals-and-non-goals)
- [3. Motivation / Background](#3-motivation--background)
- [4. Design](#4-design)
  - [4.1 Architecture and Manager Roles](#41-architecture-and-manager-roles)
  - [4.2 Network Data Model Catalog and Manager Registration](#42-network-data-model-catalog-and-manager-registration)
  - [4.3 API and Operation Contract](#43-api-and-operation-contract)
    - [Resource-scoped network outputs and inputs](#resource-scoped-network-outputs-and-inputs)
    - [Common AAP input](#common-aap-input)
    - [Workload attachment policy input](#workload-attachment-policy-input)
    - [Fixed operation and target table](#fixed-operation-and-target-table)
    - [Operation result envelope](#operation-result-envelope)
  - [4.4 Subnet Orchestration and Network Data Lifecycle](#44-subnet-orchestration-and-network-data-lifecycle)
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

A Fabric Manager configures the provider's physical network fabric for OSAC. A Kubernetes Manager connects Kubernetes-hosted workloads to that fabric. NetworkClass selects one manager for each configured role. NetworkDataModel defines the meaning, owner scope, and JSON Schema for a value managers exchange. NetworkManager registers one manager implementation, role, capabilities, and model inputs or outputs.

NetworkDataModel and NetworkManager are cluster-scoped OSAC API objects. Administrators create model objects first, manager objects second, and NetworkClass last. Registry specifications are immutable and expose Create, Read/List, and Delete only. OSAC validates references at each step and rejects an incompatible manager selection before persisting NetworkClass. Runtime values remain in separate owner-scoped output ConfigMaps.

This design defines the source-neutral manager contract, model registry, Ansible Automation Platform (AAP) task interface, and OSAC-owned exchange between managers. A Fabric Manager publishes only the model values needed by a selected Kubernetes Manager; OSAC validates schemas and owner scopes before passing matching values to the consumer. OSAC coordinates both roles, and one manager does not call another directly. Once generic OSAC registry and manager support exists, onboarding an implementation requires registry objects and Ansible content, with no implementation-specific Go code. The Unified Networking Design defines the shared resource semantics; its PRD and this design define the intended provider and implementation-author outcomes.

## 2. Goals and Non-Goals

### 2.1 Goals

- Specify the complete implementation contract for each manager role, including registration, the network data model catalog, operations, task inputs, results, and retries.
- Let OSAC validate selected manager combinations by matching the Kubernetes Manager's declared network inputs to outputs declared by the Fabric Manager.
- Pass only the required, schema-validated JSON values through a stable OSAC-owned interface with owner scope and lifetime.
- Allow a conforming implementation from any source to use the same tenant networking application programming interface (API) and OSAC dispatch behavior.
- Keep future manager onboarding within registration and Ansible content after generic OSAC contract support is implemented.

### 2.2 Non-Goals

- Change the meaning, fields, or lifecycle of tenant networking resources defined by Unified Networking.
- Require every Fabric Manager to work with every Kubernetes Manager.
- Define product-specific configuration for any backend.
- Let an implementation add resource kinds, operation identifiers, workload targets, or tenant API fields that OSAC does not understand.
- Migrate existing backend resources when a provider changes the manager selected by a NetworkClass.

## 3. Motivation / Background

The current operator discovers manager implementations and dispatches provider work through Ansible Automation Platform (AAP). Its manager parser does not yet implement the proposed implementation reference or generic input/output model declarations. The target contract adds first-class NetworkDataModel and NetworkManager objects, validated owner-scoped output artifacts, and an input resolver so selected managers exchange only the declared values they need. Current OSAC code does not yet enforce this generic contract; platform implementation work must add it.

A provider must be able to declare the data each implementation produces or needs, and OSAC must reject invalid object references and incompatible selections before a NetworkClass is persisted. The declarations establish a data boundary; they do not replace conformance to the shared networking behavior. Managers remain separate implementations.

## 4. Design

### 4.1 Architecture and Manager Roles

A Fabric Manager implements the shared networking resource operations and physical workload attachment operations assigned to that role. It configures the provider's network fabric for VirtualNetworks, Subnets, external address pools and attachments, and network address translation gateways. It enforces SecurityGroup policy on physical interfaces, attaches bare-metal and CaaS worker interfaces to Subnets, and handles supported lease lookup targets.

A Kubernetes Manager implements Kubernetes-side Subnet networking and SecurityGroup enforcement for ComputeInstance VM overlays. It does not replace the Fabric Manager and does not implement shared networking-resource operations. Every NetworkClass selects one Fabric Manager. It may select one Kubernetes Manager when the deployment supports workloads that need Kubernetes-side networking.

NetworkDataModel objects define reusable values. NetworkManager objects register the independent implementations that produce or consume those values. NetworkClass selects the registrations. The fulfillment service validates manager references and exact model-ID compatibility before persisting NetworkClass; the operator then resolves the validated manager objects and dispatches each operation to its assigned role through AAP. Managers do not call one another or receive one another's credentials. OSAC owns the integration boundary and orchestration order.

Every networking resource is infrastructure-agnostic: it retains the same meaning for virtual machines, managed clusters, and bare-metal servers. Every manager role is backend-agnostic: any implementation that fulfills its assigned contract can provide that behavior. The Unified Networking Design defines tenant resource semantics; this design defines the provider integration contract.

### 4.2 Network Data Model Catalog and Manager Registration

A NetworkDataModel is a cluster-scoped OSAC API object that defines one reusable value exchanged between manager roles. Its Kubernetes metadata.name is the stable model ID. The ID identifies both the value's meaning and its schema contract; matching JSON types alone do not make two models interchangeable. For example, a VLAN ID and a VXLAN VNI are different models even when both are represented as integers. The osac. prefix is reserved for definitions shipped by OSAC; providers use their own namespace.

The model's spec.ownerScope identifies the single OSAC resource that owns each value. OSAC stores one value for each model ID and owner UID; a JSON array or object represents a value with multiple members. NetworkClass scope means a value is shared by resources using that provider profile; resource scope means each instance of that resource kind owns its own value. OSAC resolves values only from the operation resource and its available networking context. A model whose owner is not present in that context is not applicable to that operation. New resource kinds and operation contexts still require OSAC API and lifecycle support.

NetworkDataModel objects use API version osac.openshift.io/v1alpha1 and are cluster-scoped. The OSAC installation supplies the common networking models below as these objects. Providers create additional model objects before creating managers that reference them.

| Field | Required | Meaning |
|---|---|---|
| metadata.name | Yes | Stable, unique, lowercase, dot-separated model ID. Its value is the ID managers reference. |
| spec.description | Yes | Human-readable meaning and intended use of the value. |
| spec.ownerScope | Yes | One supported data owner: NetworkClass, VirtualNetwork, or Subnet. Adding another owner kind requires OSAC resource-context support. |
| spec.schema | Yes | JSON Schema Draft 2020-12 for the value. It may describe any JSON value. Local schema references may be used; remote references and unknown schema vocabularies are rejected. |

The model ID, meaning, owner scope, and schema are immutable. Provider-facing actions are Create, Read/List, and Delete; Update and Patch are rejected. A change to meaning, owner scope, or schema requires a new model ID. Runtime output artifacts remain separate owner-scoped ConfigMaps; registry objects contain definitions, never runtime values, credentials, or secrets.

For example, this provider-defined model carries an object with two fields:

~~~yaml
apiVersion: osac.openshift.io/v1alpha1
kind: NetworkDataModel
metadata:
  name: acme.networking.subnet.segment
spec:
  description: Provider segment identity consumed by a Kubernetes Manager.
  ownerScope: Subnet
  schema:
    $schema: https://json-schema.org/draft/2020-12/schema
    type: object
    properties:
      fabric:
        type: string
        minLength: 1
      segment:
        type: integer
        minimum: 1
    required: [fabric, segment]
    additionalProperties: false
~~~

OSAC validates model IDs, owner scopes, and schemas when a model object is created. Invalid schemas, unsupported dialects or vocabularies, remote references, unsupported owner scopes, and duplicate model IDs are rejected with an admission diagnostic naming the object and invalid field. JSON Schema validates structure, types, and expressible constraints. It does not prove that a manager configured its backend correctly or that a value satisfies a relationship with other OSAC resources unless that relationship is encoded in a generic OSAC rule. Managers remain responsible for resource-relative semantics, and conformance verifies they honor the model description and shared networking behavior.

| Model ID | Owner scope | JSON Schema assertions | Meaning and additional conformance |
|---|---|---|---|
| osac.networking.virtual-network.vxlan-l3-vni | VirtualNetwork | Integer from 1 through 16,777,215. | Identifies the VirtualNetwork Layer 3 VXLAN domain. It may be allocated during the first Subnet operation but remains owned by the VirtualNetwork. [RFC 7348](https://www.rfc-editor.org/rfc/rfc7348.html) |
| osac.networking.subnet.vxlan-l2-vni | Subnet | Integer from 1 through 16,777,215. | Identifies the Subnet's VXLAN Layer 2 segment. [RFC 7348](https://www.rfc-editor.org/rfc/rfc7348.html) |
| osac.networking.subnet.vlan-id | Subnet | Integer from 1 through 4094. | Identifies the Subnet's IEEE 802.1Q VLAN segment. [RFC 2674](https://www.rfc-editor.org/rfc/rfc2674.html) |
| osac.networking.subnet.reserved-ipv4-cidrs | Subnet | JSON array, possibly empty, whose items are strings. | Each value is an IPv4 CIDR reserved by the fabric and contained within the Subnet. Managers validate this resource-relative meaning; an overlay IP address manager excludes every listed range from workload allocation. |

A NetworkManager is a cluster-scoped OSAC API object that registers one Fabric or Kubernetes Manager implementation. It uses API version osac.openshift.io/v1alpha1. Its spec.managerName is the immutable logical name selected by NetworkClass; metadata.name is a separate DNS-safe Kubernetes object name. spec.implementationRef identifies the fully qualified Ansible collection role AAP invokes. The logical name and implementation reference are independent.

| Field | Required | Meaning |
|---|---|---|
| metadata.name | Yes | DNS-safe Kubernetes object name, unique across NetworkManager objects. It need not match the logical manager name. |
| spec.managerName | Yes | Immutable logical name referenced by NetworkClass; unique within its role. Existing names may contain underscores. |
| spec.role | Yes | Exactly Fabric or Kubernetes. |
| spec.implementationRef | Yes | Fully qualified Ansible collection role name, such as acme.networking.fabric_manager; the role must be installed in the AAP execution environment. |
| spec.capabilities | Yes | List from the OSAC-defined vocabulary. This proposal accepts exactly ipv4; ipv6, dualStack, and unknown values are rejected. |
| spec.networkOutputs | Fabric only | Required list, possibly empty, of model IDs this Fabric Manager can produce. |
| spec.networkInputs | Kubernetes only | Required list, possibly empty, of model IDs this Kubernetes Manager requires whenever the model owner is present for one of its assigned operations. |
| spec.description | No | Human-readable description for provider administration. |

A Fabric object must set networkOutputs and must not set networkInputs. A Kubernetes object must set networkInputs and must not set networkOutputs. Each listed ID must name an existing NetworkDataModel object; repeated and unknown IDs are rejected. An empty list means the manager has no declarations for that direction. OSAC admission validation rejects malformed objects and cannot accept a NetworkManager until all referenced models exist.

The pair (spec.role, spec.managerName) must be unique across NetworkManager objects. OSAC rejects duplicate pairs at admission so each NetworkClass reference resolves to exactly one registration. Kubernetes already requires metadata.name to be unique across all NetworkManager objects.

~~~yaml
apiVersion: osac.openshift.io/v1alpha1
kind: NetworkManager
metadata:
  name: netris-manager
spec:
  managerName: netris
  role: Fabric
  implementationRef: acme.networking.fabric_manager
  description: Fabric integration
  capabilities: [ipv4]
  networkOutputs:
    - osac.networking.virtual-network.vxlan-l3-vni
    - osac.networking.subnet.vxlan-l2-vni
    - osac.networking.subnet.reserved-ipv4-cidrs
~~~

~~~yaml
apiVersion: osac.openshift.io/v1alpha1
kind: NetworkManager
metadata:
  name: cudn-evpn
spec:
  managerName: cudn_evpn
  role: Kubernetes
  implementationRef: acme.networking.vm_overlay
  description: Kubernetes VM overlay integration
  capabilities: [ipv4]
  networkInputs:
    - osac.networking.virtual-network.vxlan-l3-vni
    - osac.networking.subnet.vxlan-l2-vni
    - osac.networking.subnet.reserved-ipv4-cidrs
~~~

The selection sequence is NetworkDataModel objects, then NetworkManager objects, then NetworkClass. NetworkClass continues to store the existing logical manager names. When it is created, the fulfillment service resolves each name against the cluster-scoped NetworkManager objects by role and spec.managerName, then validates manager capabilities and that every Kubernetes input ID appears in the selected Fabric Manager outputs. The validator uses a read-only Kubernetes API client and fails closed if the registry cannot be read. An invalid or incompatible selection is rejected before NetworkClass is persisted; no tenant networking resource or AAP job can use it.

The comparison uses exact model IDs, not only JSON shape, because identity, meaning, and owner scope are part of the contract. A matching declaration establishes that the selected implementation advertises required data; it does not prove that either manager preserves the shared VirtualNetwork L3, Subnet L2, attachment, or SecurityGroup behavior. Each implementation and selected path must also pass conformance. OSAC does not contain a manager-name compatibility table.

| Fabric Manager | Kubernetes Manager | Result |
|---|---|---|
| netris outputs VXLAN L3 VNI, VXLAN L2 VNI, and reserved IPv4 CIDRs | cudn_evpn requires those same model IDs | Accepted |
| agentless_net outputs the Subnet VLAN ID | A Kubernetes Manager requires VXLAN VNIs and reserved IPv4 CIDRs | Rejected because the required model IDs are absent |
| agentless_net outputs the Subnet VLAN ID | A future LocalNet Manager requires the Subnet VLAN ID | Accepted by the data contract; this does not claim LocalNet support in the current CUDN proposal |

A provider-defined model uses the same path. The Fabric Manager lists its model ID in networkOutputs, and the Kubernetes Manager lists the same ID in networkInputs. OSAC places a produced value in an output artifact as model ID plus JSON value, validates it against the registered schema, and passes the matching entry to the consumer. The consumer's Ansible role interprets the documented fields and maps them to its backend API. OSAC does not need a Go type or branch for a provider-defined model; the provider still supplies both managers' Ansible behavior and conformance evidence.

Manager and data model specs are immutable. Providers may Create, Read/List, and Delete these objects; OSAC rejects Update and Patch. A manager deletion is blocked while a NetworkClass selects its role and spec.managerName. A model deletion is blocked while any NetworkManager references its ID or a retained owner-scoped output artifact contains a value under that ID. To replace manager configuration, remove dependent tenant resources and the NetworkClass as required by the Unified Networking lifecycle, delete the old NetworkManager, then create the replacement. A change to a model's meaning, owner scope, or JSON Schema requires a new model ID.

#### NetworkClass capability behavior

Manager registrations declare capabilities from a fixed OSAC vocabulary; they cannot define custom capabilities. This proposal supports Internet Protocol version 4 (IPv4) only; Internet Protocol version 6 (IPv6) and dual-stack networking (IPv4 and IPv6 together) are unsupported. Every selected manager, including the Fabric Manager and any selected Kubernetes Manager, must declare `ipv4`. A missing declaration or unsupported value makes NetworkClass invalid and prevents provider dispatch. OSAC derives the read-only NetworkClass IP-family output from those declarations. The resulting API fields and behavior are:

| Registration capability | NetworkClass output | Behavior |
|--------------------------|---------------------|-----------------------|
| `ipv4` | `supportsIpv4` | Required from every selected role; every valid NetworkClass reports true and the networking API accepts IPv4 addresses and prefixes. |
| `ipv6` | `supportsIpv6` | Rejected in registrations; output is false and IPv6 inputs are rejected by the networking API. |
| `dualStack` | `supportsDualStack` | Rejected in registrations; output is false and dual-stack inputs are rejected by the networking API. |

These manager-derived fields are distinct from `NetworkClass.spec.east_west_capabilities`, which the provider declares to advertise supported FabricDomain types. East-west capability declaration and validation are defined by the Unified Networking and [Multi-Fabric East-West Networking designs](/enhancements/OSAC-1382-multi-fabric-east-west-networking/design.md); they are not part of manager registration or the manager intersection. Adding a manager capability that changes request behavior requires a platform contract update.

### 4.3 API and Operation Contract

This contract does not change the tenant-facing networking API. It defines the manager registration, common AAP input, required operations and targets, results, and the OSAC-owned exchange of declared network data. List and Get operations remain served by OSAC; the operations below cover provider backend reconciliation.

#### Resource-scoped network outputs and inputs

A Fabric Manager publishes its declared values through a durable, OSAC-owned ConfigMap in the networking hub namespace. OSAC creates each target ConfigMap before dispatch and supplies `network_output_targets`; each target identifies the owning resource kind and UID, the namespace, the OSAC-generated ConfigMap name, and the model IDs required at that scope for the operation. OSAC derives those IDs from the selected Kubernetes Manager's `networkInputs`, the model owner scopes available in the operation context, and values not already published. Thus a provider-defined model uses the same target generation and validation path as a model shipped by OSAC. There is one output artifact per owner resource UID, not one per manager pair or operation. The Fabric task writes the JSON document only to the targets OSAC provides and does not change tenant resource status.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: network-output-<resource-uid>
  namespace: "<networking-hub-namespace>"
data:
  outputs.json: |
    {
      "resource_kind": "Subnet",
      "resource_uid": "<subnet-uid>",
      "outputs": [
        {"model_id": "osac.networking.subnet.vxlan-l2-vni", "value": 40120},
        {"model_id": "osac.networking.subnet.reserved-ipv4-cidrs", "value": ["192.0.2.0/28", "192.0.2.32/27"]}
      ]
    }
```

The resource kind and UID in the document must match the `network_output_targets` entry. Each `model_id` must be declared by the Fabric Manager, listed in that target's `model_ids`, and have an `ownerScope` matching the target resource kind. For a specific operation, `model_ids` lists the values required for that resource target; the Fabric task must publish each listed value. OSAC validates the JSON structure, model ID, owner scope, and value against the registered JSON Schema after the Fabric job succeeds. Unknown, duplicate, missing-required, wrong-scope, or schema-invalid outputs fail the operation and prevent the Kubernetes stage from starting. The producer may publish one parent-VirtualNetwork artifact and one Subnet artifact during `subnet.create`; a VNI allocated during this operation is still stored under the VirtualNetwork UID when the VirtualNetwork owns it.

OSAC serializes writes to the same owner artifact. After validation, it merges newly requested model values by ID and preserves other valid entries already stored for that owner. A manager cannot replace a value for the same model ID and owner UID during that owner's lifetime. Retries and later operations reuse the stored value; deleting an owner removes its values only after dependent cleanup succeeds.

Before dispatching a Kubernetes operation, OSAC resolves the selected manager's `networkInputs` from the operation resource and its available context. An input applies when its model's `ownerScope` is represented in that context; OSAC requires exactly one valid value for each applicable owner and fails before dispatch if a value is missing or invalid. It then passes the normalized entries inside `osac_job_vars.network_inputs`. A Subnet operation may receive values owned by the Subnet, its parent VirtualNetwork, and its NetworkClass. A workload-attachment operation may receive values owned by the attached Subnet, its parent VirtualNetwork, and the NetworkClass. The Fabric Manager's implementation details and credentials are not passed to the Kubernetes Manager. For example:

```yaml
osac_job_vars:
  network_inputs:
    - model_id: osac.networking.virtual-network.vxlan-l3-vni
      resource_kind: VirtualNetwork
      resource_uid: "<virtual-network-uid>"
      value: 4020
    - model_id: osac.networking.subnet.vxlan-l2-vni
      resource_kind: Subnet
      resource_uid: "<subnet-uid>"
      value: 40120
    - model_id: osac.networking.subnet.reserved-ipv4-cidrs
      resource_kind: Subnet
      resource_uid: "<subnet-uid>"
      value: ["192.0.2.0/28", "192.0.2.32/27"]
```

The Kubernetes Manager receives only values whose model IDs it declared and whose owner scopes apply to the operation. It validates and translates them into its native API fields; it must not allocate a replacement value or infer one model from another. Output artifacts remain available while any dependent create or delete operation may consume them. An artifact is removed only after its owner is deleted and all dependent manager cleanup that could consume it has succeeded. This applies to NetworkClass-scoped profile data as well as resource-scoped values.

#### Common AAP input

OSAC passes a common job envelope in the `osac_job_vars` variable. The task-specific resource is the OSAC resource being reconciled. The `manager.name` field carries the selected NetworkManager's spec.managerName; metadata.name identifies the Kubernetes object only. The manager reference selects the collection role, while the operation names one fixed task from the operation table. `manager.role` is normalized by OSAC to exactly `fabric` or `kubernetes`; a collection cannot choose or override it. Fabric tasks receive `network_output_targets` for required, not-yet-published model values whose owners are available in the operation context. Kubernetes tasks receive applicable values for model IDs in their registration's `networkInputs`; that list is empty when the selected manager declares no cross-manager data dependency or no declared owner scope applies to the operation.

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

For example, when CUDN declares the three model IDs below as inputs and none of these values has already been published, the Fabric `subnet.create` task receives these output targets. OSAC computes the target list from the model catalog and operation context; the manager registration does not hard-code an operation-to-model mapping. A paired Kubernetes task receives the `network_inputs` entries shown above:

```yaml
osac_job_vars:
  operation: subnet.create
  network_output_targets:
    - namespace: "<networking-hub-namespace>"
      resource_kind: VirtualNetwork
      resource_uid: "<virtual-network-uid>"
      config_map_name: network-output-<virtual-network-uid>
      model_ids: [osac.networking.virtual-network.vxlan-l3-vni]
    - namespace: "<networking-hub-namespace>"
      resource_kind: Subnet
      resource_uid: "<subnet-uid>"
      config_map_name: network-output-<subnet-uid>
      model_ids:
        - osac.networking.subnet.vxlan-l2-vni
        - osac.networking.subnet.reserved-ipv4-cidrs
```

OSAC validates and resolves these values; the Kubernetes task must not read Fabric credentials, call the Fabric Manager directly, or change OSAC resource semantics. The Fabric task uses a separate, job-scoped credential to update only the pre-created output ConfigMaps named in `network_output_targets`. OSAC does not put this credential in `osac_job_vars` or pass it to the Kubernetes Manager. A successful task means that the backend has converged to the requested state unless the operation defines a result artifact below.

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
|-----------|-------------------------|-----------------|-------------------------|---------------------------

---

## Provenance

Authored: draft @ design 0.11.3 - 2bd6607, workspace main @ 1f3b63b82 (52 behind origin/main)
Final: revise @ design 0.11.3 - 2bd6607, workspace main @ 1f3b63b82 (99 behind origin/main, dirty)

> Context changed between draft and revise.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.3","ai_workflows":"2bd6607","source_repo":"1f3b63b82 (dirty)","source_repo_branch":"main","commits_behind_main":99,"commits_ahead_main":0,"main_ref":"main","phases":["draft","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
