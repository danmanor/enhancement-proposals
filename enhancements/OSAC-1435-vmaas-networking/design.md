---
title: vmaas-networking
authors:
  - dmanor@redhat.com
creation-date: 2026-07-08
last-updated: 2026-10-12
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-1435
prd: "prd.md"
see-also:
  - "Unified Networking: /enhancements/OSAC-1433-unified-networking"
  - "Default Networking: /enhancements/OSAC-1433-default-networking"
  - "Network Manager Integration Contract: /enhancements/OSAC-1433-network-manager-integration-contract-networking/design.md"
replaces:
  - N/A
superseded-by:
  - N/A
---

# VMaaS Networking — Optional Attachments and Auto External Access

## Contents

- [Summary](#summary)
- [Motivation](#motivation)
- [Proposal](#proposal)
  - [Working with provider-supplied managers](#working-with-provider-supplied-managers)
  - [VM placement and interface ownership](#vm-placement-and-interface-ownership)
  - [Workflow Description](#workflow-description)
  - [API contract](#api-contract)
  - [Component behavior and constraints](#component-behavior-and-constraints)
- [Security Considerations](#security-considerations)
- [Failure Handling and Recovery](#failure-handling-and-recovery)
- [RBAC and Tenancy](#rbac-and-tenancy)
- [Observability and Monitoring](#observability-and-monitoring)
- [Risks and Mitigations](#risks-and-mitigations)
- [Drawbacks](#drawbacks)
- [Provider Networking Disabled](#provider-networking-disabled)
- [Alternatives Considered](#alternatives-considered)
- [Open Questions](#open-questions)
- [Test Plan](#test-plan)
- [Graduation Criteria](#graduation-criteria)
- [Upgrade and Downgrade Strategy](#upgrade-and-downgrade-strategy)
- [Version Skew Strategy](#version-skew-strategy)
- [Support Procedures](#support-procedures)
- [Infrastructure Needed](#infrastructure-needed)

This enhancement extends the unified networking API to support VMaaS-specific requirements: a single ComputeInstance network attachment with an implicit primary/default route, optional network attachments with tenant defaults, and auto-provisioned external access (ExternalIP). The repeated attachment field is retained for API compatibility and is validated to contain at most one entry.

## Summary

This enhancement is a service-specific extension of the [Unified Networking EP](/enhancements/OSAC-1433-unified-networking/design.md), providing the detailed per-service flow for this service type. The Unified Networking EP defines NetworkClass and the shared networking resource semantics and hierarchy. The Network Manager Integration Contract defines manager registration and dispatch. This document defines how VMaaS consumes those shared contracts.

VMaaS inherits the [Unified Networking deployment support
boundary](/enhancements/OSAC-1433-unified-networking/design.md#deployment-support-boundary):
VM networking supports connected deployments only and does not add air-gapped
or disconnected networking support.

VMaaS networking also inherits the [Unified Networking hub support
boundary](/enhancements/OSAC-1433-unified-networking/design.md#networking-hub-support-boundary):
OSAC networking supports exactly one provider-owned hub per deployment.
Multi-hub networking placement, cross-hub resource coordination, and
cross-hub network connectivity are unsupported. This boundary applies only to
the networking area and does not define hub behavior for other OSAC areas.
Multiple hosting/workload clusters remain supported where a networking feature
explicitly specifies them.

ComputeInstance currently uses a `ComputeNetworkAttachment` message. This enhancement keeps the existing repeated attachment field optional (populating it with tenant defaults when omitted), enforces a maximum of one entry, and adds `auto_external_ip_attachment` to enable fully connected VMs in a single API call. VMaaS has no primary field: the sole attachment is implicitly the default route. See [PRD](prd.md) for detailed requirements.

The manager-backed resource flows and provider IP-discovery/readiness behavior
in this document describe the enabled mode. The feature-gated disabled behavior
is specified at the end of the Proposal section. API-level SecurityGroup rule
validation and reference checks remain active in disabled mode. OSAC submits no
provider operation to apply or remove SecurityGroup rules; rules already
programmed in the backend may continue to affect traffic until provider-side
cleanup. [User]

The [Network Manager Integration Contract](/enhancements/OSAC-1433-network-manager-integration-contract-networking/design.md)
owns registration, task/input/result schemas, no-op requirements, overlapping
address spaces, stable API ownership, NetworkData, retries and shared cleanup.
The [internal request protocol](/enhancements/OSAC-1433-network-manager-integration-contract-networking/design.md#workload-attachment-policy-lifecycle)
defines attachment status and projection recovery. This design specifies the
service's handoffs and readiness gates, not another copy of those rules.

## Motivation

A VM needs one tenant network attachment whose provider-specific interface
and policy are established before it starts. OSAC resolves tenant intent into
a stopped VM, delegates its networking to the selected Kubernetes Manager,
and publishes the discovered address through the workload API. Optional
external access follows the shared allocation and attachment lifecycle.

### Goals

- Single-NIC support with an implicit primary attachment for the default gateway
- Resource-specific attachment message (`ComputeNetworkAttachment`); the sole attachment is implicitly primary
- Optional `network_attachments` field — populate with tenant defaults when omitted
- Auto ExternalIP attachment (`auto_external_ip_attachment`) for single-call inbound connectivity
- BM-only deployment validation to reject VM provisioning when no k8s_manager is available

### Non-Goals

- CaaS or BMaaS networking (this EP covers VMaaS only)
- Implementation of the shared manager dispatch contract, which is defined by the Network Manager Integration Contract.
- Building or selecting a particular Kubernetes Manager implementation; VMaaS uses the deployment-selected manager that conforms to the shared contract.

## Proposal

### Working with provider-supplied managers

The deployment selects its Fabric and Kubernetes Managers through NetworkClass.
Neither the tenant nor the VM template chooses a binding plugin or provider.
The same workload template works with any conforming selected Kubernetes
Manager: it prepares compute/disks, and that manager supplies network fields.

| Phase | VMaaS responsibility | Provider-manager responsibility |
|---|---|---|
| Prerequisites | Require a ready Subnet placement under the unified lifecycle. | Fabric creates the network; Kubernetes realizes that placement using stored Fabric outputs. |
| Prepare | Create the stopped VM and capture its backend UID. | No VM attachment action yet. |
| Attach | Create the internal request with the exact placement and target identity. | Kubernetes apply configures the primary interface and policy. |
| Start and observe | Start only after accepted attachment success; mirror real VMI IP feedback. | Preserve the interface/policy and expose its address through the agreed VMI observation. |
| Delete | Complete external cleanup, stop the VM, wait for attachment cleanup, then remove compute/disks. | Kubernetes removes only the owned attachment; shared Subnet cleanup waits for all consumers. |

For example, one manager may configure the stopped VM to use its CUDN binding;
another may configure a different supported KubeVirt network binding. OSAC
does not build either provider's interface configuration. Both use the same
`apply_workload_attachment` entry point and result gate. The exact field
ownership and replay rules follow below. [User]

### VM placement and interface ownership

OSAC owns placement; the Kubernetes Manager owns backend network realization.
For each Subnet/hosting-cluster placement, OSAC sets `placement.tenant_namespace` to
`osac-subnet-<Subnet API UUID>` and persists it with the placement snapshot
before Subnet dispatch. Kubernetes `create_subnet` creates or verifies that
namespace and its networking. It preserves the tenant annotation and exact
Subnet owner API ID; an existing namespace with a different owner is an error.
The manager may choose its own CUDN/NAD/backend object names, but cannot move
the workload to another namespace or return a replacement placement.
Deletion removes only its owned realization/namespace after all consumers
are gone. Names do not replace the existing stable API ownership and concrete backend-object UID checks. [User]

The workload controller chooses the hosting cluster from the Subnet's ready
placements and persists `placement.workload_selector={kind: VirtualMachine,
name: osac-vm-<ComputeInstance API UUID>}` and `interface_selector: primary`.
The workload template creates the VM and disks in that namespace with
`spec.runStrategy: Halted`, automatic pod-interface attachment disabled, and
no provider-specific `networks` or `interfaces` entries. It confirms no VMI
is running and captures the created VM UID as
`placement.workload_selector.uid` before attachment dispatch. Preparation
retries preserve any already configured manager-owned fields.
[KubeVirt interfaces](https://kubevirt.io/user-guide/network/interfaces_and_networks/),
[run strategies](https://kubevirt.io/user-guide/compute/run_strategies/).

Kubernetes `apply_workload_attachment` verifies that VM UID. Initial
configuration requires a halted VM with no running VMI. An identical replay
after start verifies the existing interface and policy without rewriting live
network fields; drift that requires changing the interface fails rather than
stopping or mutating the running workload. Live attachment updates require a
separately defined update lifecycle. The manager owns `spec.template.spec.networks`,
`spec.template.spec.domain.devices.interfaces`, and
`spec.template.spec.domain.devices.autoattachPodInterface` (kept false).
It configures exactly one matching network/interface pair named `primary`
and installs the complete policy before reporting success. These fields are
patched without replacing disks, compute settings, run strategy, or unrelated
metadata. Provider-specific labels/annotations used by networking must use a
provider-owned prefix and must not overwrite OSAC tenant/ownership metadata.
Each role documents its keys; preparation/start/reconcile templates preserve
them. The adapter interprets its existing Fabric-produced NetworkData inputs
and its own configuration; it produces no Kubernetes-owned NetworkData.

After accepted attachment success, the workload controller verifies the
same VM UID and single-interface shape, then changes only the workload-owned
run control to start it. Subsequent workload reconciliation must not reapply
a full stale VM manifest that overwrites the network fields. The selected
adapter must expose the real `primary` interface address through VMI status;
generic discovery does not depend on a CUDN/NAD name or binding plugin.
Deletion confirms no VMI remains, invokes attachment cleanup with the same
VM UID, and removes the stopped VM only after cleanup. A missing/replaced VM
is not permission to patch a same-name object; cleanup uses captured identity
and provider mappings. Backend object loss follows the existing failed-stage
recovery rules. These are fixed responsibilities of the existing apply/delete
tasks, not another output channel or lifecycle. [User]


### Workflow Description

#### Networking prerequisites

Use the shared [creation readiness gates](/enhancements/OSAC-1433-unified-networking/design.md#creation-readiness-gates) and [deletion dependency guards](/enhancements/OSAC-1433-unified-networking/design.md#deletion-dependency-guards), together with [tenant defaults](/enhancements/OSAC-1433-default-networking/design.md).
Before VM creation, the selected Subnet, its SecurityGroups and required
hosting placement must be ready. Fabric values are committed before Kubernetes
realization; VMaaS does not allocate or reinterpret them.

#### VM Creation

1. **Tenant creates ComputeInstance:**
   ```bash
   # Explicit networking:
   osac create computeinstance --template ocp_virt_vm \
     --network-attachment subnet=my-subnet,security-groups=my-sg \
     --name my-vm

   # Or with defaults + auto external access:
   osac create computeinstance --template ocp_virt_vm \
     --external-ip-attachment --name my-vm
   ```
   - fulfillment-service:
     - If `network_attachments` is omitted or empty: populates the sole attachment with the tenant's default Subnet and default SecurityGroup (see Default Networking PRD)
     - If one attachment is supplied, defaults only missing fields: a missing Subnet receives the tenant default Subnet, and a missing or empty SecurityGroup list receives the tenant default SecurityGroup only when the resolved Subnet belongs to the tenant's default VirtualNetwork; otherwise the caller must provide SecurityGroups from the resolved Subnet's VirtualNetwork; supplied values are preserved
     - Validates: at most one attachment; the subnet is Ready and the security groups belong to the same VN
     - If `auto_external_ip_attachment == true`: auto-selects ExternalIPPool (READY, most available capacity), creates ExternalIP in the same DB transaction as the ComputeInstance — ExternalIP starts in **Pending** state. Pool capacity is decremented atomically; if the pool is exhausted, the API call fails and no resources are persisted. ExternalIPAttachment is **not** created at this point — it is deferred to the fulfillment-service internal reconciler, which creates it only after the ExternalIP is Allocated and the ComputeInstance is Ready. See [Unified Networking — Auto-provisioning lifecycle](/enhancements/OSAC-1433-unified-networking/design.md#auto-provisioned-external-access) for the full stepped flow.
   - Creates ComputeInstance CR with `network_attachments`

2. **Prepare a stopped VM and apply attachment policy:**
   - The ComputeInstance lifecycle controller resolves and persists the selected hosting cluster, Subnet overlay namespace, deterministic VM name and interface/selector identity. The ordinary AAP template prepares its DataVolume and `osac-vm-<ComputeInstance API UUID>` VM with `runStrategy: Halted`, automatic pod-interface attachment disabled, and no provider-specific network/interface entries. It records the VM UID before attachment dispatch; no VMI runs or tenant traffic is enabled.
   - After registering its workload cleanup finalizer, the controller creates one target-only SubnetAttachment. The networking controller resolves the workload's immutable attachment, ready same-VirtualNetwork Subnet/SecurityGroup references, stateful rules, applicable NetworkData and the exact stopped-VM placement.
   - It dispatches Kubernetes `apply_workload_attachment` with request `resource`, ComputeInstance `target`, and normalized `network_attachment`. Request UID/generation identify the result; the ComputeInstance API ID, logical interface and selected placement own backend state. The captured placement includes the actual hosting cluster, namespace, VM name and selector so the adapter can operate before a running VMI exists.
   - The manager verifies that exact stopped VM UID, configures its one `primary` network/interface pair and complete stateful policy. It owns the network/interface fields defined by the shared contract; the workload template preserves those fields. It must support this lifecycle and may not require a running VMI to apply policy. Apply failure keeps the VM stopped and the request non-ready; `AttachmentReady` follows confirmed apply/result persistence. [User]

3. **Start the prepared VM (`osac.templates.ocp_virt_vm`):**
   - The lifecycle controller resumes ordinary template execution only after `AttachmentReady`. It starts the prepared KubeVirt VM with its one interface in the resolved overlay namespace.
   - DHCP supplies the real address, gateway, prefix and DNS. Workload labels identify the selected interface; labels alone do not establish policy success.
   - Workload provisioning and networking jobs remain separately tracked. VM Ready requires successful attachment policy and real IP discovery. [User]

#### IP Discovery (feedback loop)

4. **SubnetAttachment controller discovers the VM IP:**
   - Watches KubeVirt VMI network status and maps the sole tenant interface to the resolved attachment.
   - Writes the assigned IPv4 address and `IPDiscoveryComplete` to request status.
   - The ComputeInstance lifecycle controller mirrors the request's readiness/IP to `computeNetworkAttachmentStatuses` on workload status.
   - The feedback controller transports that status through Signal RPC to fulfillment-service; fulfillment persists `compute_network_attachment_statuses` for tenant visibility and ExternalIPAttachment target resolution.
   - The request owns manager job history and discovery status; the workload API remains the sole source of desired attachment intent. [User]

#### External Access (optional, auto-provisioned when `auto_external_ip_attachment=true`)

5. **ExternalIP reconciliation:**
   - ExternalIP (created at step 1) is pushed to the hub cluster by the fulfillment-service reconciler
   - osac-operator dispatches `create_external_ip` to the manager selected by the NetworkClass profile. The Fabric Manager reserves the address under the ExternalIP API ID and returns it in `osac_result.data.external_ip.address`; OSAC validates the result identity and pool membership, writes status, and transitions the ExternalIP to Allocated. See [Unified Networking — ExternalIP Address Selection and Ownership](/enhancements/OSAC-1433-unified-networking/design.md#externalip-address-selection-and-ownership).
   - ExternalIP labeled `osac.openshift.io/auto-created: "true"` and `osac.openshift.io/auto-created-for: <compute-instance-id>`

6. **Deferred ExternalIPAttachment creation (fulfillment-service internal reconciler):**
   - Once ExternalIP is **Allocated** AND ComputeInstance is **Ready** (with `compute_network_attachment_statuses` populated): fulfillment-service internal reconciler creates ExternalIPAttachment, labeled `osac.openshift.io/auto-created: "true"`
   - This follows the standard creation readiness gate — no exceptions
   - osac-operator ExternalIPAttachment controller dispatches to AAP → fabric manager creates DNAT rule: external IP → VM's primary subnet IP (from `compute_network_attachment_statuses`)
   - ExternalIPAttachment transitions to **Ready**

#### Deletion (reverse order)

7. **Delete ComputeInstance:**
   - **Auto-provisioned cleanup:** If ExternalIP/ExternalIPAttachment were created by the system (`auto_external_ip_attachment=true`, labeled `osac.openshift.io/auto-created: "true"`): parent finalizer deletes ExternalIPAttachment first, then ExternalIP.
   - **Manually created ExternalIPAttachments block deletion** — if the tenant created ExternalIPAttachments explicitly (not labeled `osac.openshift.io/auto-created`), the delete request is rejected. The tenant must remove them first. See [Unified Networking — Deletion Dependency Guards](/enhancements/OSAC-1433-unified-networking/design.md#deletion-dependency-guards).
   - **Default networking resources (VN, Subnet, SG, NATGateway) are NOT cleaned up** — they are tenant-scoped and shared across resources.
   - The lifecycle controller stops the VM and waits until guest execution has ended, retaining the VM object and its immutable placement. It then deletes its SubnetAttachment and waits for its finalizer. The networking controller dispatches `delete_workload_attachment` to the Kubernetes Manager, removing workload-owned policy and detaching the overlay interface.
   - The ordinary `osac-delete-compute-instance` template deletes the stopped KubeVirt VM object and DataVolume only after request cleanup completes. VM teardown does not restore a physical provisioning port. [User]

8. **Delete networking resources:**
    - Shared-network deletion waits for all consumers. Subnet cleanup runs every possibly-started Kubernetes placement before Fabric cleanup. VirtualNetwork cleanup follows its children.
    - SecurityGroup and NetworkClass are OSAC objects with no standalone provider delete job; the fixed shared task table determines all dispatch.

<a id="api-extensions"></a>

### API contract

#### Proto (fulfillment-service)

Use the existing `ComputeNetworkAttachment` field with a single-entry limit:

```protobuf
message ComputeNetworkAttachment {
  SubnetLocalReference subnet = 1;                         // Optional on input; immutable after resolution
  repeated SecurityGroupLocalReference security_groups = 2; // Optional on input; immutable after resolution
}

message ComputeInstanceSpec {
  // ... existing fields ...
  repeated ComputeNetworkAttachment network_attachments = 14; // optional; max 1
  optional bool auto_external_ip_attachment = 18;  // auto-provision ExternalIP + ExternalIPAttachment
}

message ComputeNetworkAttachmentStatus {
  string subnet_ref = 1;               // Subnet ID (echoed from spec)
  string ip_address = 2;               // Discovered from KubeVirt VMI network status after DHCP/overlay assignment
}

message ComputeInstanceStatus {
  // ... existing fields ...
  repeated ComputeNetworkAttachmentStatus compute_network_attachment_statuses = 7; // NEW
}
```

#### Operator CRD (osac-operator)

`ComputeInstanceSpec.NetworkAttachments` has the following behavior:
- Validation limits the repeated field to at most one attachment
- `PrimarySubnetRef()` returns the sole attachment (or no subnet when the list is empty)

CEL validation rule:
```yaml
- rule: "self.networkAttachments.size() <= 1"
  message: "at most one network attachment is supported"
```

`ComputeInstanceStatus` contains the discovered attachment status:

```go
type ComputeInstanceStatus struct {
    // ... existing fields ...
    ComputeNetworkAttachmentStatuses []ComputeNetworkAttachmentStatus `json:"computeNetworkAttachmentStatuses,omitempty"`
}

type ComputeNetworkAttachmentStatus struct {
    SubnetRef string `json:"subnetRef"`
    IPAddress string `json:"ipAddress,omitempty"` // Discovered from KubeVirt VMI after DHCP/overlay assignment
}
```

The SubnetAttachment controller discovers the IP from KubeVirt VMI status and writes request status. The workload controller mirrors the discovered address to `ComputeNetworkAttachmentStatuses`; the feedback controller synchronizes workload status to fulfillment-service.

#### Server Validation (fulfillment-service)

- Validate that `network_attachments` contains at most one entry.
- Primary/default-route resolution: the sole attachment is implicitly primary; VMaaS has no primary field
- BM-only deployment check: if the NetworkClass has no k8sManager, reject ComputeInstance creation

<a id="template-changes-osac-aap"></a>

#### Template responsibilities (osac-aap)

- `osac.templates.ocp_virt_vm/tasks/create_build_spec.yaml`: prepare compute/disks and a stopped VM without provider-specific network/interface definitions. Disable automatic pod-interface attachment during preparation.
- OSAC chooses and persists the namespace, VM name/UID and `primary` selector. Kubernetes `apply_workload_attachment` uses its Fabric-produced inputs and backend configuration to patch the VM network/interface fields and install policy. CUDN may use its `l2bridge` binding; other conforming managers configure their own binding through the same task. Kubernetes Managers do not produce NetworkData.
- Start/reconcile operations preserve manager-owned networking fields and metadata; full stale-manifest replacement is forbidden. The [shared VM ownership contract](/enhancements/OSAC-1435-vmaas-networking/design.md#vm-placement-and-interface-ownership) defines the namespace formula, exact owned fields, result gate and deletion behavior.
- Prepare the DataVolume and stopped VM with deterministic placement/selector; resume VM execution only after AttachmentReady. Stop guest execution before delete_workload_attachment, then remove the VM object after CleanupComplete. [User]
- The sole attachment receives IP + default gateway + DNS via DHCP

<a id="implementation-detailsnotesconstraints"></a>

### Component behavior and constraints

#### Component Responsibility

| Component | Responsibility |
|-----------|---------------|
| fulfillment-service | Validate network_attachments, create CR, auto-provision ExternalIP, write `compute_network_attachment_statuses` from feedback |
| osac-operator ComputeInstance controller | Resolve workload placement, create/wait for target-only SubnetAttachment, mirror request status, trigger AAP, and gate teardown on request and external child cleanup |
| osac-operator ComputeInstance feedback controller | Signal fulfillment-service when mirrored workload status changes |
| osac-operator networking controllers | Own networking resources and SubnetAttachment status/jobs; resolve policy, dispatch apply/delete, discover VMI address |
| AAP template (ocp_virt_vm) | Prepare stopped single-NIC VM in resolved namespace; start after attachment readiness; stop before attachment cleanup and delete afterward |
| fabric_manager (Ansible role) | Shared VN/Subnet/address/NAT operations; VM interface policy belongs to the Kubernetes Manager |
| k8s_manager (Ansible role) | Create/delete Subnet overlay; apply/delete each VM interface attachment and stateful SecurityGroup policy |

#### Primary Attachment Resolution

- The sole attachment is implicitly primary/default route
- More than one attachment is rejected

#### Auto-Provisioned Resource Lifecycle

- ExternalIP created at workload creation time (pool is Ready, creation readiness gate satisfied). ExternalIPAttachment created later by fulfillment-service internal reconciler (after ExternalIP Allocated + ComputeInstance Ready)
- Labeled `osac.openshift.io/auto-created: "true"`
- Parent resource finalizer deletes in order: ExternalIPAttachment → ExternalIP
- **Cleanup failure:** the parent finalizer remains while auto-created ExternalIPAttachments or ExternalIPs remain. Reconciliation retries with backoff and reports the failed child operation; an administrator repairs the cause before cleanup resumes. Enabled-mode failure never silently orphans children. [User]

#### Backward Compatibility Strategy

The existing repeated `network_attachments` field is retained unchanged for API
compatibility. The server and operator validate that it contains at most one
entry; omitted and empty lists invoke default resolution, while a supplied
single entry receives defaults only for missing fields. No new singular field or
dual-field migration is required. The sole VM attachment is implicitly
primary/default, and changing any resolved attachment field requires deleting
and recreating the VM.

### Security Considerations

This feature inherits the existing security model:
- Tenant isolation via `osac.openshift.io/tenant` annotation enforced by OPA policies
- Auto-provisioned resources (ExternalIP, ExternalIPAttachment) inherit tenant annotation from parent ComputeInstance
- No new authentication or authorization changes
- SecurityGroup rules control VM inbound traffic (tenant-configurable via explicit SG or default SG)
- The sole VM attachment uses the same SecurityGroup enforcement as the rest of the fabric

### Failure Handling and Recovery

#### ComputeInstance Controller Reconciliation Failures

- Subnet resolution failure (subnet not found, not Ready): the API rejects the create request with a `FailedPrecondition` error; the ComputeInstance is never created. See [Unified Networking — Creation Readiness Gates](/enhancements/OSAC-1433-unified-networking/design.md#creation-readiness-gates)
- Namespace resolution failure (subnet has no target namespace): ComputeInstance enters Failed state, retries after manual correction
- AAP job failure (template execution error): ComputeInstance enters Failed state with AAP job ID in status, manual investigation required

#### Auto ExternalIP Allocation Failures

- Pool exhaustion: create API call returns error, no resources persisted (pool capacity checked synchronously during the API call — see [auto-provisioning lifecycle](/enhancements/OSAC-1433-unified-networking/design.md#auto-provisioned-external-access))
- ExternalIP provisioning failure: ExternalIP reports failure and optional public access remains unavailable. ComputeInstance readiness follows its ordinary provisioning, attachment and real-IP gates; it does not wait for the optional ExternalIP
- ExternalIPAttachment provisioning failure: DNAT rule not created, inbound traffic does not reach VM (VM functional, external access unavailable)

#### Cleanup Failures

- Auto-provisioned resource cleanup transient failure: finalizer retries
- **Cleanup failure:** the parent finalizer remains while auto-created ExternalIPAttachments or ExternalIPs remain. Reconciliation retries with backoff and reports the failed child operation; an administrator repairs the cause before cleanup resumes. Enabled-mode failure never silently orphans children. [User]

### RBAC and Tenancy

No RBAC or tenancy changes. All new resources (ComputeInstance with its existing fields, auto-provisioned ExternalIP/ExternalIPAttachment) inherit tenant isolation from parent:
- `osac.openshift.io/tenant` annotation propagated from ComputeInstance to auto-created resources
- OPA policies enforce tenant-scoped operations according to each resource API;
  networking resources use create/list/get/delete and do not expose
  update/patch, while supported non-network workload updates remain available
- Tenant User can view and manage auto-provisioned resources (labeled `osac.openshift.io/auto-created: "true"`) via standard API

### Observability and Monitoring

New structured log events:
- ComputeInstance controller: `ResolvedPrimarySubnet` (info), `SubnetResolutionFailed` (error), `AttachmentProvisioning` (info)
- fulfillment-service: `AutoProvisionedExternalIP` (info), `ExternalIPPoolExhausted` (error)

New Kubernetes events on ComputeInstance:
- `NetworkingResolved`: subnet → namespace resolution succeeded
- `NetworkingResolutionFailed`: subnet resolution failed (not found, not Ready, BM-only deployment)
- `AutoExternalIPCreated`: ExternalIP and ExternalIPAttachment auto-provisioned

No new metrics or alerts (existing provisioning duration and failure rate metrics apply).

### Risks and Mitigations

#### Risk: Qualified Kubernetes Manager is unavailable

**Impact:** OSAC-1511 (CUDN) and OSAC-1717 (EVPN) are closed evaluation spikes, not evidence that a Kubernetes Manager meeting the Network Manager Integration Contract is deployed and qualified. Without a compatible, qualified Kubernetes Manager, VM overlay networking cannot be provisioned.

**Mitigation:** Gate VM overlay networking on a deployment-selected Kubernetes Manager that passes NetworkClass compatibility validation and provider qualification for its assigned tasks. Do not treat completion of either evaluation spike as manager qualification.

**Reviewed by:** Engineering / Product

#### Risk: Shared manager contract support is delayed

**Impact:** The legacy two-job lifecycle tracking delivered by OSAC-1459 does not provide the manager registrations, generic task dispatch, validated results, and uncertain-job recovery specified by the Network Manager Integration Contract. VM networking cannot meet its readiness and cleanup requirements until that shared contract is implemented.

**Mitigation:** Deliver the shared contract implementation before enabling VMaaS networking. Require the selected Fabric and Kubernetes Manager stages and their documented readiness and cleanup behavior; do not treat legacy multi-job support alone as qualification.

**Reviewed by:** osac-operator team

#### Risk: ExternalIPPool exhaustion

**Impact:** Auto ExternalIP allocation fails, create API call returns error, tenant cannot create VM with `auto_external_ip_attachment=true`.

**Mitigation:** Pool capacity visible in status; clear error directs tenant to explicit allocation from another pool or contact admin.

**Reviewed by:** Cloud Provider Admin

### Drawbacks

#### Single-attachment compatibility constraint

The repeated `network_attachments` field remains in place to avoid an API
shape change. New requests containing more than one entry are rejected by
validation.

### Provider Networking Disabled

Apply the [unified disabled-mode contract](/enhancements/OSAC-1433-unified-networking/design.md#provider-networking-control)
for API validation, skipped jobs, external-address state and logical cleanup.
VMaaS-specific behavior: a valid VM provisions on the platform default network,
with its ordinary default interface instead of the enabled-mode stopped-empty
preparation/manager handoff. Tenant Subnet placement and SecurityGroup behavior
are not claimed. Real platform VMI feedback may be reported, but cannot be
relabeled as evidence of tenant attachment or public routing. VM provisioning
does not wait for an unallocated automatic ExternalIP. [PRD: FR-8] [User]

Re-enabling provider networking follows the [shared activation preflight](/enhancements/OSAC-1433-unified-networking/design.md#provider-networking-control). For this extension, drain and delete every ComputeInstance through the normal VM lifecycle; the extension does not migrate a running baseline VM to a tenant Subnet. The shared gate also waits for workloads owned by other extensions and blocks new workload Creates while activation is pending. Reads and deletes remain available. After the deployment profile and configured pools regain real provider readiness, ComputeInstances use their ordinary network-reference gates. A disabled-mode ProvisioningDisabled state does not satisfy enabled readiness. An ordinary restart of an already enabled deployment does not require a new drain. [User]

## Alternatives Considered

### Alternative 1: Single shared NetworkAttachment message with optional primary field

Instead of creating `ComputeNetworkAttachment`, extend the shared `NetworkAttachment` message with an optional `primary` field usable by all resource types.

**Rejected because:** Other resource types (Cluster, BaremetalInstance) have different attachment semantics. Resource-specific attachment messages provide cleaner API surface and type-specific validation; all current workload types enforce a single tenant attachment.

### Alternative 2: Capacity exhaustion creates Failed resource instead of returning error

Instead of returning an error when ExternalIPPool has no capacity, create a Failed ComputeInstance with a status condition.

**Rejected because:** Pool capacity is validated synchronously during the API call — if the pool is exhausted, the call fails atomically and no resources are persisted. Creating a Failed resource adds cleanup burden and audit trail complexity. Clear API error with no persisted state is simpler.

## Open Questions

### ~~1. Should capacity exhaustion return an API error or create a Failed resource?~~ — Resolved

Resolved: Return error, no resource persisted. Pool capacity checked synchronously. No Failed resource.

## Test Plan

- **[DEV/QE] VM handoff:** exercise two provider bindings through the same unchanged workload template. Verify exact supplied namespace/VM UID, empty stopped preparation, manager-owned interface fields, no policy-free start, preservation on restart/reconcile, identical replay after start, rejection of replacement VM UIDs and cleanup before VM deletion.

### Shared Attachment Contract Validation

- **[DEV] Contract:** assert request-as-resource, VM-as-target, normalized attachment policy, request UID/generation results, and mandatory adapter entry points. A valid no-change result is accepted; missing required data is rejected.
- **[DEV] Contract:** prepare deterministic stopped-VM placement and selector before apply; adapters apply policy without a running VMI; VM start cannot precede AttachmentReady, and VM stop is confirmed before delete dispatch.
- **[DEV] Envtest:** install failure leaves request/VM non-ready; request owns discovered VMI IP and workload controller mirrors it; delete failure retains finalizers and blocks VM teardown.
- **[QE] E2E:** two VMs on one Subnet using different groups enforce their respective stateful policies before traffic; restart during apply/delete preserves stable API ownership and backend-VM UID fencing; external child cleanup completes before request/workload teardown. [User]


### Provider Networking Control (FR-8)

- **[DEV] Envtest / [QE] E2E:** request re-enablement with a baseline workload or a workload whose networking was suspended by disablement still present. Assert activation and new workload creates are blocked, existing Read/Delete remain available, and no apply/lease/reservation or tenant attachment mutation runs. Drain workloads and jobs, repair backend leftovers, then activate; stale skipped readiness cannot admit new workloads until genuine network prerequisites are ready. Repeated enabled restarts do not invoke a fresh drain.

- Verify normal VM provision/delete jobs still run with the shared setting
  disabled and sufficient baseline connectivity; no network provider job runs.
- Verify active network jobs are cancelled and awaited before skipped status
  or network-finalizer release, including deletion and retryable AAP failures.
- Verify non-allocating Networking API resources report `Ready=True`, reason
  `ProvisioningDisabled`, with a skipped-work message after logical
  preconditions pass; ExternalIP status distinguishes unconfirmed from
  confirmed real allocation and never exposes a placeholder. Enabled mode
  retains normal provider behavior.
- Verify invalid API/defaulting/dependency requests remain rejected,
  SecurityGroup defaulting/immutability remains unchanged, and automatic
  attachments still wait for Allocated + workload Ready.
- Verify omitted or empty API attachments retain existing defaulting and
  missing-default errors. A valid VM that has no tenant attachment provisions
  on the platform default network when disabled; supplied API references retain
  their existing validation, and enabled mode continues tenant placement.

### Unit Tests

- fulfillment-service: max-one validation (accept no attachment or one attachment)
- fulfillment-service: max-one `network_attachments` validation
- fulfillment-service: omitted and partial attachment defaulting (empty `security_groups` is missing; supplied values are preserved; a missing group list defaults only for the tenant default VirtualNetwork and is rejected for a non-default subnet without caller-supplied groups)
- fulfillment-service: BM-only deployment validation (reject VM when no k8s_manager)
- fulfillment-service: auto ExternalIP pool selection (pick READY pool with most capacity, respect IP family)
- osac-operator ComputeInstance controller: `PrimarySubnetRef()` resolution (implicit single attachment)

### Integration Tests

- **[DEV/QE] Optional external access:** fail asynchronous ExternalIP allocation after an admitted workload Create. Core workload readiness still follows its ordinary provision/attachment/IP gates; the ExternalIP reports failure and no ExternalIPAttachment is created until a real allocation and Ready target exist. Pool exhaustion at initial admission still fails atomically.

- E2E: create ComputeInstance with two attachments, verify the API rejects the request
- E2E: create ComputeInstance with `--external-ip-attachment`, verify auto ExternalIP + ExternalIPAttachment created, DNAT rule functional
- E2E: delete ComputeInstance with auto-provisioned resources, verify ExternalIPAttachment and ExternalIP cleaned up
- E2E: create ComputeInstance in BM-only deployment, verify error returned
- E2E: create ComputeInstance with one `network_attachments` entry, verify it is used as the default route

### Tricky Test Cases

- ComputeInstance with one attachment (verify implicit default-route behavior)
- ExternalIPPool exhaustion (verify error returned, no resource created)
- Auto-provisioned resource cleanup failure (verify retained finalizer/ownership on persistent failure, successful cleanup after repair)

## Graduation Criteria

**Note:** This section will be updated when the enhancement is targeted at a release.

Proposed maturity level: **Tech Preview** → **GA**

Tech Preview criteria:
- [ ] API fields (`network_attachments`, `auto_external_ip_attachment`) implemented in fulfillment-service
- [ ] Operator CRD updated with max-one CEL validation
- [ ] Single-NIC template support (`osac.templates.ocp_virt_vm`) implemented
- [ ] Auto ExternalIP attachment provisioning functional
- [ ] Integration tests pass (E2E coverage for max-one validation, auto ExternalIP)
- [ ] Documentation: API reference, user guide for simplified VM creation

GA criteria:
- [ ] Network Manager Integration Contract registration, dispatch, result validation, and recovery behavior delivered; the selected compatible Fabric/Kubernetes Manager pair passes provider qualification and is production-tested
- [ ] Production deployment verified (MOC or other OSAC deployment)
- [ ] User feedback incorporated (usability, error messages, edge cases)

## Upgrade and Downgrade Strategy

### Implementation baseline

ComputeInstance already participates in the networking API. Today's flow:

1. Tenant creates VirtualNetwork, Subnet, SecurityGroup via API
2. osac-operator's networking controllers reconcile each resource as a standalone AAP job, using `implementation_strategy` to select the Ansible role (e.g., `osac.templates.cudn_net.create_subnet`)
3. Tenant creates ComputeInstance with `network_attachments` (`ComputeNetworkAttachment`, no `primary` field, single-NIC only)
4. osac-operator's ComputeInstance controller resolves subnet → namespace, triggers AAP job
5. AAP template (`osac.templates.ocp_virt_vm`) creates KubeVirt VirtualMachine with one `l2bridge` interface in the subnet's CUDN namespace

### What Already Works

- `network_attachments` field exists on ComputeInstanceSpec (field 14)
- Operator CRD has `NetworkAttachments []ComputeNetworkAttachment` with CEL cardinality and immutability rules; the complete resolved attachment is immutable
- Subnet-to-namespace resolution is implemented
- The template creates VMs in the correct namespace
- ExternalIPAttachment with `compute_instance` target works end-to-end

### What's Missing

- Existing `ComputeNetworkAttachment` has no `primary` field; the compatibility field remains single-attachment only
- Service-level maximum-one validation and field-level defaulting are still required
- At most one attachment — template creates one `l2bridge` interface
- The current implementation still uses `implementation_strategy` annotations; migration to the generic dispatcher is defined by the Network Manager Integration Contract.
- BM-only deployment validation (reject VM when no k8sManager)
- Auto ExternalIP allocation (tenant must manually create ExternalIP + ExternalIPAttachment)

### Upgrade and downgrade gates

Follow the [shared coordinated rollout and rollback requirements](/enhancements/OSAC-1433-unified-networking/design.md#upgrade-and-downgrade-strategy).
Preserve existing resource/provider bindings, accepted results and cleanup
context. Wire compatibility does not establish backend migration safety.
Drain or verifiably adopt legacy state before using new ownership keys; finish
cleanup with compatible components before downgrading to a version that cannot
interpret the retained state. Disabled-mode provider leftovers still require
confirmed provider/manual cleanup before their addresses or hosts are reused.

### Service compatibility

The repeated `network_attachments` wire field and existing single
`--network-attachment` flag remain. New validation permits at most one entry.
Existing single-attachment requests remain compatible; preservation of running
VMs additionally requires verified manager ownership and backend VM UID checks.
The new network-status field is additive. Provider-specific stopped-VM
preparation, field ownership and attachment gates require the corresponding
workload template and controllers before new dispatch.

## Version Skew Strategy

### Control Plane Skew

Use the [shared version-skew rules](/enhancements/OSAC-1433-unified-networking/design.md#version-skew-strategy).
Fulfillment, the workload/networking controllers and AAP content must agree on
the handoff and cleanup protocol. A coordinated installer release still has a
rolling transition; incompatible components must not dispatch new work.

### Client Skew

osac-cli (n-1) with fulfillment-service (n):
- The CLI uses the existing single `--network-attachment` flag and sends the existing `network_attachments` field

osac-cli (n) with fulfillment-service (n-1):
- The CLI remains compatible with the existing `network_attachments` field; a second attachment is rejected client-side and server-side

Recommendation: keep osac-cli and fulfillment-service within one minor version.

## Support Procedures

### Symptom: ComputeInstance stuck in Pending, condition "NetworkingResolutionFailed"

**Detection:**
```bash
kubectl describe computeinstance <name> -n <namespace>
# Check status.conditions for NetworkingResolutionFailed
```

**Cause:** Subnet not found, not Ready, or BM-only deployment (no k8s_manager)

**Resolution:**
1. Check Subnet status: `kubectl get subnet <subnet-name> -n <namespace>`
2. If Subnet is not Ready, investigate Subnet provisioning failure (check AAP job logs)
3. If BM-only deployment, tenant must create VM in a deployment with k8s_manager configured

### Symptom: VM has no default gateway

**Detection:** VM cannot reach external networks, `ip route` shows no default route

**Cause:** The sole attachment was not resolved or the subnet did not provide DHCP gateway information

**Resolution:**
1. Check ComputeInstance spec: `kubectl get computeinstance <name> -n <namespace> -o yaml`
2. Verify the single `networkAttachments[]` entry resolves to the intended subnet
3. If missing or incorrect, delete and re-create ComputeInstance with the intended subnet

### Symptom: Deletion waits for automatic external cleanup

**Detection:** The parent remains Deleting with its finalizer and child references; an auto-created ExternalIPAttachment or ExternalIP reports a failed cleanup operation.

**Cause:** The provider has not confirmed DNAT removal or address release.

**Resolution:**
1. Inspect parent, child and AAP operation diagnostics to identify the failed task.
2. Repair provider connectivity/configuration or the backend operation; let reconciliation retry with the same durable operation identity.
3. Confirm attachment deletion before address release and parent deletion. Preserve finalizers and address ownership until successful cleanup; do not force-remove finalizers. [User]

### Disabling automatic ExternalIP requests

To disable auto ExternalIP attachment:
- Remove or redact ExternalIPPool CRs (capacity exhaustion prevents auto allocation)
- No API extension to disable (fields are part of CRD, cannot be removed at runtime)

Consequences:
- Auto ExternalIP allocation fails with error (resource not created)
- Manual ExternalIP workflows remain functional
- No impact on existing running VMs

### Provider networking intentionally skipped

Set `global.networking.provisioningEnabled=false` through the Helm setting or Enclave Wizard checkbox
installation/upgrade value map and complete both operator rollouts. Inspect
`Ready=True`/`ProvisioningDisabled` conditions on non-allocating Networking API
resources and tracked network job states. Ordinary workload provisioning
remains active with the baseline connectivity described
above. Networking APIs remain available; no provider allocation, routing,
port movement, DHCP discovery, or cleanup is supplied by the skipped path.
Existing provider resources may require manual/provider-side cleanup. [User]

## Infrastructure Needed

- AAP execution environment with `osac.templates.ocp_virt_vm` role updated for single-NIC support
- A deployment-selected Kubernetes Manager that implements the shared contract and passes provider qualification for VM overlay provisioning
- Integration test environment with a deployment-selected, compatible Fabric/Kubernetes Manager pair, including an overlay-capable Kubernetes Manager.

---

## Provenance

Authored: revise @ design 0.11.5 - 6d28c88, workspace docs/OSAC-1433-network-manager-provider-guide @ eabff3b59 (76 behind origin/main, dirty)
Final: revise @ design 0.11.5 - 1c6e8de, workspace docs/OSAC-1433-network-manager-provider-guide @ eabff3b59 (98 behind origin/main, dirty)

> Context changed between revise and revise.

> This document's phase history does not include an initial /draft — structure was not verified against the template from origin.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.5","ai_workflows":"1c6e8de","source_repo":"eabff3b59 (dirty)","source_repo_branch":"docs/OSAC-1433-network-manager-provider-guide","commits_behind_main":98,"commits_ahead_main":2,"main_ref":"main","phases":["revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":true} -->
