---
title: vmaas-networking
authors:
  - dmanor@redhat.com
creation-date: 2026-07-08
last-updated: 2026-10-15
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

This enhancement is a service-specific extension of the [Unified Networking EP](/enhancements/OSAC-1433-unified-networking/design.md). The Unified Networking EP defines shared tenant-resource meaning and lifecycle. The [Network Manager Integration Contract](/enhancements/OSAC-1433-network-manager-integration-contract-networking/design.md) defines manager tasks, inputs, results, and attachment descriptor mapping. This document defines VMaaS sequencing, KubeVirt construction, and address observation.

VMaaS controllers create the internal SubnetAttachment before the VM exists. The Kubernetes Manager installs policy and returns an attachment descriptor containing the namespace, network role, optional NAD name, and binding. OSAC validates and stores the descriptor. VMaaS builds the KubeVirt network/interface fields from the fixed contract table, stamps the stable ComputeInstance API ID label while retaining the existing hub-CR-name label, and creates the VM with those fields and its requested run strategy. VMaaS observes the assigned address from VMI status.

VMaaS networking remains tied to its current KubeVirt runtime. A different VM runtime requires a separate VMaaS runtime integration and service-specific design.

VMaaS inherits the [Unified Networking deployment support boundary](/enhancements/OSAC-1433-unified-networking/design.md#deployment-support-boundary) and [networking hub support boundary](/enhancements/OSAC-1433-unified-networking/design.md#networking-hub-support-boundary). This service extension supports one hosting cluster.

ComputeInstance currently uses a `ComputeNetworkAttachment` message. This enhancement keeps the existing repeated attachment field optional (populating it with tenant defaults when omitted), enforces a maximum of one entry, and adds `auto_external_ip_attachment` to enable inbound connectivity in one API call. VMaaS has no primary field: the sole attachment is implicitly the default route. See the [PRD](prd.md) for detailed requirements.

The manager-backed resource flows and provider IP-discovery/readiness behavior in this document describe enabled mode. Feature-gated disabled behavior remains described under [Provider Networking Disabled](#provider-networking-disabled). API-level SecurityGroup rule validation and reference checks remain active in disabled mode; OSAC submits no provider operation to apply or remove rules.

The Network Manager Integration Contract owns registration, task/input/result envelopes, attachment descriptor validation and mapping, provider values, retries, and shared cleanup. This design specifies VMaaS sequencing and readiness gates, not another copy of those shared rules.

## Motivation

Each VM receives its tenant attachment before the VM object exists. The selected Kubernetes Manager installs SecurityGroup policy through the target-only `SubnetAttachment` and returns a closed descriptor. VMaaS maps that descriptor to KubeVirt network/interface fields, creates the VM with its requested run strategy, and publishes the address observed from VMI status. Optional external access follows the shared allocation and attachment lifecycle.

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
- A VM runtime abstraction or support for VM runtimes other than KubeVirt.

## Proposal

### Working with provider-supplied managers

The deployment selects its Fabric and Kubernetes Managers through NetworkClass. A provider may supply either role or both. The same VMaaS template works with any conforming Kubernetes Manager for the current KubeVirt target. The Kubernetes Manager prepares Subnet networking and, for each VM attachment, installs policy and returns the closed descriptor defined by the manager contract. It does not read or modify a VirtualMachine.

| Step | VMaaS responsibility | Provider-manager responsibility |
|---|---|---|
| Subnet preparation | Wait for Subnet Ready. | Fabric then Kubernetes `create_subnet`; each role may return opaque `provider_values`. |
| Attach | Create SubnetAttachment with ComputeInstance kind and API UUID; wait for `NetworkAttachmentsReady=True`. | Kubernetes `apply_subnet_attachment` installs policy and returns the descriptor. |
| Create and observe | Build VM network fields from the contract table, stamp the stable ComputeInstance UUID label, create VM with requested run strategy, and mirror VMI IP. | No VM access or modification. |
| Delete | Delete VM and wait until it is gone; then delete SubnetAttachment and wait for finalizer completion. | Kubernetes `delete_subnet_attachment` removes attachment-owned policy and per-VM backend objects. |

The service controller owns the order. The networking controller dispatches attachment tasks, validates the descriptor, updates SubnetAttachment status, and retains its finalizer until cleanup succeeds.

### VM placement and interface ownership

The design uses the single hosting cluster configured through `OSAC_REMOTE_CLUSTER_KUBECONFIG`. The deterministic ComputeInstance API UUID is known before VM creation and is passed to `apply_subnet_attachment` as the target ID; no VM name or VM object is passed.

The Kubernetes Manager owns the descriptor (`namespace`, network role, NAD name for a secondary network, and binding). OSAC owns the mapping from that descriptor to KubeVirt fields. VMaaS writes those fields into the VM at creation, preserves them on later reconciles, and owns every other VM field.

The manager contract's fixed table is the single source of truth. VMaaS replaces the workload template's default `networks` and `interfaces` entries (today `pod: {}` with `l2bridge` in both Linux and Windows specs) with the table row selected by the descriptor. For a secondary network, VMaaS also sets `autoattachPodInterface: false`; for a primary network it leaves that field unset. VMaaS adds `osac.openshift.io/compute-instance-id: <ComputeInstance API UUID>` to the VM pod template. It retains the existing `osac.openshift.io/computeinstance: <hub CR name>` label as well.

Example VM as created by VMaaS for the primary/l2bridge descriptor:

```yaml
metadata:
  namespace: osac-net-7f3a
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        osac.openshift.io/computeinstance: <hub-cr-name>
        osac.openshift.io/compute-instance-id: <compute-instance-api-uuid>
    spec:
      networks:
        - {name: tenant, pod: {}}
      domain:
        devices:
          interfaces:
            - {name: tenant, binding: {name: l2bridge}}
```

Kubernetes `apply_subnet_attachment` installs the complete effective policy, selecting the fixed ComputeInstance UUID label, then returns the descriptor. It must not read or modify any VirtualMachine. VMaaS creates the VM only after the descriptor is accepted and SubnetAttachment is ready.

### Workflow Description

The ComputeInstance's `network_attachments` field is the tenant's desired Subnet and SecurityGroups. The internal `SubnetAttachment` is an OSAC request for applying that attachment to one workload. It tracks the manager job, descriptor, attachment readiness, discovered address, and cleanup finalizer. It is not a second tenant attachment. The networking controller resolves the ComputeInstance target and desired attachment before dispatch. The VM does not exist until the request is Ready.

#### Networking prerequisites

Use the shared [creation readiness gates](/enhancements/OSAC-1433-unified-networking/design.md#creation-readiness-gates) and [deletion dependency guards](/enhancements/OSAC-1433-unified-networking/design.md#deletion-dependency-guards), together with [tenant defaults](/enhancements/OSAC-1433-default-networking/design.md).
Before VM creation, the selected Subnet and its SecurityGroups must be ready. Fabric tasks complete before Kubernetes Subnet realization; VMaaS does not allocate or reinterpret provider values.

#### VM Creation

1. **Tenant creates ComputeInstance.** The API resolves omitted or empty network attachments through the documented tenant defaults, validates the one-attachment limit and same-VirtualNetwork SecurityGroup references, and creates the ComputeInstance with its desired attachment.

2. **Prepare the network attachment before creating the VM.** The ComputeInstance controller registers its workload cleanup finalizer and creates one SubnetAttachment whose target contains only `kind: ComputeInstance` and the stable API UUID. The networking controller resolves the ready Subnet, VirtualNetwork and complete SecurityGroup rules, then dispatches Kubernetes `apply_subnet_attachment`. The manager installs policy using the fixed `osac.openshift.io/compute-instance-id: <target.id>` selector and returns the attachment descriptor. OSAC validates and stores it; the SubnetAttachment becomes `NetworkAttachmentsReady=True` only after the exact job result is accepted.

3. **Create the VM with its network already configured.** VMaaS maps the accepted descriptor through the fixed table in the [Network Manager Integration Contract](/enhancements/OSAC-1433-network-manager-integration-contract-networking/design.md#workload-attachment-result), replaces the template's default network/interface entries, adds the stable UUID pod label, and runs the normal VM template with the requested run strategy. The VM object is created with the network already configured; when the requested run strategy starts it, its first VMI uses that network. The manager never reads or modifies the VM.

4. **Observe the workload.** DHCP supplies the guest address, gateway, prefix and DNS. The ComputeInstance controller observes the first non-empty tenant interface address from VMI status, mirrors it to workload status, and completes readiness when attachment and address discovery succeed. The supported single-interface flow keeps that address unambiguous.

#### IP Discovery (feedback loop)

4. **SubnetAttachment controller discovers the VM IP:**
   - Reads the first non-empty `ipAddress` in KubeVirt VMI `status.interfaces`, matching the existing `ComputeInstance.status.ipAddress` observation behavior. The supported VM profile exposes only the tenant-facing interface, so this address belongs to the sole attachment.
   - DHCP assigns the address inside the guest; KubeVirt VMI status is OSAC's observation source. OSAC does not query a DHCP server for the VM address. More than one address-reporting interface is outside this design.
   - Writes the assigned IPv4 address and `IPDiscoveryComplete` to request status.
   - The ComputeInstance lifecycle controller mirrors the request's readiness/IP to `computeNetworkAttachmentStatuses` on workload status.
   - The feedback controller transports that status through Signal RPC to fulfillment-service; fulfillment persists `compute_network_attachment_statuses` for tenant visibility and ExternalIPAttachment target resolution.
   - The request owns manager job history and discovery status; the workload API remains the sole source of desired attachment intent. [User]

#### External Access (optional, auto-provisioned when `auto_external_ip_attachment=true`)

5. **ExternalIP reconciliation:**
   - ExternalIP (created at step 1) is pushed to the hub cluster by the fulfillment-service reconciler
   - osac-operator dispatches `create_external_ip` to the manager selected by the NetworkClass profile. The Fabric Manager reserves the address under the ExternalIP API ID and returns it in `osac_result.data.external_ip.address`; OSAC validates the result identity and pool membership, writes status, and transitions the ExternalIP to Allocated. See [Unified Networking — ExternalIP Address Selection and Ownership](/enhancements/OSAC-1433-unified-networking/design.md#externalip-address-selection-and-ownership).
   - ExternalIP labeled `osac.openshift.io/auto-created: "true"` and `osac.openshift.io/auto-created-for: <compute-instance-id>`

6. **ExternalIPAttachment creation after readiness (fulfillment-service internal reconciler):**
   - Once ExternalIP is **Allocated** AND ComputeInstance is **Ready** (with `compute_network_attachment_statuses` populated): fulfillment-service internal reconciler creates ExternalIPAttachment, labeled `osac.openshift.io/auto-created: "true"`
   - This follows the standard creation readiness gate — no exceptions
   - osac-operator ExternalIPAttachment controller dispatches to AAP → fabric manager creates DNAT rule: external IP → VM's primary subnet IP (from `compute_network_attachment_statuses`)
   - ExternalIPAttachment transitions to **Ready**

#### Deletion (reverse order)

1. **Delete ComputeInstance.** Auto-provisioned ExternalIP and ExternalIPAttachment cleanup runs first. VMaaS deletes the VM and waits until the VirtualMachine is gone. Only then does it delete the SubnetAttachment and wait for the networking finalizer. Kubernetes `delete_subnet_attachment` removes attachment-owned policy and per-VM backend objects. The VM is never patched to remove networking fields.

2. **Delete shared networking resources.** Shared-resource deletion waits for all consumers. Subnet cleanup runs its Kubernetes task before Fabric cleanup; VirtualNetwork cleanup follows its children.

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
  string ip_address = 2;               // Sole attachment IP observed from the KubeVirt VMI after DHCP/overlay assignment
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
    IPAddress string `json:"ipAddress,omitempty"` // Sole attachment IP observed from the KubeVirt VMI after DHCP/overlay assignment
}
```

The SubnetAttachment controller observes the first non-empty KubeVirt VMI
interface IP for the sole tenant attachment and writes request status. The
workload controller mirrors the discovered address to
`ComputeNetworkAttachmentStatuses`; the feedback controller synchronizes
workload status to fulfillment-service. The existing ComputeInstance
`status.ipAddress` path also reads the first non-empty VMI interface IP. This
behavior is specific to VMaaS's current KubeVirt single-interface flow.

#### Server Validation (fulfillment-service)

- Validate that `network_attachments` contains at most one entry.
- Primary/default-route resolution: the sole attachment is implicitly primary; VMaaS has no primary field
- BM-only deployment check: if the NetworkClass has no k8sManager, reject ComputeInstance creation

<a id="template-changes-osac-aap"></a>

#### Template responsibilities (osac-aap)

- The workload template receives the already-built `networks`, `interfaces`, optional `autoattachPodInterface`, and stable ComputeInstance UUID label. It writes those values into the VirtualMachine alongside workload-owned fields; it contains no networking logic.
- The networking controller creates the SubnetAttachment before the VM and stores the accepted attachment descriptor. The Kubernetes Manager installs policy and returns the descriptor; it never reads or modifies a VirtualMachine.
- The VM template preserves the selected attachment fields on later reconciles. The single-interface flow receives its IPv4 address, gateway, and DNS through DHCP.

### Component behavior and constraints

#### Component Responsibility

| Component | Responsibility |
|-----------|---------------|
| fulfillment-service | Validate network_attachments, create CR, auto-provision ExternalIP, write `compute_network_attachment_statuses` from feedback |
| osac-operator ComputeInstance controller | Create/wait for target-only SubnetAttachment before VM creation, mirror request status, trigger AAP, and order VM deletion before attachment cleanup and external child cleanup |
| osac-operator ComputeInstance feedback controller | Signal fulfillment-service when mirrored workload status changes |
| osac-operator networking controllers | Own networking resources and SubnetAttachment status/jobs; resolve policy, dispatch apply/delete, discover VMI address |
| AAP template (ocp_virt_vm) | Write the OSAC-mapped network/interface fields and stable UUID label into the VM at creation; delete the VM before SubnetAttachment cleanup |
| fabric_manager (Ansible role) | Shared VN/Subnet/address/NAT operations; VM interface policy belongs to the Kubernetes Manager |
| k8s_manager (Ansible role) | Create/delete the manager's Subnet overlay realization; apply/delete per-attachment policy and return/consume the closed attachment descriptor through `SubnetAttachment` |

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

**Impact:** OSAC-1511 (CUDN) and OSAC-1717 (EVPN) are closed evaluation spikes, not evidence that a Kubernetes Manager meeting the Network Manager Integration Contract is deployed and qualified. Without a Kubernetes Manager that fulfills its role contract and passes provider conformance, VM overlay networking cannot be provisioned.

**Mitigation:** Gate VM overlay networking on a deployment-selected Kubernetes Manager that fulfills its role contract and passes provider conformance. Do not treat completion of either evaluation spike as manager qualification.

**Reviewed by:** Engineering / Product

#### Risk: Shared manager contract support is delayed

**Impact:** The legacy two-job lifecycle tracking delivered by OSAC-1459 does not provide manager registration, generic task dispatch, or validated results required by the Network Manager Integration Contract. VM networking cannot meet its readiness and cleanup requirements until that shared contract is implemented.

**Mitigation:** Deliver the shared contract implementation before enabling VMaaS networking. Require each selected manager role to meet its documented readiness and cleanup behavior; do not treat legacy multi-job support alone as qualification.

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

- **[DEV/QE] Single-cluster Subnet lifecycle:** verify one Kubernetes `create_subnet` task follows the Fabric task and Subnet readiness waits for both.
- **[DEV/QE] Descriptor-to-VM flow:** exercise primary/l2bridge and secondary/bridge descriptors with Linux and Windows templates. Verify SubnetAttachment becomes ready while no VirtualMachine exists; VMaaS constructs exactly the `networks`, `interfaces`, and `autoattachPodInterface` fields from the contract table, stamps `osac.openshift.io/compute-instance-id: <ComputeInstance API UUID>`, and creates the VM with its requested run strategy. The manager policy selects that label; a VM lacking it receives only Subnet baseline behavior. The first non-empty VMI interface address is mirrored to status. Retry returns the same descriptor. Delete the VM first, then removes policy through SubnetAttachment cleanup.

### Shared Attachment Contract Validation

- **[DEV] Contract:** assert the SubnetAttachment as `resource`, and overlay `target` containing only `kind: ComputeInstance` and stable API UUID, plus normalized attachment policy and result identity. Validate missing namespace, absent tenant label, invalid role/binding, invalid NAD presence, and extra descriptor fields.
- **[DEV] VM construction:** verify VMaaS maps each supported descriptor through the contract table for both Linux and Windows templates, adds the fixed UUID label, and creates the VM with its requested run strategy. The manager has no VirtualMachine read or write. Verify VM deletion precedes attachment cleanup.
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
- [ ] Network Manager Integration Contract registration, dispatch, result validation, and recovery behavior delivered; the selected Fabric and Kubernetes Managers each pass provider qualification, and their documented value requirements align and is production-tested
- [ ] Production deployment verified (MOC or other OSAC deployment)
- [ ] User feedback incorporated (usability, error messages, edge cases)

## Upgrade and Downgrade Strategy

### Current implementation baseline

This subsection describes the existing code as a migration starting point; it
is not the proposed manager flow described above. The target design replaces legacy dispatch with the registered manager contract and creates VMs with network fields derived from the accepted attachment descriptor.

ComputeInstance already participates in the networking API. The current flow
is:

1. Tenant creates VirtualNetwork, Subnet, SecurityGroup via API
2. osac-operator's networking controllers reconcile each resource as a standalone AAP job, using `implementation_strategy` to select the Ansible role (e.g., `osac.templates.cudn_net.create_subnet`)
3. Tenant creates ComputeInstance with `network_attachments` (`ComputeNetworkAttachment`, no `primary` field, single-NIC only)
4. osac-operator's ComputeInstance controller resolves subnet → namespace, triggers AAP job
5. AAP template (`osac.templates.ocp_virt_vm`) creates KubeVirt VirtualMachine with one `l2bridge` interface in the subnet's CUDN namespace
6. The ComputeInstance controller observes the first non-empty IP in KubeVirt VMI interface status and publishes it as `ComputeInstance.status.ipAddress`; this is the existing KubeVirt-specific IP observation path.

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
- The current implementation still uses `implementation_strategy` annotations; migration to captured manager names and the generic dispatcher is defined by the Network Manager Integration Contract.
- BM-only deployment validation (reject VM when no k8sManager)
- Auto ExternalIP allocation (tenant must manually create ExternalIP + ExternalIPAttachment)

### Upgrade and downgrade gates

Follow the [shared coordinated rollout and rollback requirements](/enhancements/OSAC-1433-unified-networking/design.md#upgrade-and-downgrade-strategy).
Preserve existing resource/provider bindings, accepted results and cleanup
context. Wire compatibility does not establish backend migration safety.
Drain or verifiably adopt legacy state before using new ownership keys; finish
cleanup with components that support the same manager contract before downgrading to a version that cannot
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
rolling transition; components that do not support the same manager contract must not dispatch new work.

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
2. Repair provider connectivity/configuration or the backend task; let reconciliation retry idempotently with the same stable ComputeInstance API ID and captured task inputs.
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
- Integration test environment with a deployment-selected Fabric and Kubernetes Managers whose documented value requirements align, including an overlay-capable Kubernetes Manager.

---

---

## Provenance

Authored: revise @ design 0.11.3 - cc0daa6, workspace main @ 06d340f90 (43 behind origin/main)
Final: manual-edit [manual] @ design 0.11.5 - 0c7ea35, workspace main @ 14c4d6f1c (17 behind origin/main)

> Context changed between revise and manual-edit.

> This document's phase history does not include an initial /draft — structure was not verified against the template from origin.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.5","ai_workflows":"0c7ea35","source_repo":"14c4d6f1c","source_repo_branch":"main","commits_behind_main":17,"commits_ahead_main":0,"main_ref":"main","phases":["revise","revise","respond","manual-edit","revise","manual-edit","revise","manual-edit","revise","respond","respond","revise","revise","revise","revise","manual-edit"],"authoring_modes":["manual","skill"],"context_changed":true,"origin_untracked":true} -->
