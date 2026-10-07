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
    - [East-West capability boundary](#east-west-capability-boundary)
  - [4.3 API and Operation Contract](#43-api-and-operation-contract)
    - [Provider registry API operations](#provider-registry-api-operations)
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

A Fabric Manager configures the provider's physical network fabric for OSAC. A Kubernetes Manager connects Kubernetes-hosted workloads to that fabric. NetworkClass selects one manager for each configured role. NetworkDataModel defines the meaning, owner scope, and JSON Schema for a value managers exchange. NetworkManager registers one manager implementation, role, and model inputs or outputs.

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

OSAC coordinates the manager roles and orchestration order. Managers do not call one another or receive one another's credentials.

Every networking resource is infrastructure-agnostic: it retains the same meaning for virtual machines, managed clusters, and bare-metal servers. Every manager role is backend-agnostic: any implementation that fulfills its assigned contract can provide that behavior. The Unified Networking Design defines tenant resource semantics; this design defines the provider integration contract.

### 4.2 Network Data Model Catalog and Manager Registration

A NetworkDataModel is a cluster-scoped OSAC API object that defines one reusable value exchanged between manager roles. Its Kubernetes `metadata.name` is the canonical name managers reference; the API has no second identifier field.

The name identifies the value's meaning and schema contract, so matching JSON types alone do not make two models interchangeable. For example, a VLAN ID and a VXLAN VNI are different models even when both are represented as integers. The `osac.` prefix is reserved for definitions shipped by OSAC; providers use their own namespace.

Kubernetes generates `metadata.uid` for each object lifetime. Manager declarations reference a model by its stable `metadata.name`; runtime values are keyed by the owning NetworkClass, VirtualNetwork, or Subnet UID so deleting and recreating an owner cannot inherit stale output. [Kubernetes names and UIDs](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/).

The model's `spec.ownerScope` identifies the single OSAC resource that owns each value. OSAC stores one value for each NetworkDataModel name and owner UID; a JSON array or object represents one value with multiple members. NetworkClass scope means a value is shared by resources using that provider profile; resource scope means each instance of that resource kind owns its own value. OSAC resolves values only from the operation resource and its available networking context. A model whose owner is not present in that context is not applicable to that operation. New resource kinds and operation contexts still require OSAC API and lifecycle support.

NetworkDataModel objects use API version osac.openshift.io/v1alpha1 and are cluster-scoped. The OSAC installation supplies the common networking models below as these objects. Providers create additional model objects before creating managers that reference them.

| Field | Required | Meaning |
|---|---|---|
| metadata.name | Yes | Stable, unique, lowercase, dot-separated resource name. Managers use this exact name as the reference. |
| spec.description | Yes | Human-readable meaning and intended use of the value. |
| spec.ownerScope | Yes | One supported data owner: NetworkClass, VirtualNetwork, or Subnet. Adding another owner kind requires OSAC resource-context support. |
| spec.schema | Yes | Inline JSON Schema Draft 2020-12 object for the value. It may describe any JSON value. The CRD preserves this nested object, including `$schema`; that keyword is data inside `spec.schema`, not a top-level CRD field. The root `$schema` must be `https://json-schema.org/draft/2020-12/schema`; only same-document references and supported standard vocabularies are accepted. |

The NetworkDataModel name, meaning, owner scope, and schema are immutable. Provider-facing actions are Create, Read/List, and Delete; Update and Patch are rejected. A change to meaning, owner scope, or schema requires a new resource name. Runtime output artifacts remain separate owner-scoped ConfigMaps; registry objects contain definitions, never runtime values, credentials, or secrets.

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

Providers submit each schema inline in `NetworkDataModel.spec.schema`; this is content, not a schema URL or reference to a remote document.

The NetworkDataModel CRD declares `spec.schema` as `type: object` with `x-kubernetes-preserve-unknown-fields: true`. Kubernetes therefore preserves `$schema` as a nested key in the JSON object; it is not a NetworkDataModel or top-level CRD field. Kubernetes validates the outer resource shape but does not interpret JSON Schema keywords.

OSAC accepts only the exact Draft 2020-12 `$schema` dialect identifier, validates the submitted schema against a locally bundled meta-schema, and never fetches a schema or meta-schema over the network. All schema reference keywords, including `$ref` and `$dynamicRef`, must resolve to fragments in the same document. Remote and file references, schema-level `$id` values, unsupported vocabularies and owner scopes, malformed schemas, and duplicate NetworkDataModel names are rejected with an admission diagnostic naming the object and invalid field. Because schemas are provider input, OSAC enforces finite schema-size, nesting, and validation-work limits through the fulfillment-service API on Create. JSON Schema validates structure, types, and expressible constraints. It does not prove that a manager configured its backend correctly or that a value satisfies a relationship with other OSAC resources unless that relationship is encoded in a generic OSAC rule. Managers remain responsible for resource-relative semantics, and conformance verifies they honor the model description and shared networking behavior.

| NetworkDataModel name | Owner scope | JSON Schema assertions | Meaning and additional conformance |
|---|---|---|---|
| osac.networking.virtual-network.vxlan-l3-vni | VirtualNetwork | Integer from 1 through 16,777,215. | Identifies the VirtualNetwork Layer 3 VXLAN domain. It may be allocated during the first Subnet operation but remains owned by the VirtualNetwork. [RFC 7348](https://www.rfc-editor.org/rfc/rfc7348.html) |
| osac.networking.subnet.vxlan-l2-vni | Subnet | Integer from 1 through 16,777,215. | Identifies the Subnet's VXLAN Layer 2 segment. [RFC 7348](https://www.rfc-editor.org/rfc/rfc7348.html) |
| osac.networking.subnet.vlan-id | Subnet | Integer from 1 through 4094. | Identifies the Subnet's IEEE 802.1Q VLAN segment. [RFC 2674](https://www.rfc-editor.org/rfc/rfc2674.html) |
| osac.networking.subnet.reserved-ipv4-cidrs | Subnet | JSON array, possibly empty, whose items are strings. | Each value is an IPv4 CIDR reserved by the fabric and contained within the Subnet. Managers validate this resource-relative meaning; an overlay IP address manager excludes every listed range from workload allocation. |

A NetworkManager is a cluster-scoped OSAC API object that registers one Fabric or Kubernetes Manager implementation. It uses API version osac.openshift.io/v1alpha1. Its `spec.managerName` is the immutable logical name selected by NetworkClass; `metadata.name` is the separate DNS-safe Kubernetes object name. `spec.implementationRef` identifies the fully qualified Ansible collection role AAP invokes. The logical name and implementation reference are independent.

| Field | Required | Meaning |
|---|---|---|
| metadata.name | Yes | DNS-safe Kubernetes object name, unique across NetworkManager objects. It need not match the logical manager name. |
| spec.managerName | Yes | Immutable logical name referenced by NetworkClass; unique within its role. Existing names may contain underscores. |
| spec.role | Yes | Exactly Fabric or Kubernetes. |
| spec.implementationRef | Yes | Fully qualified Ansible collection role name, such as acme.networking.fabric_manager; the role must be installed in the AAP execution environment. |
| spec.networkOutputs | Fabric only | Required list, possibly empty, of model names this Fabric Manager can produce. |
| spec.networkInputs | Kubernetes only | Required list, possibly empty, of model names this Kubernetes Manager requires whenever the model owner is present for one of its assigned operations. |
| spec.description | No | Human-readable description for provider administration. |

A Fabric object must set networkOutputs and must not set networkInputs. A Kubernetes object must set networkInputs and must not set networkOutputs. Each listed name must identify an existing NetworkDataModel; duplicate and unknown names are rejected. An empty list means the manager has no declarations for that direction. The fulfillment-service API validates these fields and references before writing the backing custom resource; Kubernetes schema validation enforces field types, required fields, and immutability.

The pair (spec.role, spec.managerName) must be unique across NetworkManager objects. The fulfillment-service API rejects duplicate pairs when creating a manager so each NetworkClass reference resolves to exactly one registration. Kubernetes already requires metadata.name to be unique across all NetworkManager objects.

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
  networkInputs:
    - osac.networking.virtual-network.vxlan-l3-vni
    - osac.networking.subnet.vxlan-l2-vni
    - osac.networking.subnet.reserved-ipv4-cidrs
~~~

The selection sequence is NetworkDataModel objects, then NetworkManager objects, then NetworkClass. NetworkClass continues to store the existing logical manager names. When it is created, the fulfillment service resolves each name against cluster-scoped NetworkManager objects by role and `spec.managerName`, then checks that every NetworkDataModel name in the Kubernetes Manager's inputs appears in the selected Fabric Manager's outputs. IPv4 is the fixed address-family contract for all managers; no registration-time family negotiation is needed. The registry reader fails closed if the backing Kubernetes API cannot be read. An invalid or incompatible selection is rejected before NetworkClass is persisted; no tenant networking resource or AAP job can use it.

The comparison uses exact model names, not only JSON shape, because identity, meaning, and owner scope are part of the contract. A matching declaration establishes that the selected implementation advertises required data; it does not prove that either manager preserves the shared VirtualNetwork L3, Subnet L2, attachment, or SecurityGroup behavior. Each implementation and selected path must also pass conformance. OSAC does not contain a manager-name compatibility table.

| Fabric Manager | Kubernetes Manager | Result |
|---|---|---|
| netris outputs VXLAN L3 VNI, VXLAN L2 VNI, and reserved IPv4 CIDRs | cudn_evpn requires those same model names | Accepted |
| agentless_net outputs the Subnet VLAN ID | A Kubernetes Manager requires VXLAN VNIs and reserved IPv4 CIDRs | Rejected because the required model names are absent |
| agentless_net outputs the Subnet VLAN ID | A future LocalNet Manager requires the Subnet VLAN ID | Accepted by the data contract; this does not claim LocalNet support in the current CUDN proposal |

A provider-defined model uses the same path. The Fabric Manager lists its model name in networkOutputs, and the Kubernetes Manager lists the same NetworkDataModel name in networkInputs. OSAC places a produced value in an output artifact as model name plus JSON value, validates it against the registered schema, and passes the matching entry to the consumer. The consumer's Ansible role interprets the documented fields and maps them to its backend API. OSAC does not need a Go type or branch for a provider-defined model; the provider still supplies both managers' Ansible behavior and conformance evidence.

Manager and data model specs are immutable. Providers may Create, Read/List, and Delete these objects; OSAC rejects Update and Patch. A manager deletion is blocked while a NetworkClass selects its role and spec.managerName. A model deletion is blocked while any NetworkManager references its name or a retained owner-scoped output artifact contains a value under that name. To replace manager configuration, remove dependent tenant resources and the NetworkClass as required by the Unified Networking lifecycle, delete the old NetworkManager, then create the replacement. A change to a model's meaning, owner scope, or JSON Schema requires a new model name.

#### East-West capability boundary

NetworkManager registrations do not declare generic feature or IP-family capabilities. The provider's `NetworkClass.spec.east_west_capabilities` remains the explicit declaration of supported FabricDomain types; its behavior and validation are defined by the Unified Networking and Multi-Fabric East-West designs. Manager-pair validation uses the exact NetworkDataModel names required and produced by the selected roles.
### 4.3 API and Operation Contract

#### Provider registry API operations

The fulfillment service exposes `NetworkDataModels` and `NetworkManagers` services with `Create`, `List`, `Get`, and `Delete` operations. The request and response objects use the same fields defined in Section 4.2. Only Cloud Infrastructure Admins may call these services; tenant users cannot read or modify provider registrations. The CRDs are the operator-visible backing representation, not a second registry. OSAC installation may create built-in registry objects directly; provider configuration uses the fulfillment-service API.

| Resource | Create | List/Get | Delete | Update/Patch |
|---|---|---|---|---|
| NetworkDataModel | Validate name, scope, description, and schema, then persist the CRD. | Return the stored definition. | Reject while a manager or retained output value references it. | Rejected. |
| NetworkManager | Validate role, implementation reference, and model references, then persist the CRD. | Return the stored registration. | Reject while a NetworkClass selects it. | Rejected. |

The fulfillment service also checks that manager names are unique within their role. Kubernetes CRD schemas enforce field shape and immutability on the backing objects. If a registry read or write fails, the service returns an error and does not report the operation as successful.

These provider APIs manage registry definitions. The existing NetworkClass API selects managers by logical name. The operation and target table below describes AAP backend reconciliation, not fulfillment-service CRUD operations.

#### Resource-scoped network outputs and inputs

A Fabric Manager publishes its declared values through a durable, OSAC-owned ConfigMap in the networking hub namespace. OSAC creates each target ConfigMap before dispatch and supplies `network_output_targets`; each target identifies the owning resource kind and UID, the namespace, the OSAC-generated ConfigMap name, and the model names required at that scope for the operation. OSAC derives those model names from the selected Kubernetes Manager's `networkInputs`, the model owner scopes available in the operation context, and values not already published. Thus a provider-defined model uses the same target generation and validation path as a model shipped by OSAC. There is one output artifact per owner resource UID, not one per manager pair or operation. The Fabric task writes the JSON document only to the targets OSAC provides and does not change tenant resource status.

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
        {"model_name": "osac.networking.subnet.vxlan-l2-vni", "value": 40120},
        {"model_name": "osac.networking.subnet.reserved-ipv4-cidrs", "value": ["192.0.2.0/28", "192.0.2.32/27"]}
      ]
    }
```

The resource kind and UID in the document must match the `network_output_targets` entry. Each `model_name` must be declared by the Fabric Manager, listed in that target's `model_names`, and have an `ownerScope` matching the target resource kind. For a specific operation, `model_names` lists the values required for that resource target; the Fabric task must publish each listed value. OSAC validates the JSON structure, model name, owner scope, and value against the registered JSON Schema after the Fabric job succeeds. Unknown, duplicate, missing-required, wrong-scope, or schema-invalid outputs fail the operation and prevent the Kubernetes stage from starting. The producer may publish one parent-VirtualNetwork artifact and one Subnet artifact during `subnet.create`; a VNI allocated during this operation is still stored under the VirtualNetwork UID when the VirtualNetwork owns it.

OSAC serializes writes to the same owner artifact. After validation, it merges newly requested model values by NetworkDataModel name and preserves other valid entries already stored for that owner. A manager cannot replace a value for the same model name and owner UID during that owner's lifetime. Retries and later operations reuse the stored value; deleting an owner removes its values only after dependent cleanup succeeds.

Before dispatching a Kubernetes operation, OSAC resolves the selected manager's `networkInputs` from the operation resource and its available context. An input applies when its model's `ownerScope` is represented in that context; OSAC requires exactly one valid value for each applicable owner and fails before dispatch if a value is missing or invalid. It then passes the normalized entries inside `osac_job_vars.network_inputs`. A Subnet operation may receive values owned by the Subnet, its parent VirtualNetwork, and its NetworkClass. A workload-attachment operation may receive values owned by the attached Subnet, its parent VirtualNetwork, and the NetworkClass. The Fabric Manager's implementation details and credentials are not passed to the Kubernetes Manager. For example:

```yaml
osac_job_vars:
  network_inputs:
    - model_name: osac.networking.virtual-network.vxlan-l3-vni
      resource_kind: VirtualNetwork
      resource_uid: "<virtual-network-uid>"
      value: 4020
    - model_name: osac.networking.subnet.vxlan-l2-vni
      resource_kind: Subnet
      resource_uid: "<subnet-uid>"
      value: 40120
    - model_name: osac.networking.subnet.reserved-ipv4-cidrs
      resource_kind: Subnet
      resource_uid: "<subnet-uid>"
      value: ["192.0.2.0/28", "192.0.2.32/27"]
```

The Kubernetes Manager receives only values whose model names it declared and whose owner scopes apply to the operation. It validates and translates them into its native API fields; it must not allocate a replacement value or infer one model from another. Output artifacts remain available while any dependent create or delete operation may consume them. An artifact is removed only after its owner is deleted and all dependent manager cleanup that could consume it has succeeded. This applies to NetworkClass-scoped profile data as well as resource-scoped values.

#### Common AAP input

OSAC passes a common job envelope in the `osac_job_vars` variable. The task-specific resource is the OSAC resource being reconciled. The `manager.name` field carries the selected NetworkManager's spec.managerName; metadata.name identifies the Kubernetes object only. The manager reference selects the collection role, while the operation names one fixed task from the operation table. `manager.role` is normalized by OSAC to exactly `fabric` or `kubernetes`; a collection cannot choose or override it. Fabric tasks receive `network_output_targets` for required, not-yet-published model values whose owners are available in the operation context. Kubernetes tasks receive applicable values for model names in their registration's `networkInputs`; that list is empty when the selected manager declares no cross-manager data dependency or no declared owner scope applies to the operation.

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

For example, when CUDN declares the three model names below as inputs and none of these values has already been published, the Fabric `subnet.create` task receives these output targets. OSAC computes the target list from the model catalog and operation context; the manager registration does not hard-code an operation-to-model mapping. A paired Kubernetes task receives the `network_inputs` entries shown above:

```yaml
osac_job_vars:
  operation: subnet.create
  network_output_targets:
    - namespace: "<networking-hub-namespace>"
      resource_kind: VirtualNetwork
      resource_uid: "<virtual-network-uid>"
      config_map_name: network-output-<virtual-network-uid>
      model_names: [osac.networking.virtual-network.vxlan-l3-vni]
    - namespace: "<networking-hub-namespace>"
      resource_kind: Subnet
      resource_uid: "<subnet-uid>"
      config_map_name: network-output-<subnet-uid>
      model_names:
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
|-----------|-------------------------|-----------------|-------------------------|------------------------------|
| `virtual_network.create` / `virtual_network.delete` | Fabric | N/A | `create_virtual_network` / `delete_virtual_network` | Create or remove the isolated VirtualNetwork routing domain and associated allocation. Subnets in the same VirtualNetwork are L3-routable to one another; separate VirtualNetworks remain isolated. |
| `subnet.create` / `subnet.delete` | Create: Fabric, then Kubernetes when configured. Delete: Kubernetes, then Fabric. | N/A | `create_subnet` / `delete_subnet` | Fabric creates/removes one L2 broadcast domain for the Subnet and publishes declared output values into OSAC-provided artifacts for their owning resource scopes. Kubernetes receives only its declared, matched inputs, which may include the parent VirtualNetwork and current Subnet. Workloads in one Subnet share that L2 domain and are L3-routable within their parent VirtualNetwork. OSAC removes Subnet outputs after successful consumer cleanup and Fabric deletion; parent-VirtualNetwork outputs persist through individual Subnet deletion. The Subnet CIDR belongs to its parent VirtualNetwork and cannot overlap a sibling Subnet. |
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

`operation`, `resourceUID`, and `observedGeneration` must match the operation OSAC dispatched and the resource version it dispatched. `data` is an empty object; operation-specific outputs use their separately defined channels. ExternalIP allocation reports its address through the guarded `osac.openshift.io/allocated-address` annotation, not in `data`. Network data outputs use the separate resource-scoped ConfigMaps defined above, not this envelope. `dhcp_lease.query` returns the `leases` artifact instead of `osac_result`. The envelope has no version field; changes to its fields or result data require a coordinated OSAC and manager contract update.

OSAC validates the artifact name and all required fields before accepting task success. A missing, malformed, stale, or mismatched envelope is a failed operation and cannot advance resource readiness or release capacity. A manager must not report a successful operation with an envelope for a different operation, resource UID, or generation.

Ansible supports role inclusion by a variable role name and the `tasks_from` selector. The fully qualified collection role must be installed in the AAP execution environment. See [Ansible Core include_role documentation](https://docs.ansible.com/projects/ansible-core/2.17/collections/ansible/builtin/include_role_module.html) and [using collection roles by FQCN](https://docs.ansible.com/projects/ansible/latest/collections_guide/collections_using_playbooks.html). [Research: §1]

### 4.4 Subnet Orchestration and Network Data Lifecycle

For create, OSAC runs Fabric first, validates each requested output against its registered model, then resolves the exact values required by the selected Kubernetes Manager. A Subnet operation may publish the parent's VirtualNetwork or NetworkClass output even when that value is first allocated while creating the Subnet. During delete, OSAC keeps those outputs available for Kubernetes cleanup and invokes Fabric deletion only after that cleanup succeeds. Each value remains until its owner and all dependent resources that can consume it have been deleted.

```mermaid
sequenceDiagram
    participant NC as NetworkClass
    participant Operator as OSAC operator
    participant AAP as AAP
    participant Fabric as Fabric Manager
    participant K8s as Kubernetes Manager

    NC->>Operator: Select manager registrations
    Operator->>Operator: Match K8s networkInputs against Fabric networkOutputs
    Operator->>AAP: subnet.create for Fabric with resource-scoped output targets
    AAP->>Fabric: create_subnet(resource, network_output_targets)
    Fabric->>Fabric: Publish schema-valid outputs under owner UIDs
    AAP-->>Operator: Successful Fabric job
    Operator->>Operator: Validate model names, owner scopes, and JSON Schemas
    Operator->>Operator: Resolve applicable K8s-declared inputs from operation context
    Operator->>AAP: subnet.create for Kubernetes with network_inputs
    AAP->>K8s: create_subnet(resource, network_inputs)
    K8s-->>AAP: Converged result
    AAP-->>Operator: Successful Kubernetes job
    Operator->>Operator: Mark Subnet Ready after both roles succeed

    Operator->>Operator: Retain both scope artifacts during Kubernetes cleanup
    Operator->>AAP: subnet.delete for Kubernetes with network_inputs
    AAP->>K8s: delete_subnet(resource, network_inputs)
    K8s-->>AAP: Detached result
    AAP-->>Operator: Successful Kubernetes job
    Operator->>AAP: subnet.delete for Fabric
    AAP->>Fabric: delete_subnet(resource)
    Fabric-->>AAP: Removed result
    AAP-->>Operator: Successful Fabric job
    Operator->>Operator: Remove owner artifacts after dependent cleanup
    Operator->>Operator: Complete Subnet deletion
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

The current OSAC implementation does not yet provide the proposed generic model catalog, schema validator, owner-scoped output resolver, or input envelope. Generic lifecycle support must validate Fabric output artifacts against registered schemas before Kubernetes dispatch, preserve each scope through retries, and order consumer cleanup before the owner removes its outputs. Each manager task remains independently implemented and idempotent; OSAC owns matching, validation, ordering, and retries.

### 4.5 Scalability and Performance

NetworkManager and NetworkDataModel objects contain small declarations and schemas that OSAC reads during admission and NetworkClass validation. Schemas are loaded once and reused to validate runtime values. Runtime handoff values remain in small owner-scoped ConfigMaps; the registry adds no backend database. Subnet provisioning adds an ordered AAP stage when a Kubernetes Manager is configured, so readiness waits for both jobs to finish.

### 4.6 Security Considerations

NetworkManager and NetworkDataModel objects contain implementation references, declarations, and schemas, not credentials. Cluster-scoped RBAC limits their administration to Cloud Infrastructure Admins. The fulfillment service has the registry access needed to serve its provider APIs; the operator reads the registry for manager resolution and dispatch. AAP jobs cannot modify registry objects. A Fabric AAP job receives a separate, job-scoped credential that permits updates only to the output ConfigMaps pre-created for that operation; provider backend credentials remain separate. Output targets are generated by OSAC, and managers cannot choose another namespace, name, resource scope, or model name. OSAC validates the published resource kind and UID against the target and validates each value against the registered schema before accepting outputs. The Kubernetes Manager receives validated values, not the Fabric writer credential. Model values are JSON data and must not contain credentials or secrets. Ansible job artifacts and logs must not disclose secrets. Manager tasks act only on the tenant-scoped resources OSAC passes and must preserve existing tenant and owner-reference boundaries.

### 4.7 Failure Handling and Recovery

- **Invalid registry object:** The fulfillment-service API rejects a NetworkDataModel or NetworkManager with a diagnostic naming the object and invalid field or model name. Kubernetes enforces the outer CRD shape and preserves the raw schema object; the fulfillment service validates its JSON Schema semantics.
- **Invalid NetworkClass selection:** Fulfillment-service validation rejects unresolved manager names, role mismatches, unsatisfied Kubernetes inputs before persisting the profile.
- **Registry lookup failure:** NetworkClass creation fails closed if OSAC cannot read the referenced NetworkManager objects. It does not persist the profile or start provider work.
- **Unmatched manager input:** NetworkClass creation reports the model name required by the Kubernetes Manager but absent from the Fabric Manager outputs; no NetworkClass is persisted and no AAP job can start.
- **Missing collection role or task:** AAP fails with the missing fully qualified collection role or task name. The diagnostic names the registration or task.
- **Missing or malformed Fabric outputs:** A missing artifact, unknown or duplicate model name, wrong owner scope, or value that fails the registered JSON Schema prevents the Kubernetes stage from starting. OSAC reports the invalid model name and owner UID and retries or fails through the existing provisioning lifecycle.
- **Kubernetes create failure after Fabric success:** OSAC retains all output artifacts and retries the Kubernetes stage using the same values. Fabric create is idempotent. The Subnet is not Ready until the Kubernetes stage succeeds.
- **Kubernetes delete failure:** OSAC does not start Fabric deletion, so Subnet and parent-VirtualNetwork outputs remain available while Kubernetes cleanup retries.
- **Fabric delete failure after Kubernetes detach:** OSAC retains job state and retries Fabric cleanup. The Kubernetes target is already detached; repeated Fabric deletion of absent state succeeds.
- **Backend timeout or transient error:** The AAP task returns a diagnostic and a non-success result. Existing OSAC job retry/backoff behavior retries the operation. All create/apply and delete tasks are idempotent by resource UID.
- **Workload attachment policy apply failure:** The manager keeps traffic unavailable; OSAC does not report the workload Ready and retries the same normalized attachment. Policy delete failure retains the workload finalizer and blocks teardown until cleanup succeeds.
- **Invalid operation result:** A missing, malformed, stale, or mismatched result fails the operation; OSAC does not advance readiness or release capacity.
- **Invalid ExternalIP or lease result:** Missing allocated-address output or malformed/ambiguous lease output fails the task; OSAC does not report allocation or lease discovery as successful.

### 4.8 RBAC and Tenancy

No tenant-facing RBAC changes are required. The fulfillment service continues to authorize tenant networking requests and needs write access to persist or delete registry objects through its provider APIs. Its NetworkClass preflight reads selected registrations without modifying them. The operator reads registry objects for manager resolution and dispatch, while the AAP writer credential is scoped only to OSAC-created output ConfigMaps. Managers receive only the resource context authorized for the requested tenant and must not read or modify resources belonging to another tenant.

### 4.9 Extensibility and Future-Proofing

A new manager is onboarded by creating any provider-defined NetworkDataModel objects first, creating a NetworkManager object that references those models, installing its collection into the AAP execution environment, and selecting the manager in NetworkClass. After OSAC ships the generic CRDs, admission validation, registry reader, JSON Schema validator, owner-scope resolver, and manager contract, adding a model or manager requires provider objects and Ansible content only; it does not require manager-specific Go changes. Adding an OSAC resource kind, manager role, operation, workload target, or data handoff transport requires platform support. A model name, meaning, scope, and schema remain a provider-level contract and do not require a new OSAC manager integration.

### 4.10 Risks and Mitigations

- **Matching declarations may not preserve the shared network and policy semantics.** Treat input/output matching as necessary but not sufficient; require each manager and selected network path to pass conformance for VirtualNetwork L3 routing, Subnet L2 broadcast behavior, and workload attachment policy.
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

### IC-1: Immutable manager and model registry APIs

**Requirements:** FR-1, FR-2, FR-3, FR-5, FR-6

OSAC adds cluster-scoped NetworkDataModel and NetworkManager API objects. Model schemas and scopes are validated when NetworkDataModel objects are created; manager role declarations and references to existing models are validated when NetworkManager objects are created. Provider-facing operations are Create, Read/List, and Delete only; Update and Patch are rejected. Deletes are blocked while dependents reference the object. Section 4.2 defines the object fields, admission rules, examples, and dependency order.

### IC-2: Generic AAP task invocation

**Requirements:** FR-1, FR-2, FR-3

AAP playbooks resolve the role from the selected NetworkManager object's fully qualified implementation reference and pass the fixed operation envelope in Section 4.3. Each manager collection provides the complete task set for its role.

### IC-3: Network data matching and fixed role dispatch

**Requirements:** FR-2, FR-3, FR-4, FR-6

NetworkClass creation requires a Fabric Manager and validates that its declared outputs cover every input declared by a selected Kubernetes Manager. An invalid selection is rejected before NetworkClass persistence with a diagnostic naming unresolved references or unsatisfied model names. OSAC routes each operation only to its contract-assigned role.

### IC-4: Operation results and retry behavior

**Requirements:** FR-2

Manager tasks follow the operation, target, artifact, idempotency, and error rules in Section 4.3 and Section 4.7, including ExternalIP allocation and DHCP lease results.

### IC-5: Resource-scoped Fabric-to-Kubernetes data exchange

**Requirements:** FR-1, FR-2, FR-3, FR-4

The Fabric Manager publishes JSON values under the UIDs of their registered owner resources. OSAC validates each value against its catalog schema and owner scope, then passes only applicable Kubernetes Manager inputs on dependent operations. Section 4.3 defines the artifact and AAP schemas; Section 4.4 defines their lifetime and ordering.

### IC-6: Workload attachment and SecurityGroup enforcement

**Requirements:** FR-2, FR-3

OSAC sends the resolved workload attachment and complete SecurityGroup rules to the manager that owns its interface. The Kubernetes Manager handles ComputeInstance overlays; the Fabric Manager handles BaremetalInstance and CaaS worker interfaces. Apply and delete ordering, payload fields, stateful rule semantics, and failure behavior are specified in Sections 4.3 and 4.4.

### IC-7: Provider-installed network data model registry

**Requirements:** FR-5, FR-6

Providers register stable model names, meanings, supported owner scopes, and JSON Schemas as NetworkDataModel objects. OSAC validates registry entries and runtime values through the generic model mechanism, independent of manager names or backend implementation. A provider-defined model may use any JSON value shape supported by its schema and existing OSAC resource context.

## 6. Alternatives Considered

### Fetch the schema from a URL

A URL keeps the NetworkDataModel small but makes schema admission and value validation depend on an external service. Fetching a provider-controlled URL would also give an OSAC component a path to cluster-internal or otherwise restricted endpoints, and remote content could change without changing the immutable registration. OSAC therefore accepts schema content inline and performs no network or filesystem resolution.

### Reference a separate ConfigMap

A ConfigMap reference avoids outbound URL fetching, but splits one immutable model definition across two objects and introduces another lookup, permission, and lifecycle dependency. Inline schema content keeps the name, meaning, scope, and validation rules together in the NetworkDataModel that OSAC validates.

### Keep a hard-coded manager-pair table in Go

A central pair table gives OSAC direct control over every combination, but requires a Go change whenever a provider adds a manager or a new pairing. Matching input and output model names from the provider-installed catalog lets the generic validator assess new implementations without a vendor-specific pair table.

### Let managers call each other directly

Direct calls can pass segment data without OSAC persistence. They couple managers to one another's APIs and credentials, bypass OSAC job tracking, and make retries and partial failure depend on vendor-specific coordination. OSAC-owned sequencing preserves separate manager implementations and one audited handoff.

### Keep the manager jobs independent

Independent jobs cannot guarantee that a Kubernetes Manager receives values published by Fabric, and they can report partial readiness. The ordered create and delete stages provide the dependency and retain each resource-scoped output artifact for retries and cleanup.

### Derive the implementation from the logical manager name

Name-derived collection roles match the current built-in role convention. An independently maintained implementation would need to extend or replace the OSAC collection. An explicit fully qualified role reference keeps the selection source-neutral while operation names and payloads remain OSAC-defined.

### Pass an opaque vendor-specific artifact

An opaque vendor map would let each Fabric Manager return its own schema, but the Kubernetes Manager and OSAC could not validate the data before use. Registered model names, JSON Schemas, and owner scopes let providers extend the data vocabulary while keeping validation generic and backend-only identifiers inside the Fabric Manager.

## 7. Observability and Monitoring

No new metrics are required. Existing NetworkClass state/message, resource conditions, events, AAP job history, and reconciliation logs report registry admission, profile validation, operation, and backend errors. Diagnostics identify the NetworkManager or NetworkDataModel object, manager role and logical name, operation stage, model name, owner resource UID, and schema or scope failure. Logs do not include credential values.

## 8. Impact and Compatibility

This is the target contract; the current operator and AAP implementation do not yet enforce all of it. Generic OSAC implementation work adds the NetworkDataModel and NetworkManager CRDs, provider-facing fulfillment-service APIs, write and read validation, read-only registry resolution for NetworkClass creation, dependency-safe deletion, JSON Schema validation, owner-scope resolution, output publication and input resolution, generic collection-role invocation, and output lifetime handling. Existing ConfigMap registrations must be migrated to equivalent NetworkManager objects before ConfigMap discovery is removed. Once this generic support ships, a provider can add models and conforming managers through registry objects plus Ansible content; no manager-specific Go changes are required.

During rollout, install the NetworkDataModel and NetworkManager CRDs and built-in model objects, convert existing model and manager ConfigMaps to their API-object forms, and verify each NetworkClass passes preflight validation before disabling ConfigMap discovery. Existing flat Subnet output must also migrate to the schema-validated, owner-scoped format. The manager contract adds no tenant-facing resource or API fields. An unmatched required input or invalid runtime value prevents dependent dispatch with a diagnostic.

Changing manager names in NetworkClass does not automatically migrate backend state for existing resources. Providers must follow an explicit migration or resource replacement procedure before switching manager assignments. This contract guarantees common API behavior for newly reconciled resources, not transparent state transfer between different backends.

---

## Provenance

Authored: draft @ design 0.11.3 - 2bd6607, workspace main @ 1f3b63b82 (52 behind origin/main)
Final: revise @ design 0.11.3 - 2bd6607, workspace main @ 1f3b63b82 (99 behind origin/main, dirty)

> Context changed between draft and revise.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.3","ai_workflows":"2bd6607","source_repo":"1f3b63b82 (dirty)","source_repo_branch":"main","commits_behind_main":99,"commits_ahead_main":0,"main_ref":"main","phases":["draft","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
