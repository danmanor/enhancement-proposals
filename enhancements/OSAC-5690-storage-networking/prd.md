---
title: storage-networking
authors:
  - dmanor@redhat.com
creation-date: 2026-09-27
last-updated: 2026-10-04
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-5690
see-also:
  - enhancements/OSAC-1433-unified-networking/prd.md
  - enhancements/OSAC-1111-storage-backend/prd.md
  - enhancements/OSAC-1110-storage-tier/prd.md
  - enhancements/OSAC-1332-caas-cluster-storage/prd.md
  - enhancements/OSAC-1435-vmaas-networking/prd.md
  - enhancements/OSAC-1436-caas-networking/prd.md
---

# Storage Networking (Phase 1)

| Field       | Value                                              |
|-------------|----------------------------------------------------|
| Author(s)   | Dan Manor                                          |
| Jira        | [OSAC-5690](https://redhat.atlassian.net/browse/OSAC-5690) |
| Date        | 2026-10-04                                         |

## Terminology

| Term | Definition |
|------|-----------|
| **VAST VIP** | A Virtual IP address exposed by the VAST storage cluster that workloads connect to for block storage data-plane operations. VIPs are managed by VAST and are located outside the managed network fabric gateway. |
| **NetApp ONTAP data endpoint** | An IP data endpoint used by Trident for ONTAP block storage over iSCSI or NVMe/TCP. Fibre Channel is outside this IP-networking scope. |
| **Storage CIDR** | One or more IP ranges reserved at OSAC installation time for VAST VIPs and NetApp ONTAP block data endpoints. These ranges must not overlap with tenant VirtualNetwork CIDRs and must route outside the fabric gateway. |
| **Per-Tenant VIP Pool** | Each tenant receives a dedicated VAST VIP pool. Storage tenant isolation at the network level is not required for the first phase, but per-tenant VIP pools exist on the VAST side. |

## Problem Statement

VMaaS, CaaS, and BMaaS workloads need a network path to remote VAST and
NetApp ONTAP block storage. The OSAC networking and storage subsystems are
currently independent: the storage design (OSAC-1332, OSAC-1111) assumes "CaaS cluster
nodes have network reachability to the storage backend" without specifying how
that reachability is achieved.

The first phase needs a clear, minimal connectivity solution that allows
supported consumers to reach either backend without waiting for a full
tenant-isolated storage network design. The following problems must be solved:

1. **No defined network path from workloads to storage.** Tenant workloads run
   inside isolated VirtualNetworks on the OSAC fabric. VAST and NetApp ONTAP
   block data endpoints are outside the managed network fabric gateway. There
   is no mechanism today ensuring that traffic destined for those endpoints
   routes externally rather than being trapped within the fabric.

2. **IP overlap risk.** Tenants choose their own VirtualNetwork CIDRs (or
   receive defaults from NetworkClass). If a VN CIDR overlaps with a VAST VIP
   or NetApp ONTAP data endpoint range, packets destined for storage will be
   routed within the fabric instead of externally, breaking storage access
   silently.

3. **NAT capacity for storage traffic.** Block storage workloads generate
   concurrent NVMe/TCP or iSCSI sessions and CSI operations. The NATGateway's ExternalIP
   must support these connections. There is no guidance or validation today for
   NAT capacity relative to storage consumption.

If not addressed, storage will be unreachable from tenant workloads — blocking
the first phase.

## In Scope

- Network connectivity from VMaaS and CaaS environments to VAST and NetApp
  ONTAP block storage through the corresponding CSI integrations (VAST CSI
  and Trident).
- Network connectivity from BMaaS hosts to VAST and NetApp ONTAP block storage
  (network path only; tenant-side storage configuration remains manual).
- One or more Storage CIDRs configured at OSAC installation time, reserved
  for VAST VIPs and NetApp ONTAP block data endpoints.
- Validation preventing tenants from creating VirtualNetworks whose CIDRs
  overlap with the Storage CIDR, ensuring storage-bound packets always
  route externally.
- Ensuring VirtualNetworks that host storage-consuming workloads have a
  NATGateway with adequate NAT capacity for storage traffic.
- VAST and NetApp ONTAP block storage only. For NetApp ONTAP, this scope covers
  IP-based iSCSI and NVMe/TCP data paths; Fibre Channel is out of scope.

## Out of Scope

- **File or object storage.** Only block storage is supported in the first phase.
- **Storage backends other than VAST and NetApp ONTAP**, including Pure Storage
  and Ceph.
- **Storage tenant isolation at the network level.** Per-tenant VAST VIP pools
  exist but are not network-isolated from each other, and this feature does not
  add network isolation for NetApp ONTAP endpoints. Tenant-isolated storage
  networking is deferred to OSAC-5073.
- **Automating tenant storage configuration or CSI setup for BMaaS.** BMaaS
  tenants configure storage manually.
- **Direct-attach / VLAN-based storage networking.** This design assumes both
  backends are external and accessed over SNAT. Dedicated storage VLANs, SR-IOV, or
  RDMA paths are not covered.
- **Per-subnet or per-workload NAT.** NATGateway operates at the
  VirtualNetwork level. Per-subnet NAT granularity is future work.
- **Storage traffic QoS or bandwidth reservation.**
- **East-west storage paths.** GPU-to-storage over east-west fabric
  (Spectrum-X, InfiniBand) is deferred per OSAC-1382.

## User Stories

### VMaaS and CaaS Users

- As a VMaaS or CaaS user, I want my workloads to reach VAST or NetApp ONTAP
  block storage through the corresponding CSI integration so that I can
  provision and mount
  PersistentVolumes without needing to understand the underlying network
  topology.

### BMaaS Tenants

- As a BMaaS tenant, I want the network path to VAST and NetApp ONTAP storage
  to be available so that I can configure storage manually if I choose to use it.

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want to configure Storage CIDRs at OSAC
  installation time so that the platform reserves VAST VIP and NetApp ONTAP
  data endpoint ranges and prevents conflicts with tenant networks.

- As a Cloud Infrastructure Admin, I want the platform to reject
  VirtualNetwork creation requests whose CIDR overlaps with any Storage CIDR
  so that storage-bound traffic always routes externally and never gets
  trapped in the fabric.

### Cloud Provider Admin

- As a Cloud Provider Admin, I want a minimal connectivity path that can be
  delivered within the first phase timeframe so that storage is unblocked
  without requiring the full tenant-isolated storage network design.

- As a Cloud Provider Admin, I want the default VirtualNetwork CIDR configured
  in NetworkClass to be validated against the Storage CIDRs so that newly
  onboarded tenants do not receive a default network that conflicts with
  storage.

## Assumptions

- VAST and NetApp ONTAP block data endpoints are reachable from the fabric's
  external network. Tenants consume both backends over IP; there is no in-fabric
  storage deployment for the first phase.

- Each tenant receives a dedicated VAST VIP pool. Storage tenant isolation at
  the network level is not required for the first phase — tenant workloads use
  the same SNAT path to their configured VAST or NetApp ONTAP block endpoints.

- SNAT via NATGateway is sufficient for IP-based block data-plane traffic
  (NVMe/TCP and iSCSI sessions). No inbound (DNAT) connectivity from either
  backend to tenant workloads is required — storage connections are initiated
  by the client side.

- A single NATGateway ExternalIP per VirtualNetwork provides enough NAT
  capacity for the expected storage connection count in the first phase scope.

- Storage CIDRs are configured once at installation and do not change during
  the deployment's lifetime. Multiple ranges may be needed for distinct VAST
  and NetApp ONTAP data endpoint networks.

- All tenant VirtualNetworks that host workloads requiring storage must have a
  NATGateway configured. The default VirtualNetwork created during tenant
  onboarding already includes a NATGateway.

- BMaaS hosts have network connectivity to VAST and NetApp ONTAP through their
  management or fabric network interface. BMaaS tenants are responsible for
  configuring storage on their hosts.

## Dependencies

- **Unified Networking (OSAC-1433):** VirtualNetwork, NATGateway, ExternalIP,
  and NetworkClass must be implemented and operational. Storage networking
  builds on these primitives — it does not introduce new networking resources.

- **Storage Backend & Tier (OSAC-1111, OSAC-1110):** StorageBackend
  registration and StorageTier assignment must be functional so that VAST
  VIP pool or NetApp ONTAP endpoint configuration is available.

- **CaaS Cluster Storage (OSAC-1332):** The storage controller that installs
  CSI drivers and StorageClasses on tenant clusters must be operational. This
  PRD addresses the network reachability prerequisite that OSAC-1332 assumes.

- **Tenant Onboarding (Default Networking — OSAC-1433):** The default
  networking onboarding flow must validate the Storage CIDR constraint and
  ensure NATGateway provisioning.

- **OSAC-5073 (Shared VAST Global VIP Pool):** OSAC-5073 covers the broader
  VAST-specific effort. This feature adds the first-phase connectivity path
  for both VAST and NetApp ONTAP block storage without changing OSAC-5073's
  scope.

---

## Provenance

Authored: revise @ prd 0.11.3 - 2bd6607, workspace main @ 1f3b63b82 (52 behind origin/main)

> This document's phase history does not include an initial /draft — structure was not verified against the template from origin.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"prd","workflow_version":"0.11.3","ai_workflows":"2bd6607","source_repo":"1f3b63b82","source_repo_branch":"main","commits_behind_main":52,"commits_ahead_main":0,"main_ref":"main","phases":["revise"],"authoring_modes":["skill"],"context_changed":false,"origin_untracked":true} -->
