---
title: bmaas-networking
authors:
  - dmanor@redhat.com
creation-date: 2026-07-08
last-updated: 2026-09-24
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-1437
prd: "prd.md"
see-also:
  - "Unified Networking: /enhancements/OSAC-1433-unified-networking"
  - "Default Networking: /enhancements/OSAC-1433-default-networking"
  - "baremetal-instance-api: https://github.com/osac-project/baremetal-instance-api"
  - "CaaS BM Worker Provisioning: /enhancements/OSAC-2135-caas-bare-metal-worker-provisioning"
replaces:
  - N/A
superseded-by:
  - N/A
---

# BMaaS Networking — Switch Port Configuration and Tenant-Defined Interface Mapping

BMaaS networking provides single-NIC BaremetalInstance provisioning with optional tenant-specified physical interface mapping, switch port configuration via dispatcher, IP address feedback through CR status, and auto-provisioned external access (ExternalIP).

## Summary

This document is a per-service expansion of the [Unified Networking EP](/enhancements/OSAC-1433-unified-networking/design.md). The unified EP defines the shared architecture and [deployment support boundary](/enhancements/OSAC-1433-unified-networking/design.md#deployment-support-boundary); BMaaS networking supports connected deployments only and does not add air-gapped or disconnected networking support. This document defines how BMaaS consumes that architecture.
Networking resources support only Create, List/Get, and Delete, and
`BaremetalInstance` network attachment fields are immutable after creation;
changes require delete and recreate.

BMaaS networking also inherits the [Unified Networking hub support
boundary](/enhancements/OSAC-1433-unified-networking/design.md#networking-hub-support-boundary):
OSAC networking supports exactly one provider-owned hub per deployment.
Multi-hub networking placement, cross-hub resource coordination, and
cross-hub network connectivity are unsupported. This boundary applies only to
the networking area and does not define hub behavior for other OSAC areas.
Multiple hosting/workload clusters remain supported where a networking feature
explicitly specifies them.

`BaremetalInstance` keeps its repeated `BareMetalNetworkAttachment` field for API compatibility, with a contract of at most one entry. The optional `interface` and `primary` fields retain their existing semantics; with one entry, `primary` is implicit, omission and `true` are accepted, and `false` is rejected. The fulfillment-service copies that attachment into the existing BaremetalInstance CR. At the networking handoff, BMF submits the shared private `NetworkAttachment` request; fulfillment reconciliation materializes an internal `NetworkAttachment` CR that the osac-operator networking controller reconciles. BMF retains host provisioning, reboot, and lifecycle orchestration. The same request/controller boundary is used for VMaaS and CaaS. See [PRD](prd.md) for detailed requirements.

## Motivation

Bare-metal servers require explicit switch port configuration to participate in the OSAC Networking API. Unlike VMs (which live inside an OVN overlay bridged to the fabric), BM servers connect directly to the physical fabric — each NIC's switch port must be moved between network segments during the provisioning lifecycle.

### Architecture: Lifecycle Owner, Attachment Request, and Networking Controller

```
fulfillment-service → BaremetalInstance CR → hub cluster
       │                                         │
       │                         bare-metal-fulfillment-operator
       │                           - inventory and OS provisioning
       │                           - submits private NetworkAttachment request
       │                           - waits for NetworkAttachmentsReady
       │                           - handoff reboot; waits for IPDiscoveryComplete
       │                           - host power and lifecycle finalizers
       │                                         │
       └→ fulfillment reconciliation creates     │
          internal NetworkAttachment CR ─────────┘
                    │
                    └→ osac-operator NetworkAttachment controller
                          - resolves BMI's existing attachment intent
                          - moves the fabric port and queries DHCP
                          - owns NetworkAttachmentsReady and network status
                          - holds cleanup finalizer through port offboarding

    osac-operator feedback / cleanup controllers
      - BareMetalInstanceFeedbackReconciler
      - fires Signal RPC on status change
      - finalizer: baremetalinstance-feedback (removed last)
      - BareMetalInstance cleanup controller (auto ExternalIP)
```

The nested `BareMetalNetworkAttachment` on the BaremetalInstance CR remains the
sole desired-state input. When BMF reaches the network handoff after
`ProvisionTemplateComplete=True`, it calls the private
`NetworkAttachments.Create` RPC with a `NetworkAttachment` proto identifying
the BMI. Fulfillment reconciliation creates one internal
`NetworkAttachment` CR that references the BMI instead of copying its
subnet/interface fields. The CR is a shared asynchronous work record, not a
second tenant attachment or source of desired state. The networking controller
owns the readiness condition and network status on the BMI; BMF consumes those
results to sequence reboot and readiness.

The private payload is `NetworkAttachment{baremetal_instance: {id: <bmi-id>}}`.
It identifies the BMI only; the nested `BareMetalNetworkAttachment` remains
the desired attachment configuration. The generic proto and private Create RPC
are defined in the [Unified Networking design](/enhancements/OSAC-1433-unified-networking/design.md#shared-workload-attachment-request-and-controller).

### Goals

**Core Design Goals (G1–G5):**

- **G1 — OS-agnostic host networking.** All host addressing via DHCP; no per-OS host-side config. Corollary: exactly one default route at any moment.
- **G2 — Provisioning connectivity.** During inspect + deploy the server can reach its image source and the Ironic conductor.
- **G3 — Correct, minimal final state.** After provisioning: attached to exactly one tenant network segment, single default route, no residual provisioning network.
- **G4 — Isolated until ready.** The tenant reaches the server only after it is fully provisioned and in its final network state; provisioning traffic is never exposed to the tenant.
- **G5 — Achievable on stock metal3 + fabric manager today.**

**Implementation Goals:**

- Single-NIC support with optional physical interface mapping (tenant specifies one interface name from BareMetalInstanceType)
- Resource-specific attachment message (`BareMetalNetworkAttachment`) with `interface` and `primary` fields
- Optional `network_attachments` field — populate with tenant defaults when omitted
- Auto ExternalIP attachment (`auto_external_ip_attachment`) for single-call inbound connectivity
- osac-operator networking controller: after OS provisioning, dispatches the selected interface's fabric port move (provisioning network → tenant), waits for fabric readiness, then queries DHCP after the BMF handoff reboot
- Provisioning network: an idle (unassigned) server keeps its fabric NIC on an OSAC-owned provisioning network (DHCP + gateway + SNAT) so it has internet during metal3 inspection; provisioning moves the port provisioning network → tenant, deletion moves it tenant → provisioning network (see [Provisioning Network and Port Moves](#provisioning-network-and-port-moves))
- IP discovery after handoff: osac-operator networking controller queries the fabric manager's DHCP lease API via dispatcher (`query_dhcp_lease` role), matches the port MAC (resolved from the BareMetalHost `osac.openshift.io/interface-macs` annotation) to the DHCP-assigned IP, writes to the existing CR status, feedback controller syncs to fulfillment-service, and ExternalIPAttachment reads the primary IP for DNAT
- BareMetalInstanceType `network_ports` list with structured port definitions (name, role, type, speed)
- Remove unused `networkClass` field from BareMetalInstance spec entirely (unused per reviewer feedback)

### The Three Network Planes

BMaaS involves three planes; this design owns only the data-plane ones. Do not conflate them.

| Plane | Carries | OS sees it? | Fabric-managed? | Owned by |
|-------|---------|-------------|-----------------|----------|
| **BMC / OOB** | Redfish/IPMI: power, virtual-media | Depends on wiring (dedicated port: no; shared-LOM: yes, own VLAN) | Not for now (separate mgmt network) | metal3 (prerequisite) |
| **Provisioning network** | host DHCP, image download, IPA→conductor callback | Yes (in-band, fabric NIC) | **Yes** | this design |
| **Tenant network** | tenant workload | Yes (fabric NIC after handoff) | Yes | this design |

"OOB" refers only to the BMC plane. The provisioning network is **in-band** (the OS/IPA uses it) — never call it OOB.

**Note on terminology:** In the current code and configuration, the provisioning network is identified as `netris_bm_parking_vnet`. This identifier is retained for deployment stability; the docs-only rename to "provisioning network" clarifies its purpose without requiring immediate config changes.

#### Connectivity Paths

| Path | Network |
|------|---------|
| Ironic → BMC (power/virtual-media) | mgmt/BMC network; conductor routes to BMC IPs |
| IPA → Ironic (callback) | provisioning network |
| Image download | provisioning network → local mirror (or internet) |
| Tenant DHCP + lease discovery | tenant network (fabric manager) |
| Fabric-port network move | fabric manager |

Ironic reaching two planes at once is ordinary **multi-homing**: the conductor host has a NIC/route to each network; it listens on all interfaces for inbound callbacks and the kernel selects egress NIC + source IP per destination. Which network carries the callback is set by the metal3 **`Provisioning` CR** (`provisioningNetwork`, `provisioningIP`/`provisioningInterface`, `virtualMediaViaExternalNetwork`), and each BMC address is per-host on the `BareMetalHost` (`spec.bmc.address`) — none of this is OSAC operator code.

### Non-Goals

- CaaS or VMaaS networking (this EP covers BMaaS only)
- Dispatcher infrastructure implementation (deferred to Unified Networking EP implementation)
- Creating the provisioning network (network segment + DHCP + gateway + SNAT) and the initial per-server attach — a deployment prerequisite handled by the fabric infrastructure / deployment infrastructure, not the operator (see [Provisioning Network and Port Moves](#provisioning-network-and-port-moves))
- Re-provision handoff reset: NetworkHandoffComplete is never reset after initial provisioning, so an in-place re-provision (config-version change after Ready) would run over the tenant network (deferred to long-term design)

## Proposal

### BareMetalInstanceType and Interface Validation

#### BareMetalInstanceType Network Ports

The `BareMetalInstanceType` resource in the fulfillment-service (OSAC-1201) describes a class of bare-metal hardware. For networking, BareMetalInstanceTypes include a structured network ports list:

```protobuf
message BareMetalNetworkPortSpec {
  string name = 1;        // e.g., "data-0", "data-1", "mgmt-0"
  string role = 2;        // e.g., "fabric", "management", "storage", "lifecycle"
  string type = 3;        // e.g., "Ethernet"
  string speed = 4;       // e.g., "100Gbps", "1Gbps"
}
```

`BareMetalInstanceType` is a bare-metal-only resource (OSAC-1201) — BM vs VM is classified by resource type (`BareMetalInstance` vs `ComputeInstance`), not by the contents of `network_ports`. Every `BareMetalInstanceType` must declare at least one `network_ports` entry with `role=fabric`; a bare-metal profile with no fabric port is rejected at creation time because both the operator (provisioning-network port move) and the default-interface resolution (first `role=fabric` port) depend on it. Canonical port-role validation (rejecting unknown or misspelled role values) is owned by OSAC-1201's BareMetalInstanceType schema.

Interfaces are ordered. When multiple interfaces share the same role, the first one in the list is the default for that role (used by CaaS for automatic resolution — see CaaS design).

**Interface-name identity contract.** `BareMetalNetworkAttachment.interface` selects a port by `BareMetalNetworkPortSpec.name`. The operator passes this name as `logical_interface_name` to `move_network_attachment`, and the `osac.openshift.io/interface-macs` annotation keys use the same name. These three identifiers — catalog port name, fabric-manager logical interface, and interface-macs annotation key — must be consistent; a mismatch causes the port move or DHCP lease query to target the wrong NIC. Inventory tooling and BareMetalInstanceType registration must align on the same naming convention.

#### How BMaaS Uses BareMetalInstanceType

The tenant provides `BareMetalNetworkAttachment` with an explicit `interface` field referencing a port `name` from the BareMetalInstanceType's `network_ports` list. The fulfillment-service validates:
- The `interface` name exists in the BareMetalInstanceType's `network_ports` list
- The BareMetalInstanceType is resolved from the instance's `instance_type` field

Unlike CaaS (which picks the interface automatically by role), BMaaS gives the tenant direct control over which physical interface maps to which subnet.

#### BareMetalInstanceType as the Authoritative Model

The [BareMetalInstanceType EP](/enhancements/OSAC-1201-baremetal-instance-types) provides the tenant-facing hardware catalog for BMaaS, with full hardware specs and network port definitions (name, role, type, speed). Per [OSAC-2135 (CaaS Bare-Metal Worker Node Provisioning)](/enhancements/OSAC-2135-caas-bare-metal-worker-provisioning/design.md), `HostType` is deprecated and decommissioned — `BareMetalInstanceType` is now the sole source of truth for hardware profiles and interface resolution:

- BMaaS tenants discover available interfaces via the BareMetalInstanceType API (with type + speed info)
- Interface validation uses BareMetalInstanceType's `network_ports` list
- CaaS fulfillment resolves the fabric interface from `BareMetalInstanceType.network_ports[].role=fabric` at cluster creation and stores it on the node set
- `BareMetalInstanceType.host_label_selector` provides direct inventory matching (OSAC-1201), replacing the former HostType reverse lookup

> **CaaS network attachment source:** For CaaS bare-metal workers, the network attachment originates from the private `ClusterOrder.spec.networkAttachment` (`ClusterNetworkAttachment`) and is enriched per-BMI by the `BareMetalWorkerReconciler` with the immutable `fabric_interface` already resolved and stored on the node set during cluster creation. See [OSAC-2135](/enhancements/OSAC-2135-caas-bare-metal-worker-provisioning/design.md) for the full enrichment flow.

#### Interface Role Convention

| Role | Meaning |
|------|---------|
| `fabric` | Primary fabric traffic (east-west, tenant workloads) |
| `management` | In-band management/control plane traffic |
| `storage` | Storage fabric traffic |
| `lifecycle` | Out-of-band lifecycle management (PXE boot, Redfish/BMC) |

Roles are conventions, not enforced enums. BMaaS uses them for display/documentation; the tenant selects by port name, not role. Ports with role `lifecycle` are used by the provisioning system (Ironic, Metal3) for PXE boot and BMC operations — they are NOT tenant-attachable and should not appear in `network_attachments`.

### Workflow Description

#### Phase 1: Tenant Creates Networking Resources

Same as VMaaS/CaaS — the networking API is uniform.

1. **Create VirtualNetwork:**
   ```bash
   osac create virtualnetwork --network-class moc --cidr 10.0.0.0/16 --name my-net
   ```
   Dispatcher → `osac.templates.{{ fabric_manager }}.create_virtual_network`

2. **Create Subnet:**
   ```bash
   osac create subnet --virtual-network my-net --cidr 10.0.1.0/24 --name my-subnet
   ```
   Dispatcher → fabric_manager creates VLAN/fabric segment. If the NetworkClass has a k8s_manager: also creates CUDN overlay (but BM doesn't use it — the overlay exists for VMs that may share the same subnet).

3. **Create SecurityGroup:**
   ```bash
   osac create security-group --virtual-network my-net --name my-sg \
     --ingress "protocol:tcp,port:443,source:0.0.0.0/0"
   ```
   Dispatcher → `osac.templates.{{ fabric_manager }}.create_security_group`

#### Phase 2: Tenant Creates BM Server

4. **Create BaremetalInstance with network_attachments:**

   Single interface (simple case):
   ```bash
   osac create baremetalinstance --template bcm_h100 \
     --network-attachment interface=data-0,subnet=my-subnet,security-groups=my-sg \
     --name my-server
   ```

   A second `--network-attachment` flag is rejected; multi-NIC BMaaS is not
   supported.

   With defaults + auto external access:
   ```bash
   osac create baremetalinstance --template bcm_h100 \
     --external-ip-attachment --name my-server
   ```

   After provisioning completes, `osac get baremetalinstance` shows the discovered internal IP:
   ```
   ID          NAME       CATALOG ITEM   STATE    INTERNAL IP
   01a0...     my-server  ci-bm-default  RUNNING  10.100.0.2
   ```

5. **fulfillment-service:**
   - If `network_attachments` is omitted or empty: populates the sole attachment with the tenant's default Subnet, default SecurityGroup, and the first port with role `fabric` from `BareMetalInstanceType.network_ports` (see [Default Networking PRD](/enhancements/OSAC-1433-default-networking)).
   - If one attachment is supplied, defaults only missing fields: a missing Subnet receives the tenant default Subnet, a missing or empty SecurityGroup list receives the tenant default SecurityGroup only when the resolved Subnet belongs to the tenant's default VirtualNetwork, and a missing interface receives the first `fabric` port from `BareMetalInstanceType.network_ports`; supplied values are preserved.
   - Validates:
     - At most one network attachment is specified
     - Each subnet exists, is Ready
     - The subnet and each SecurityGroup belong to the same VirtualNetwork
     - The optional `interface` references a valid interface name from the BareMetalInstanceType's network ports list
     - If `interface` is omitted, defaults to the first port with `role=fabric` from the BareMetalInstanceType
     - If one attachment is present, it is the implicit primary; omitted or `primary: true` is accepted but redundant, while `primary: false` is rejected
   - If `auto_external_ip_attachment == true`: auto-selects ExternalIPPool (READY, most available capacity, matching IP family), creates ExternalIP (labeled `osac.openshift.io/auto-created: "true"` and `osac.openshift.io/auto-created-for: <baremetal-instance-id>`) + ExternalIPAttachment (labeled `osac.openshift.io/auto-created: "true"`) in the same DB transaction — both start in **Pending** state. The ExternalIPAttachment references the BaremetalInstance but does not yet have a DNAT target IP; the address is unknown until the networking controller completes DHCP discovery. Pool capacity is decremented atomically; if the pool is exhausted, the API call fails and no resources are persisted (including the BaremetalInstance). See [Unified Networking — Auto-provisioning lifecycle](/enhancements/OSAC-1433-unified-networking/design.md#external-access-same-for-all-resource-types) for the shared two-phase flow.
   - Creates BaremetalInstance CR with `network_attachments` in spec

6. **bare-metal-fulfillment-operator BareMetalInstance controller:**

   a. `reconcileInventory` (unchanged):
      - FindFreeHost → AssignHost (Ironic/Metal3)
      - Populates HostClass from inventory

   b. `reconcileProvisioning` (runs after inventory):
      - Triggers AAP job via `RunProvisioningLifecycle`
      - Template does OS provisioning (PXE boot, user-data, etc.)
      - Server stays on the provisioning network during this phase
      - Host-side networking is handled by DHCP — the template does NOT configure static IPs, gateway, or DNS. The host receives its IP automatically from the provisioning network DHCP server.

   c. Waits for `NetworkAttachmentsReady=True`, set by the osac-operator networking controller after the port move and fabric readiness check.

   d. **`reconcileReboot` (runs after networking):**
      - Issues reboot via BareMetalHost annotation so the OS re-DHCPs on the tenant network
      - Waits for reboot to complete
      - Sets condition: `NetworkHandoffComplete=True`

   e. Waits for `IPDiscoveryComplete=True`, set by the networking controller after it finds the tenant DHCP lease; only then can the BaremetalInstance reach `Ready`.

   f. `reconcilePower` (unchanged)

7. **osac-operator networking controller (after reboot):**
   - Waits for BMF to set `NetworkHandoffComplete=True`, then queries the fabric manager's DHCP lease API via dispatcher (`osac.templates.{{ fabric_manager }}.query_dhcp_lease`). The role queries DHCP leases for the tenant subnet and matches the server's port MAC address (resolved from the BareMetalHost `osac.openshift.io/interface-macs` annotation — see [IP Discovery](#ip-discovery)) to find the corresponding DHCP-assigned IP on the tenant network.
   - Writes the discovered IP to `status.networkAttachmentStatuses[].ipAddress` on the existing BaremetalInstance CR and sets `IPDiscoveryComplete=True`.
   - Feedback controller watches CR status changes → fires Signal RPC to fulfillment-service
   - fulfillment-service reconciler syncs the discovered IP to the DB via existing `syncStatus()` pattern

#### Phase 3: External Access (optional)

8. **Create ExternalIP:**
   ```bash
   osac create externalip --pool external-pool-1 --name my-ip
   ```
   Dispatcher → `osac.templates.{{ fabric_manager }}.create_external_ip`

9. **Create ExternalIPAttachment:**
    ```bash
    osac create externalipattachment --externalip my-ip \
      --baremetal-instance my-server --name bm-att
    ```
    - ExternalIPAttachment controller resolves the BaremetalInstance target by UUID label
    - Checks two preconditions before dispatching (requeues if either is not met):
      1. **ExternalIP must be Allocated** (have an allocated address from the fabric manager)
      2. **BaremetalInstance must have a primary IP** — reads `status.networkAttachmentStatuses[].ipAddress` for the attachment where `primary: true`. This IP is written by the osac-operator networking controller after `NetworkHandoffComplete` and synced to the fulfillment-service via the feedback controller.
    - Once both preconditions are met: writes `osac.openshift.io/target-ip` annotation on the ExternalIPAttachment CR
    - Calls `osac.templates.{{ fabric_manager }}.create_external_ip_attachment`
    - Fabric manager creates DNAT rule: external IP → BM's primary subnet IP
    - ExternalIPAttachment transitions from Pending to Ready

    For auto-provisioned ExternalIPAttachments (`auto_external_ip_attachment=true`), the same flow applies — the attachment is created at API time in Pending state and the controller activates it once the BM's IP becomes known. The wait time depends on the networking controller completing DHCP discovery after the BMF handoff reboot.

#### Deletion (reverse order)

10. **Delete BaremetalInstance:**
    - **Auto-provisioned cleanup (osac-operator):** The osac-operator adds a cleanup finalizer (`osac.openshift.io/baremetalinstance-cleanup`) on BaremetalInstance CRs that have `auto_external_ip_attachment=true`. On deletion, it performs the phased requeue cleanup: deletes ExternalIPAttachment first (by target reference), waits, then deletes ExternalIP (by `auto-created-for` label), waits, then removes its finalizer. See [Unified Networking — Auto-provisioned resource cleanup](/enhancements/OSAC-1433-unified-networking/design.md#external-access-same-for-all-resource-types) for the pattern. This runs concurrently with the bare-metal-fulfillment-operator's deletion flow but does not conflict (different CRs).
    - **Manually created resources are NOT cleaned up** — tenant manages their lifecycle.
    - **Default networking resources (VN, Subnet, SG, NATGateway) are NOT cleaned up** — tenant-scoped and shared.
    - bare-metal-fulfillment-operator and osac-operator networking controller (power-off-first ordering ensures tenant workloads **never** run on the provisioning network):
      - BMF `reconcileNetworkOffboardShutdown` powers off the host **while the port is still on the tenant network**, then sets `NetworkOffboardShutdownComplete=True`. If the host is already powered off, this is a no-op. The condition means it is safe for networking to move the port.
      - The networking controller waits for `NetworkOffboardShutdownComplete=True`, then dispatches the same `osac-move-network-attachment` job to move the fabric NIC **tenant network → provisioning network**. A missing tenant Subnet CR is tolerated; the controller still returns the port to the configured provisioning network when its fabric manager operation permits it. Once the move completes, it sets `NetworkOffboardComplete=True` and removes its `osac.openshift.io/baremetalinstance-networking` finalizer.
      - BMF waits for `NetworkOffboardComplete=True` and the networking finalizer to be removed before `reconcileDeprovisioning`. Ironic then powers the host back on via BMC and PXE-boots a cleaning ramdisk on the provisioning network — not the tenant OS.
      - BMF removes its host-management finalizer after deprovisioning; inventory reconciliation unassigns the host and removes the inventory finalizer.
    - `reconcileInventory` deletion: UnassignHost from Ironic/Metal3, removes inventory finalizer
    - osac-operator feedback controller: waits for other finalizers, removes feedback finalizer, fires final Signal

11. **Tenant deletes networking resources** (independently):
    - Delete ExternalIPAttachments, ExternalIPs, SecurityGroup, Subnet, VirtualNetwork — each via its own dispatcher-triggered delete job

**BMaaS-specific deletion dependency guard:**

The Subnet controller gates its deprovision job on the complete removal
of all BareMetalInstance CRs with `spec.networkAttachments[].subnetRef`
referencing the subnet. This prevents the infrastructure backend from
rejecting the subnet deletion because bare-metal servers are still
attached to it. The guard lists BMI CRs in the namespace and filters by
`subnetRef` in-memory, requeuing every 10 seconds until all BMIs are gone.
See [Unified Networking — Deletion Dependency Guards](/enhancements/OSAC-1433-unified-networking/design.md#deletion-dependency-guards)
for the full guard table covering all networking resources.

**IP discovery lease validation:**

The networking controller requires a valid DHCP lease for the sole attachment
before marking `IPDiscoveryComplete=True`.
If the AAP `query_dhcp_lease` job returns no artifact or no lease for the
selected port MAC, the networking controller records a failed condition and
retries with backoff. BMF keeps the BareMetalInstance from reaching `Ready`
(and therefore `RUNNING`) until `IPDiscoveryComplete=True`. This prevents the
scenario where a DHCP lease is not yet
available (e.g., the fabric manager's DHCP server has not propagated the
lease to the new network segment) and the BMI appears as RUNNING with
no internal IP.

### API Extensions

#### Proto (fulfillment-service)

```protobuf
message BareMetalNetworkAttachment {
  SubnetLocalReference subnet = 1;                         // Optional on input; immutable after resolution
  repeated SecurityGroupLocalReference security_groups = 2; // Optional on input; immutable after resolution
  string interface = 3;                 // optional, immutable: physical interface
                                        // from BareMetalInstanceType
  optional bool primary = 4;            // omitted or true: implicit default gateway; false is rejected
}

message BareMetalInstanceSpec {
  string catalog_item = 1;              // immutable
  optional string ssh_public_key = 2;   // immutable
  optional string user_data = 3;        // immutable
  optional BareMetalInstanceRunStrategy run_strategy = 4;
  int64 restart_trigger = 5;
  map<string, google.protobuf.Any> template_parameters = 6;  // immutable
  optional BareMetalInstanceImage image = 7;                  // immutable
  repeated BareMetalNetworkAttachment network_attachments = 8; // NEW, optional; max 1 for compatibility
  bool auto_external_ip_attachment = 9;  // NEW, auto-provision ExternalIP + ExternalIPAttachment
}

message BareMetalInstanceStatus {
  // ... existing fields ...
  repeated BareMetalNetworkAttachmentStatus network_attachment_statuses = N; // NEW
}

message BareMetalNetworkAttachmentStatus {
  string interface = 1;
  string subnet_ref = 2;
  string ip_address = 3;  // Discovered after DHCP assignment, synced to fulfillment-service via feedback
  bool primary = 4;       // true for the sole resolved attachment; normalized from spec
}
```

The existing private `BareMetalNetworkAttachment` message remains nested in
`BareMetalInstanceSpec`. This design adds no top-level attachment proto/service
and no standalone attachment CRD. The existing BMI attachment is the sole
desired-state object; `NetworkAttachmentStatuses`, conditions, and job history
remain on that CR. The division of responsibility changes between controllers
without adding another fulfillment-service API surface.

#### BaremetalInstance CRD (served to both controllers)

```go
type BareMetalInstanceSpec struct {
    // ... existing fields ...
    NetworkAttachments []BareMetalNetworkAttachment `json:"networkAttachments,omitempty"`
}

type BareMetalNetworkAttachment struct {
    SubnetRef         string   `json:"subnetRef"`
    SecurityGroupRefs []string `json:"securityGroupRefs,omitempty"`
    Interface         string   `json:"interface,omitempty"`
    Primary           bool     `json:"primary,omitempty"`
}

type BareMetalInstanceStatus struct {
    // ... existing fields ...
    NetworkAttachmentStatuses []BareMetalNetworkAttachmentStatus `json:"networkAttachmentStatuses,omitempty"`
}

type BareMetalNetworkAttachmentStatus struct {
    Interface  string `json:"interface,omitempty"`
    SubnetRef  string `json:"subnetRef,omitempty"`
    IPAddress  string `json:"ipAddress,omitempty"` // Discovered after DHCP assignment
    Primary    bool   `json:"primary,omitempty"` // true for the sole resolved attachment
}
```

CEL immutability: the `network_attachments` list and every field in each
attachment are immutable after creation, including `subnetRef`,
`securityGroupRefs`, `interface`, and `primary`.

CEL validation rule:
```yaml
- rule: "self.networkAttachments.size() <= 1"
message: "at most one network attachment is supported"
```

The CRD remains the shared BaremetalInstance API used by BMF and
osac-operator; handing reconciliation to osac-operator does not introduce a
second CRD. The max-one CEL rule is required before enabling BMaaS network
attachments.

#### fulfillment-service Controller (mutateBMI)

The `mutateBMI()` function in the fulfillment-service's BM reconciler currently sets TemplateID, TemplateParameters, and RunStrategy on the K8s CR. It must also copy the sole `network_attachments` entry from the proto spec to the K8s CR spec. Before persistence, it normalizes the sole attachment to `primary: true` (the implicit-primary rule); a single attachment is never persisted with `primary: false`.

#### Server Validation Rules

- The API and CRD validators reject more than one network attachment. The
  single-attachment contract is defined here; implementing and enabling this
  validation is a prerequisite for rollout.
- An omitted or empty list receives the tenant defaults; a supplied single entry receives defaults only for missing fields
- The resolved subnet and security groups must belong to the same VirtualNetwork
- The `interface` must reference a valid port name from the BareMetalInstanceType (its network ports list defines available ports)
- Interfaces with role `lifecycle` are rejected in `network_attachments` — lifecycle interfaces (PXE boot, BMC) are reserved for the provisioning system and are not tenant-attachable
- If `interface` is omitted: defaults to the first port with `role=fabric` from the BareMetalInstanceType (consistent with the omitted-list default)
- If a single attachment is present: `primary` is implicit; omitted or `true` is accepted and `false` is rejected
- The complete resolved `network_attachments` list is immutable after creation; changing it requires deleting and recreating the BaremetalInstance

### Implementation Details/Notes/Constraints

#### Provisioning Network and Port Moves

Bare-metal servers configure host-side networking entirely via DHCP. An
**unassigned** server (owned by no tenant) has no tenant network segment, so without
intervention it has no default gateway and no internet — which breaks the Ironic
Python Agent (IPA) during metal3 inspection/cleaning (it cannot download its
rootfs). Hanging the default gateway off the management/BMC NIC is not an option:
the host would then have two DHCP default routes (management + fabric) once a
tenant subnet is attached, causing a default-gateway race.

**Solution — a provisioning network.** A fabric manager **provisioning network
segment** (DHCP + default gateway + SNAT for outbound internet) holds every
server's **fabric NIC** while the server is idle and during provisioning, so an
unassigned server always has internet via its fabric NIC. The provisioning
network exists **only in the fabric manager** — it has no OSAC Subnet CR — and
its name is a fabric-manager configuration value (`netris_bm_provisioning_vnet`,
sourced from the `NETRIS_BM_PROVISIONING_VNET` environment variable, with
backward-compatible fallback to `NETRIS_BM_PARKING_VNET`), not operator state.

**Provision and deprovision are the same primitive: move a fabric port from one
network segment to another.** The port lifecycle is:

| Flow | Trigger | Move (from → to) | When |
|------|---------|------------------|------|
| Initial | Deployment bootstrap (deployment infrastructure) | — → provisioning network | Pre-deployment |
| Provision | osac-operator networking controller after `ProvisionTemplateComplete` | provisioning network → tenant subnet's network segment | **POST-provisioning** |
| Deprovision | BMI deletion (networking cleanup) | tenant subnet's network segment → provisioning network | Deletion |

**Key difference from the previous design:** The port move now happens **AFTER
provisioning is complete**, not before. The server is provisioned while on the
provisioning network, then moved to the tenant network and rebooted so the OS
re-DHCPs there. This achieves isolation-until-ready (G4): the tenant cannot reach
the server during imaging/first-boot, and provisioning traffic (image
pull/cloud-init) never traverses the tenant network.

Creating the provisioning network (network segment + DHCP + gateway + SNAT) and
performing the initial per-server attach are deployment prerequisites (handled by
the fabric infrastructure / deployment infrastructure), not operator responsibilities. The
provisioning network name configured for the fabric manager must match the one
used at bootstrap.

**Generic `move_network_attachment` role.** The fabric manager exposes a single
generic primitive, keyed on plain network segment **names**:

```
move_network_attachment(host_name, logical_interface_name,
                        from_vnet_name, to_vnet_name)
    → detach the server's fabric port from from_vnet_name (if set),
      then attach it to to_vnet_name (if set)
```

- The role resolves host → fabric server → fabric port, then detaches from the
  source network segment and attaches to the target. Either side may be empty (a
  pure attach or pure detach).
- Detach is a **no-op when the port is not on the named segment** (robust to
  retries and unexpected state); attach fails if the target segment or port
  cannot be resolved.
- It operates purely against the fabric manager — **no Subnet CR lookup inside
  the role**. Callers resolve a `subnetRef` → tenant network segment name and
  pass the provisioning network name from configuration.
- The primitive is backend-/lifecycle-agnostic: callers decide what the segments
  mean (tenant, provisioning, …), so CaaS can reuse it for its own
  provisioning-network flow.

**Single move playbook, direction from the CR.** One AAP job template
(`osac-move-network-attachment`, playbook
`playbook_osac_move_network_attachment.yml`) serves both provision and
deprovision. It derives direction from the CR: a resource carrying
`metadata.deletionTimestamp` is **offboarding** (tenant → provisioning network);
otherwise it is **onboarding** (provisioning network → tenant). The tenant network
segment is resolved from the sole attachment's `subnetRef` (Subnet CR `metadata.name`
== fabric network segment name); the provisioning network name comes from
configuration. The
osac-operator networking controller uses the shared networking dispatcher and
the same `osac-move-network-attachment` template for both directions. It
resolves the sole attachment's subnet through the existing private Subnet,
VirtualNetwork, and NetworkClass APIs, then dispatches through the resolved
fabric manager. BMF does not create a networking AAP job or resolve
NetworkClass for this operation.

#### Controller Boundary and Status Ownership

The networking controller watches the internal `NetworkAttachment` CR created
from BMF's private fulfillment-service request. The CR references the existing
BaremetalInstance and does not copy its nested `spec.networkAttachments`; that
field remains the sole desired-state source. The private proto and CR form the
same internal request contract used by VMaaS and CaaS, with one work record per
target attachment. For a BMI, the controller adds
`osac.openshift.io/baremetalinstance-networking` before starting any AAP job
and holds that finalizer until tenant-network cleanup is complete. It uses the
shared NetworkClass resolver with dedicated providers for
`osac-move-network-attachment` and `osac-query-dhcp-lease`; the resolver selects
the manager-specific role for each job.

BMF submits the request after `ProvisionTemplateComplete=True`. The networking
controller owns creation and updates of `NetworkAttachmentsReady` on BMI
status. When it first reconciles the internal request CR, it creates the
condition as Unknown while the port move is pending; it sets False with a
reason if the operation fails, and True only after the fabric manager reports
the target segment ready. BMF does not create or set that condition; it
consumes it to gate the handoff reboot.

| Owner | Status and lifecycle fields | Purpose |
|---|---|---|
| bare-metal-fulfillment-operator | Inventory/provisioning/power status; `ProvisionTemplateComplete`; `NetworkHandoffComplete`; `NetworkOffboardShutdownComplete`; private `NetworkAttachment` request at the BMaaS handoff | Owns host lifecycle, submits the request after provisioning, and reports when the handoff reboot or safe-to-offboard shutdown has completed. It consumes networking-owned readiness. |
| osac-operator NetworkAttachment controller | Internal `NetworkAttachment` CR; `NetworkAttachmentsReady`; `IPDiscoveryComplete`; `NetworkOffboardComplete`; `NetworkAttachmentStatuses`; `NetworkingJobs`; `IPDiscoveryJobs`; `osac.openshift.io/baremetalinstance-networking` finalizer | Owns fabric attachment and DHCP discovery, records job history and the discovered address, writes the BMI readiness condition, and prevents deletion from passing network cleanup. |
| osac-operator feedback controller | Feedback finalizer and fulfillment-service Signal RPC | Observes the combined status and synchronizes it to the fulfillment-service. |

`NetworkOffboardComplete` means the fabric port has returned to the provisioning
network. It does not mean only that the host has shut down. The networking
controller sets it after the move succeeds; BMF sets
`NetworkOffboardShutdownComplete` after powering off the host and waits for
`NetworkOffboardComplete` before deprovisioning or releasing the host.

Both operators update the same status subresource. Each must re-read the latest
object on conflict and merge only the fields and condition types it owns. In
particular, a full `latest.Status = newStatus` replacement by BMF would erase
networking-owned job history, IP status, or conditions written concurrently;
that write pattern must be replaced with field-owned, conflict-safe updates
before enabling the networking controller. Conditions are merged by condition
type so one controller cannot remove conditions owned by the other.

#### Topology-Agnostic Operator (Transport is Environment Config)

The controllers divide responsibilities by domain. The osac-operator
networking controller owns **network segment membership and DHCP discovery**,
referenced by segment **name** (config/CR), never by physical transport. BMF
owns host provisioning, image changes, power, and the BMH-annotation reboot
used for the network handoff. The networking controller does not configure
host-side networking.

The **transport** is environment config, not code:

- image source = `BareMetalHost.spec.image.url` / template param (local mirror or internet via provisioning network),
- callback/PXE/DHCP network = the metal3 `Provisioning` CR (`Managed`/`Unmanaged`/`Disabled` depending on deployment).

The operator must never assume the provisioning network carries the
image/callback (no egress checks, no SNAT logic).

#### Assumptions

- Single fabric NIC per server on the fabric (moves provisioning↔tenant).
- BMC reachability (Ironic↔BMC) is a deployment prerequisite on a tenant-isolated
  mgmt network; not fabric-managed for now.
- The provisioning network (network segment + DHCP + gateway + egress) and the
  initial per-server attach are deployment prerequisites (deployment infrastructure / inventory
  tooling), as today.
- Inventory tooling sets the `osac.openshift.io/interface-macs` annotation for
  the tenant NIC.
- **Self-contained images**: first boot needs no *tenant-side* internet
  (first-boot egress happens on the provisioning network during boot #1). A
  robust "cloud-init done" signal is a long-term item.

#### Tenant Handoff Signaling

The controllers use conditions to hand off work without duplicating AAP
operations:

- After BMF reports `ProvisionTemplateComplete=True`, BMF submits the private
  `NetworkAttachment` request. Fulfillment reconciliation creates the internal
  CR; the osac-operator networking controller starts reconciliation from it.
- The networking controller sets `NetworkAttachmentsReady` after the tenant
  port move and fabric readiness wait. BMF consumes this condition; it does not
  create or set it.
- BMF waits for `NetworkAttachmentsReady`, reboots the host so its OS re-DHCPs,
  then sets `NetworkHandoffComplete`.
- The networking controller waits for `NetworkHandoffComplete`, discovers the
  tenant DHCP lease, writes `NetworkAttachmentStatuses`, and sets
  `IPDiscoveryComplete`.
- BMF waits for `IPDiscoveryComplete` before reporting the instance `Ready`.
- During deletion, BMF sets `NetworkOffboardShutdownComplete` after power-off;
  the networking controller returns the port to the provisioning network,
  sets `NetworkOffboardComplete`, and removes its finalizer.

The networking controller owns `NetworkAttachmentsReady`,
`IPDiscoveryComplete`, `NetworkOffboardComplete`, `NetworkAttachmentStatuses`,
`NetworkingJobs`, `IPDiscoveryJobs`, and the networking finalizer. BMF owns
`NetworkHandoffComplete`, `NetworkOffboardShutdownComplete`, and host lifecycle
status. **Gating rule:** neither operator reports `Ready` or surfaces a tenant
IP until the port move, fabric readiness, handoff reboot, and DHCP discovery
have all completed. The provisioning-network IP is never exposed to the tenant.
External access is signaled separately by the `ExternalIPAttachment` (DNAT) and
`NATGateway` (SNAT) CR statuses.

#### IP Discovery

IP discovery is decoupled from switch port configuration. The
`move_network_attachment` role is switch-side only — it moves the server's
fabric port onto the tenant subnet's network segment after OS provisioning and
before the handoff reboot. It does not query DHCP leases or return an IP
address.

After BMF sets `NetworkHandoffComplete=True`, the osac-operator networking
controller dispatches `osac.templates.{{ fabric_manager }}.query_dhcp_lease`,
passing the sole attachment's subnet reference and selected port MAC address.
The role queries the fabric manager's DHCP lease API for the subnet, matches
the port MAC to find the DHCP-assigned IP, and returns it. The networking
controller writes the discovered IP to
`status.networkAttachmentStatuses[].ipAddress` and sets
`IPDiscoveryComplete=True` on the BaremetalInstance CR.

**MAC resolution — the `osac.openshift.io/interface-macs` contract.** Bare-metal servers are not registered as named fabric servers, so their DHCP leases appear in the fabric manager's IPAM as MAC-only host entries (no server name). To match a lease, the networking controller reads the selected NIC MAC from a JSON map of OSAC interface name → NIC MAC on the associated `BareMetalHost`, e.g. `{"eth9":"52:54:00:16:04:83"}`, under the `osac.openshift.io/interface-macs` annotation. It resolves the sole attachment's interface to a MAC and passes it to the job as an extra var. The `query_dhcp_lease` role matches the IPAM host by MAC (the fabric manager stores lease MACs lowercase; the role compares against the lowercased `mac[].address` values). When no MAC is supplied, the role falls back to matching by server name — the path named CaaS fabric servers use. The osac-operator service account therefore needs read access to the `BareMetalHost` annotation; inventory tooling must populate it before DHCP discovery.

The networking controller writes both the discovered IP and `primary: true` to the status entry for the resolved attachment (the status-side reflection of the implicit-primary rule). The feedback controller syncs this to the fulfillment-service DB via the existing Signal / `syncStatus()` pattern, and the ExternalIPAttachment controller has one deterministic IP to read for DNAT creation.

#### Component Responsibility Summary

| Component | Responsibility |
|-----------|---------------|
| fulfillment-service | Validate network_attachments, create CR, copy the single resolved attachment to the existing K8s CR via mutateBMI, auto-provision ExternalIP |
| bare-metal-fulfillment-operator | Inventory assignment, OS provisioning (AAP), waits on networking-owned conditions, performs the handoff reboot and offboard shutdown, manages host power |
| osac-operator networking controller | Resolves the attachment's NetworkClass, dispatches the port move and DHCP lease query, owns network conditions/status/job history and the networking finalizer |
| AAP BM provisioning template | OS provisioning only (host-side networking handled by DHCP) |
| osac-operator feedback controller | Signal fulfillment-service on status changes (unchanged), sync IP addresses from CR status to DB |
| osac-operator BMI cleanup controller | Clean up auto-provisioned ExternalIPAttachment → ExternalIP on BaremetalInstance deletion (phased requeue, `baremetalinstance-cleanup` finalizer) |
| osac-operator ExternalIPAttachment controller | Read BM's primary IP from CR status, create DNAT via fabric_manager |
| fabric_manager role (`move_network_attachment`) | Switch-side only: resolve host → fabric server → fabric port, detach from the source network segment (if set) and attach to the target segment (if set). Waits for target segment active state after attach. The osac-operator networking controller calls it for both provisioning → tenant and tenant → provisioning moves. |
| fabric_manager role (`query_dhcp_lease`) | Query fabric manager's DHCP lease API for a subnet, match the port MAC (or fall back to server name) to find the DHCP-assigned IP, return it to the osac-operator networking controller. |

#### Reconciliation Phase Ordering

**Target reconcile flow (provision-then-handoff):**

```
BMF BareMetalInstance controller:
1. reconcileInventory → allocate host, populate HostClass
   Sets condition: InventoryAssigned=True

2. reconcileProvisioning → OS provisioning (AAP). Server stays on the provisioning network.
   Host PXE boots and gets IP from DHCP on the provisioning network.
   Requires: InventoryAssigned=True
   Sets condition: ProvisionTemplateComplete=True
   Submits private NetworkAttachment proto request for this BMI
   Fulfillment reconciliation creates the internal NetworkAttachment CR

osac-operator NetworkAttachment controller (internal request CR):
3. Resolve the referenced BMI's Subnet → VirtualNetwork → NetworkClass,
   add the networking finalizer to the BMI,
   then dispatch move_network_attachment: provisioning network → tenant network
   Wait for target segment active; record NetworkingJobs.
   Requires: ProvisionTemplateComplete=True
   Sets condition: NetworkAttachmentsReady=True

BMF BareMetalInstance controller:
4. reconcileReboot → reboot server (BMH annotation) so OS re-DHCPs on tenant network
   Requires: NetworkAttachmentsReady=True
   Sets condition: NetworkHandoffComplete=True

osac-operator networking controller:
5. Query fabric manager's DHCP lease API via dispatcher (query_dhcp_lease),
   match port MAC from the BareMetalHost annotation, and write the IP to CR status.
   Requires: NetworkHandoffComplete=True
   Sets condition: IPDiscoveryComplete=True

BMF BareMetalInstance controller:
6. Phase Ready → fully provisioned + on tenant network + IP known
   Requires: IPDiscoveryComplete=True

7. reconcilePower → power state management (independent)

Deletion (power-off-first — tenant workloads never touch provisioning network):
1. BMF reconcileNetworkOffboardShutdown → power off while port is on tenant network
   Sets condition: NetworkOffboardShutdownComplete=True
2. Networking controller waits for shutdown condition, moves port tenant network →
   provisioning network, sets NetworkOffboardComplete=True, removes network finalizer
3. BMF waits for NetworkOffboardComplete and finalizer removal, then
   reconcileDeprovisioning → Ironic PXE boots cleaning ramdisk (not tenant OS)
4. reconcileInventory (delete) → unassign host
```

Only the networking controller starts or polls `move_network_attachment` and
`query_dhcp_lease` jobs. BMF only consumes the networking controller's
conditions to order its host lifecycle operations.

The server sits on the **provisioning network** (config identifier `netris_bm_provisioning_vnet`, with DHCP + gateway + egress) from bootstrap through the entire metal3 deploy and first boot. First-boot cloud-init runs there **with egress**, so first-boot pulls succeed. Only after `ProvisionTemplateComplete` does the operator move the fabric port to the tenant network (waiting for the network segment to reach active state) and perform the handoff reboots so the OS re-DHCPs on the tenant network (see DHCP lease handoff below).

**DHCP lease handoff — deterministic second reboot.** Moving the fabric port from the provisioning network to the tenant network moves the host's NIC to the tenant V-Net, so the host must obtain a fresh DHCP lease there. This does not complete on the first post-switch reboot; a second reboot is deterministically required before the host holds a tenant-V-Net lease. This is expected, deterministic behavior — not a timing or race condition. BMF performs the second reboot as part of the handoff after `NetworkAttachmentsReady`; the networking controller queries the DHCP lease only after BMF sets `NetworkHandoffComplete`. (Fabric managers that scope DHCP strictly per segment may not require the second reboot.)

### Security Considerations

This feature inherits the existing security model:
- Tenant isolation via `osac.openshift.io/tenant` annotation enforced by OPA policies
- Auto-provisioned resources (ExternalIP, ExternalIPAttachment) inherit tenant annotation from parent BaremetalInstance
- No new authentication or authorization changes
- SecurityGroup rules control BM inbound traffic (tenant-configurable via explicit SG or default SG)
- The single BM network attachment uses the same SecurityGroup enforcement as the rest of the fabric

### Failure Handling and Recovery

#### BMF Host Lifecycle Reconciliation Failures

- Inventory assignment failure (no free hosts): BaremetalInstance enters Failed state with condition, retries when host becomes available
- OS provisioning, reboot, or power failure: BMF records the failing host-lifecycle condition/job and does not advance to the next gate.

#### osac-operator Networking Controller Failures

- Port-move dispatch or fabric-readiness failure: the networking controller records the AAP job in `NetworkingJobs`, sets `NetworkAttachmentsReady=False` with a failure reason, and retries after the operator or fabric issue is corrected. BMF waits on this condition and does not reboot the host.
- DHCP query returns no lease or cannot read the selected interface MAC: the networking controller records the job in `IPDiscoveryJobs`, sets `IPDiscoveryComplete=False`, and retries with backoff. BMF does not report `Ready` until discovery succeeds.
- Offboard move failure: the networking finalizer remains on the deleting BMI, preserving the host and preventing BMF from deprovisioning or releasing it before port cleanup succeeds.
- Controller restart during an AAP operation: the controller resumes polling the job ID in the existing job-history status field; all AAP operations must remain idempotent when retried.

#### Auto ExternalIP Allocation Failures

- Pool exhaustion: create API call returns error, no resources persisted (pool capacity checked synchronously during the API call — see [auto-provisioning lifecycle](/enhancements/OSAC-1433-unified-networking/design.md#external-access-same-for-all-resource-types))
- ExternalIP provisioning failure: ExternalIP enters Failed state, BaremetalInstance remains in Pending (external access unavailable, BM may still function without inbound connectivity)
- ExternalIPAttachment provisioning failure: DNAT rule not created, inbound traffic does not reach BM (BM functional, external access unavailable)

#### Cleanup Failures

- Auto-provisioned resource cleanup transient failure: finalizer retries
- Auto-provisioned resource cleanup permanent failure: after N retries, finalizer is removed, parent resource deleted, orphaned ExternalIP/ExternalIPAttachment left in cluster (manual cleanup required)

### RBAC / Tenancy

The osac-operator networking controller needs permission to watch the
BaremetalInstance CR and update its status and finalizers; read the associated
BareMetalHost to obtain `osac.openshift.io/interface-macs`; and read the
networking resources needed to resolve the sole attachment and NetworkClass.
It uses the existing fulfillment-service private APIs and shared dispatcher
credentials. Its BMI watch is restricted to the configured BM namespace, and
it writes only networking-owned status fields and finalizers. BMF no longer
needs Subnet/NetworkClass access or dispatcher permissions for BM port moves
and DHCP discovery.

All new resources (BaremetalInstance with new fields, auto-provisioned ExternalIP/ExternalIPAttachment) inherit tenant isolation from parent:
- `osac.openshift.io/tenant` annotation propagated from BaremetalInstance to auto-created resources
- OPA policies enforce tenant-scoped operations according to each resource API;
  networking resources use create/list/get/delete and do not expose
  update/patch, while supported non-network workload updates remain available
- Tenant User can view and manage auto-provisioned resources (labeled `osac.openshift.io/auto-created: "true"`) via standard API

### Observability and Monitoring

New structured log events:
- osac-operator networking controller: `BareMetalNetworkAttachmentReconciling`, `BareMetalNetworkAttachmentReady`, `BareMetalIPDiscovered`, `BareMetalNetworkOffboarded` (info), `BareMetalNetworkingFailed` (error)
- bare-metal-fulfillment-operator: lifecycle/provisioning, handoff reboot, and offboard shutdown events (info/error); it does not log or own AAP port-move or DHCP-query jobs
- fulfillment-service: `AutoProvisionedExternalIP` (info), `ExternalIPPoolExhausted` (error), `InterfaceValidationFailed` (error)

New Kubernetes events on BaremetalInstance:
- osac-operator emits `NetworkingConfigured`, `IPAddressDiscovered`, and `NetworkingConfigurationFailed` for its port-move and DHCP operations
- BMF emits handoff-reboot and offboard-shutdown events for host lifecycle transitions
- `AutoExternalIPCreated`: ExternalIP and ExternalIPAttachment auto-provisioned

No new metrics or alerts (existing provisioning duration and failure rate metrics apply).

### Risks and Mitigations

#### Risk: fabric_manager implementation blocked or delayed

**Impact:** The fabric manager `move_network_attachment` role and a provisioned provisioning network segment are prerequisites for BMaaS networking. Without them, switch port configuration cannot function.

**Mitigation:** Prioritize Netris BM roles (OSAC-2081). Accept that BMaaS remains unavailable until a fabric_manager exists. Document as a hard dependency.

**Reviewed by:** Engineering / Product

#### Risk: ExternalIPPool exhaustion

**Impact:** Auto ExternalIP allocation fails, create API call returns error, tenant cannot create BM with `auto_external_ip_attachment=true`.

**Mitigation:** Pool capacity visible in status; clear error directs tenant to explicit allocation from another pool or contact admin.

**Reviewed by:** Cloud Provider Admin

#### Risk: Concurrent controllers update the BaremetalInstance status

**Impact:** BMF, the networking controller, and the feedback/cleanup controllers watch or update the same BaremetalInstance CR. A status replacement or condition-list overwrite can erase another controller's job history, IP, or lifecycle condition; a deletion race can release a host while its switch port remains on the tenant network.

**Mitigation:** Assign each status field, condition type, and finalizer to one owner; merge updates against the latest resource version and merge conditions by type. Gate host lifecycle transitions on networking-owned conditions and hold the networking finalizer until the offboard move completes. Cover conflicts, restarts, and deletion ordering in integration tests.

**Reviewed by:** osac-operator / bare-metal-fulfillment-operator teams

### Drawbacks

#### Coordinating lifecycle and networking controllers

The BMF and osac-operator networking controller coordinate through conditions,
job-history fields, and a finalizer on the same CR. This adds a status ownership
contract and cross-controller failure modes.

**Trade-off:** Networking ownership matches the networking API and shared
dispatcher, while host lifecycle ownership remains with the Ironic/Metal3
operator. Conflict-safe status merging, condition ownership, and explicit
finalizer ordering are required to preserve that separation.

## Alternatives (Not Implemented)

### Alternative 1: Keep all BM networking operations in BMF

Leave port moves and DHCP queries in BMF, which already owns BM provisioning and host lifecycle operations.

**Rejected because:** it gives the BM lifecycle operator ownership of fabric operations, DHCP lease lookup, and networking AAP job history, duplicating networking responsibilities already centralized in osac-operator. BMF remains the source of host lifecycle sequencing but consumes networking-owned conditions.

### Alternative 2: Operator IPAM (pre-allocate IPs)

The operator pre-allocates IPs from the subnet CIDR and writes static config (IP, gateway, prefix, DNS) to CR status. The host provisioning template applies that configuration.

**Rejected because:** DHCP is simpler, OS-agnostic, and already provided by the fabric infrastructure. Static config requires per-OS template logic (cloud-init, NMState, kickstart) and adds IPAM complexity (allocation tracking, cross-operator concurrency, gateway/DNS discovery). DHCP handles all of this automatically.

### Alternative 3: Reconcile BaremetalInstance directly without an internal request CR

Let the networking controller watch the BaremetalInstance directly and omit the
shared private `NetworkAttachment` proto and internal CR.

**Rejected because:** VMaaS, CaaS, and BMaaS need one explicit asynchronous
handoff to the networking owner, while their workload CRs and lifecycle gates
differ. The internal CR provides that common request/status boundary and
references the workload; it does not duplicate the nested attachment intent or
create a second tenant-facing attachment.

## Open Questions

### ~~1. Should auto NATGateway treat a Deleting NATGateway as 'does not exist'?~~ — Resolved

Resolved: NATGateway reuse limited to Ready only. Failed/Deleting NATGateways cause the create request to fail with an error. NATGateway auto-provisioning per resource was removed — NATGateway is now a VN default created at tenant onboarding.

### ~~2. Should capacity exhaustion return an API error or create a Failed resource?~~ — Resolved

Resolved: Return error, no resource persisted. Pool capacity checked synchronously. No Failed resource.

### ~~3. IP address assignment~~ — Resolved

Resolved: DHCP handles IP assignment. The host receives its IP from the fabric's DHCP server after booting on the network segment. No operator IPAM needed.

### ~~4. How is the host's runtime IP discovered after network reconfiguration?~~ — Resolved

Resolved: After BMF sets `NetworkHandoffComplete=True`, the osac-operator networking controller queries the fabric manager's DHCP lease API via dispatcher (`query_dhcp_lease`). It matches the server's port MAC — resolved from the BareMetalHost `osac.openshift.io/interface-macs` annotation — to find the assigned IP (falling back to server-name matching for named fabric servers), writes to `status.networkAttachmentStatuses[].ipAddress`, and sets `IPDiscoveryComplete=True`. The feedback controller syncs the status to fulfillment-service via Signal RPC. `move_network_attachment` remains switch-side only (moves the fabric port between network segments).

## Test Plan

### Unit Tests

- fulfillment-service: max-one attachment and primary validation (accept single implicit primary, accept explicit primary)
- fulfillment-service: omitted and partial attachment defaulting (empty `security_groups` is missing; supplied values are preserved; a missing group list defaults only for the tenant default VirtualNetwork and is rejected for a non-default subnet without caller-supplied groups)
- fulfillment-service: interface validation (reject an interface not in BareMetalInstanceType)
- fulfillment-service: auto ExternalIP pool selection (pick READY pool with most capacity, respect IP family)
- bare-metal-fulfillment-operator: waits for `NetworkAttachmentsReady`, performs the handoff reboot, sets `NetworkHandoffComplete`, and waits for `IPDiscoveryComplete` before Ready
- osac-operator: attachment controller dispatches one `move_network_attachment` job only after provisioning completion, resolves the correct fabric manager, and waits for target-segment readiness
- osac-operator: DHCP discovery waits for `NetworkHandoffComplete`, resolves the sole interface MAC from the BareMetalHost annotation, and writes the primary IP and `IPDiscoveryComplete`
- osac-operator: offboard waits for BMF shutdown, returns the port to provisioning, sets `NetworkOffboardComplete`, and removes the networking finalizer before BMF deprovisions the host
- osac-operator and BMF: concurrent status updates preserve the other controller's status fields and conditions under resource-version conflicts

### Integration Tests

- E2E: create BaremetalInstance with two attachments, verify the API rejects the request
- E2E: create BaremetalInstance with `--external-ip-attachment`, verify auto ExternalIP + ExternalIPAttachment created, DNAT rule functional
- E2E: delete BaremetalInstance with auto-provisioned resources, verify ExternalIPAttachment and ExternalIP cleaned up
- E2E: create BaremetalInstance with interface not in BareMetalInstanceType, verify error returned
- E2E: create BaremetalInstance with a second `--network-attachment`, verify the CLI and API return a maximum-one error
- E2E: verify IP discovery (`query_dhcp_lease` role queries fabric manager DHCP lease API after BMF handoff reboot, matches port MAC to assigned IP on tenant network, osac-operator writes CR status, feedback controller syncs to fulfillment-service, ExternalIPAttachment reads primary IP)
- E2E: verify the port move and reboot flow — create BMI provisions on the provisioning network, then moves the fabric port provisioning network → tenant network + reboots; delete BMI returns it tenant → provisioning network (confirm in fabric manager; a freed server can re-inspect with internet)
- E2E: verify isolation-until-ready — before the move, a tenant vantage cannot reach the server; after move + reboot, it can, and the server is no longer on the provisioning network

### Tricky Test Cases

- BM with one attachment and omitted `primary` (verify the sole attachment is normalized to `primary: true` in the CR/status and is the default route)
- ExternalIPPool exhaustion (verify error returned, no resource created)
- Auto-provisioned resource cleanup failure (verify finalizer retry, eventual orphan cleanup)
- IP address feedback latency (verify ExternalIPAttachment controller waits for IP to appear in status)
- Delete while provisioned on tenant network (verify BMF powers off first; the networking finalizer remains until the port move completes; BMF does not deprovision/release the host early)
- Legacy `NetworkOffboardComplete=True` on an upgraded BMI (verify the networking controller does not interpret the old shutdown-only meaning as proof the port was returned)

## Long-Term Evolution (The Reboot is the Seam)

The structure `inventory → provision → establish-tenant-networking → discovery`
stays; only "establish-tenant-networking" changes:

**Ironic Standalone Networking** (metal3-docs PR #586; BMO PR #3469, ToR
networking part 1): Ironic switches the port VLAN per lifecycle phase
(provisioning→tenant) on one NIC — drop the reboot. Upstream assumes
`networking-generic-switch` (direct ToR control), which does not fit
The fabric manager owns the switch; adopting a new backend means implementing the fabric manager interface or aligning the model.

**Dedicated always-on provisioning NIC** (two NICs): attach the tenant NIC after
`provisioned`; no move of the provisioning NIC. Needs 2 NICs, a local registry,
gateway-less provisioning DHCP, and provisioning-network isolation.

**Static host-side networking + IPAM** (`networkData`): one boot, but not
OS-agnostic and reintroduces IPAM. Opt-in fast path for capable, self-contained
images.

Each evolution replaces just the reboot step while preserving the same overall
flow and operator structure.

## Graduation Criteria

**Note:** This section will be updated when the enhancement is targeted at a release.

Proposed maturity level: **Tech Preview** → **GA**

Tech Preview criteria:
- [ ] API fields (`network_attachments`, `auto_external_ip_attachment`) implemented in fulfillment-service
- [ ] BaremetalInstance CRD updated with `NetworkAttachments` field, CEL validation, and status field for IP addresses
- [ ] osac-operator BaremetalInstance networking controller implemented (shared dispatcher, provision-then-handoff port move, fabric readiness wait, DHCP lease query)
- [ ] bare-metal-fulfillment-operator `reconcileReboot` phase implemented (BMH annotation-based reboot after port move)
- [ ] BMF waits on networking-owned conditions and does not dispatch BM port moves or DHCP queries
- [ ] Networking/BMF status updates merge owned fields and conditions safely under concurrent writes
- [ ] Networking controller owns the networking finalizer and offboard move; BMF waits before deprovisioning/releasing the host
- [ ] Dispatcher integration for `move_network_attachment` (provision + deprovision via one job template); provisioning network provisioned and initial per-server attach done at deployment
- [ ] BareMetalInstanceType with network ports (`BareMetalNetworkPortSpec`) available and tested
- [ ] Auto ExternalIP attachment provisioning functional
- [ ] IP discovery implemented in osac-operator (`query_dhcp_lease` role queries fabric manager DHCP lease API after BMF handoff reboot, matches port MAC to assigned IP on tenant network, controller writes to CR status, feedback syncs to fulfillment-service)
- [ ] Tenant handoff signaling (NetworkAttachmentsReady, NetworkHandoffComplete, IPDiscoveryComplete, Ready) implemented
- [ ] Integration tests pass (E2E coverage for max-one validation, auto ExternalIP, IP feedback, isolation-until-ready)
- [ ] Documentation: API reference, user guide for simplified BM creation

GA criteria:
- [ ] fabric_manager implementation (Netris BM roles, OSAC-2081) delivered and production-tested
- [ ] Dispatcher core (OSAC-1457, OSAC-1458, OSAC-1460) implemented and stable
- [ ] NATGateway full stack (OSAC-1443) implemented and stable
- [ ] Production deployment verified (MOC or other OSAC deployment)
- [ ] User feedback incorporated (usability, error messages, edge cases)
- [ ] Reboot-based short-term validated; evolution path to Ironic Standalone Networking or dedicated provisioning NIC confirmed

## Upgrade / Downgrade Strategy

### Upgrade

This controller-ownership change keeps the existing BMI attachment as the
desired-state source and adds the shared private `NetworkAttachment` proto and
internal CR as the request/status boundary. Upgrade fulfillment-service,
osac-operator, and BMF together. The new osac-operator controller adopts
existing `NetworkingJobs`, `IPDiscoveryJobs`, and
`osac.openshift.io/baremetalinstance-networking` state so an in-flight job can
be polled instead of dispatched a second time. Deployment configuration must
not run the old BMF networking reconciler and the new osac-operator networking
controller against the same BMI at once.

For a BMI already being deleted, BMF must set the new
`NetworkOffboardShutdownComplete` condition after shutdown. The networking
controller must not treat a legacy `NetworkOffboardComplete=True` condition as
proof that the port has returned to the provisioning network; it runs or
resumes the idempotent offboard move and removes the networking finalizer only
after that move succeeds. BMF waits for both the new completion condition and
finalizer removal before deprovisioning.

The maximum-one validation is a release prerequisite in the fulfillment API
and BaremetalInstance CRD. Existing resources without an attachment remain
valid; resources with an attachment continue to use the same nested field.

### Downgrade

If `N+1` upgrade fails or cluster is misbehaving:
- Manual rollback: update fulfillment-service, osac-operator, and bare-metal-fulfillment-operator images to `N` as one compatible set
- Existing BaremetalInstance resources with new `network_attachments` field will be unrecognized by `N` operator
- Manual cleanup required: delete BaremetalInstance resources created with new field, re-create without networking fields
- Auto-provisioned ExternalIP resources remain (manual cleanup required if not needed)

Acceptable downgrade steps:
- Delete CRs using new field (`network_attachments`)
- Re-create without networking fields
- Manually delete orphaned auto-provisioned resources (ExternalIP, ExternalIPAttachment labeled `osac.openshift.io/auto-created: "true"`)

## Version Skew Strategy

### Control Plane Skew

fulfillment-service, osac-operator, and bare-metal-fulfillment-operator are
deployed together and upgraded atomically by osac-installer. The networking
controller and BMF must use compatible condition semantics and the shared BMI
job-history fields. A supported deployment must not enable both versions of
the networking reconciler simultaneously: exactly one controller may dispatch
BM port-move and DHCP-query jobs. During rollout, retain the networking
finalizer until the selected controller completes offboarding.

### Client Skew

osac-cli (n-1) with fulfillment-service (n):
- Old CLI does not support `--network-attachment` flag → creates BM without networking fields (default behavior)
- New CLI uses new `--network-attachment` flag → server accepts new field

osac-cli (n) with fulfillment-service (n-1):
- New CLI uses new `--network-attachment` flag → old server rejects unknown field
- Workaround: omit `--network-attachment` flag until server is upgraded

Recommendation: keep osac-cli and fulfillment-service within one minor version.

## Support Procedures

### Symptom: BaremetalInstance stuck in Pending, condition "NetworkingConfigurationFailed"

**Detection:**
```bash
kubectl describe baremetalinstance <name> -n <namespace>
# Check status.conditions for NetworkingConfigurationFailed
```

**Cause:** The osac-operator networking controller could not resolve the fabric manager, move the port, or confirm target-segment readiness.

**Resolution:**
1. Check osac-operator networking-controller logs and the BMI `NetworkingJobs` history for the dispatch or readiness error.
2. Check AAP job logs for `move_network_attachment` role errors (switch-side) — e.g. port not found on the server, or the provisioning/tenant network segment not resolvable.
3. If fabric manager unreachable, investigate connectivity
4. If switch port config failed, investigate switch configuration

### Symptom: BM has no default gateway

**Detection:** BM cannot reach external networks, `ip route` shows no default route

**Cause:** The resolved sole attachment/interface is missing or not Ready. Omission of `primary` is valid and means the sole attachment is implicitly primary.

**Resolution:**
1. Check BaremetalInstance spec: `kubectl get baremetalinstance <name> -n <namespace> -o yaml`
2. Verify there is exactly one `networkAttachments[]` entry
3. Inspect the resolved attachment and interface status, then correct the attachment or underlying network resource and recreate the BaremetalInstance if necessary

### Symptom: Auto-provisioned ExternalIP not cleaned up after BaremetalInstance deletion

**Detection:** `kubectl get externalip` shows orphaned ExternalIP labeled `osac.openshift.io/auto-created: "true"` with no parent

**Cause:** Finalizer cleanup failed permanently

**Resolution:**
1. Check osac-operator cleanup-controller logs for cleanup errors
2. Manually delete orphaned ExternalIPAttachment: `kubectl delete externalipattachment <name> -n <namespace>`
3. Manually delete orphaned ExternalIP: `kubectl delete externalip <name> -n <namespace>`

### Symptom: ExternalIPAttachment stuck in Pending, waiting for BM IP address

**Detection:** `kubectl describe externalipattachment <name> -n <namespace>` shows condition "WaitingForIPAddress"

**Cause:** Host has not yet received DHCP-assigned IP (provisioning still in progress or failed)

**Resolution:**
1. Check BaremetalInstance status: `kubectl get baremetalinstance <name> -n <namespace> -o jsonpath='{.status.networkAttachmentStatuses[?(@.primary==true)].ipAddress}'`
2. If IP is missing, check BMF logs for provisioning and `NetworkHandoffComplete` completion.
3. If handoff completed but IP is missing, inspect osac-operator networking-controller logs, `IPDiscoveryJobs`, and the `query_dhcp_lease` result. Confirm the BareMetalHost carries the `osac.openshift.io/interface-macs` annotation with the attachment's interface.

### Symptom: BaremetalInstance deletion is waiting on the networking finalizer

**Detection:** The BMI has a deletion timestamp and still has
`osac.openshift.io/baremetalinstance-networking` in `metadata.finalizers`.

**Resolution:** Check the BMF `NetworkOffboardShutdownComplete` condition first.
Then inspect the osac-operator networking-controller logs and AAP job history
for the tenant-to-provisioning port move. Keep the finalizer until the port move
is verified; BMF must not deprovision or return the host to inventory first.

### Disabling the feature

Do not disable BMaaS networking reconciliation while any BMI has a networking
finalizer or an in-flight move/discovery job. Disabling the networking
controller before offboarding can leave the port on the tenant network and
block host deprovisioning. Resume the networking controller and complete the
owned operation before removing its finalizer.

To disable auto ExternalIP attachment:
- Remove or redact ExternalIPPool CRs (capacity exhaustion prevents auto allocation)
- No API extension to disable (fields are part of CRD, cannot be removed at runtime)

Consequences:
- Auto ExternalIP allocation fails with error (resource not created)
- Manual ExternalIP workflows remain functional
- No impact on existing running BM servers

## Infrastructure Needed

- osac-operator networking controller enabled with the shared dispatcher, fulfillment-service private API access, and read-only access to the associated BareMetalHost MAC annotation
- AAP execution environment with the fabric manager `move_network_attachment` and `query_dhcp_lease` roles
- A provisioned provisioning network segment (DHCP + gateway + SNAT, config identifier `netris_bm_provisioning_vnet`) and the initial per-server attach, plus the BareMetalHost `osac.openshift.io/interface-macs` annotation — deployment prerequisites (deployment infrastructure)
- Dispatcher core (OSAC-1457, OSAC-1458, OSAC-1460)
- Integration test environment with fabric manager and Ironic/Metal3 backend

## Dependencies

| Dependency | Jira | Status |
|-----------|------|--------|
| Dispatcher core | OSAC-1457, OSAC-1458, OSAC-1460 | Closed |
| NATGateway full stack | OSAC-1443 (10 tasks) | 1/10 In Progress |
| ExternalIPAttachment BM target in CRD | OSAC-2041 | New |
| BM DNAT flow in controller | OSAC-1496 | New |
| BareMetalNetworkAttachment proto | OSAC-1508 | New |
| Primary field on BareMetalNetworkAttachment | OSAC-2042 | New |
| Immutability + interface + primary validation | OSAC-1509 | New |
| CLI --network-attachment for BareMetalInstance | OSAC-2075 | New |
| Existing BMF networking orchestration must be changed to wait on networking-owned conditions and preserve other status fields | Not tracked | **GAP** |
| Private `NetworkAttachments.Create` proto/RPC and fulfillment-service request-to-CR reconciliation | Not tracked | **GAP** |
| BM reboot flow (reconcileReboot issues BMH annotation-based reboot after port move) | Not tracked | **GAP** |
| Integration test | OSAC-1510 | New |
| Fabric manager `move_network_attachment` role (generic port move) | OSAC-2081 (Netris BM) | Closed |
| Provisioning network segment (DHCP + gateway + SNAT, config identifier `netris_bm_provisioning_vnet`) + initial per-server attach in setup-bmaas | osac-deployment infrastructure | New |
| BareMetalHost `osac.openshift.io/interface-macs` annotation (inventory tooling) | osac-deployment infrastructure | New |
| BareMetalInstance CRD: add NetworkAttachments | Not tracked | **GAP** |
| mutateBMI: copy network_attachments to K8s CR | Not tracked | **GAP** |
| osac-operator BaremetalInstance networking controller: dispatcher move, readiness wait, DHCP query, status ownership and networking finalizer | Not tracked | **GAP** |
| osac-operator RBAC: BaremetalInstance status/finalizer updates, Subnet/NetworkClass resolution, BareMetalHost annotation read | Not tracked | **GAP** |
| Conflict-safe BMF and networking-controller status writes with per-field and per-condition ownership | Not tracked | **GAP** |
| Remove unused BareMetalInstance spec.networkClass field | Not tracked | **GAP** |
| BareMetalInstanceType: network ports (BareMetalNetworkPortSpec) with name, role, type, speed | Not tracked | **GAP** |

---

## Provenance

Authored: revise @ design 0.11.3 - 858df2d, workspace HEAD @ 06d340f90 (22 behind origin/main)
Final: revise @ design 0.11.3 - 858df2d, workspace HEAD @ 06d340f90 (39 behind origin/main)

> Context changed between revise and revise.

> This document's phase history does not include an initial /draft — structure was not verified against the template from origin.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.3","ai_workflows":"858df2d","source_repo":"06d340f90","source_repo_branch":"HEAD","commits_behind_main":39,"commits_ahead_main":0,"main_ref":"main","phases":["revise","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":true} -->
