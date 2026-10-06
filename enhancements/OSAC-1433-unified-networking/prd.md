---
title: Unified Networking Requirements for VMaaS, CaaS, and BMaaS
authors:
  - dmanor@redhat.com
creation-date: 2026-06-03
last-updated: 2026-10-06
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-1433
see-also:
  - Unified Networking Design: /enhancements/OSAC-1433-unified-networking
  - Network Manager Integration Contract PRD: /enhancements/OSAC-1433-network-manager-integration-contract-networking/prd.md
  - Network Manager Integration Contract Design: /enhancements/OSAC-1433-network-manager-integration-contract-networking/design.md
  - BareMetal Instance API: /enhancements/OSAC-1118-baremetal-instance-api
  - Three-Layer Networking Model: https://docs.google.com/document/d/1MwBjpmYoZoUN3PVjeIRZ2Y6mBuf0lu1uvTtN6XXPPTM
replaces:
  - N/A
superseded-by:
  - N/A
---

# Unified Networking Requirements for VMaaS, CaaS, and BMaaS

| Field       | Value   |
|-------------|---------|
| Author(s)   | Dan Manor (dmanor@redhat.com) |
| Jira        | https://redhat.atlassian.net/browse/OSAC-1433 |
| Target release | OSAC 0.2 |
| Date        | 2026-10-06 |

## Contents

- [Problem Statement](#1-problem-statement)
- [Goals and Non-Goals](#2-goals-and-non-goals)
  - [Goals](#21-goals)
  - [Success Metrics](#22-success-metrics)
  - [Non-Goals](#23-non-goals)
- [User Stories](#user-stories)
  - [Tenant Admin](#tenant-admin)
  - [Tenant User](#tenant-user)
  - [Cloud Infrastructure Admin](#cloud-infrastructure-admin)
  - [Cloud Provider Admin](#cloud-provider-admin)
- [Requirements](#3-requirements)
  - [Functional Requirements](#31-functional-requirements)
  - [Non-Functional Requirements](#32-non-functional-requirements)
- [Acceptance Criteria](#4-acceptance-criteria)
  - [Shared tenant networking](#shared-tenant-networking)
  - [External access and provider choice](#external-access-and-provider-choice)
  - [Resource lifecycle](#resource-lifecycle)
  - [Provider networking control](#provider-networking-control-fr-10)
  - [Unified networking UI and documentation](#unified-networking-ui-and-documentation-fr-11)
- [Dependencies](#5-dependencies)
- [Support Boundaries](#support-boundaries)
  - [Deployment support boundary](#deployment-support-boundary)
  - [Networking hub support boundary](#networking-hub-support-boundary)

## 1. Problem Statement

OSAC tenants use virtual machines, managed clusters, and bare-metal servers, but networking is not consistent across those workloads. VMaaS has a tenant networking model, while CaaS and BMaaS rely on separate service-specific flows. Tenant Admins and Tenant Users cannot apply one familiar network model across their workloads, and Cloud Infrastructure Admins must support different provider networking paths. A shared set of networking capabilities gives tenants consistent control and gives providers one configurable networking contract to support.

## 2. Goals and Non-Goals

### 2.1 Goals

- Tenant Admins and Tenant Users can use the same networking capabilities with VMs, clusters, and bare-metal servers.
- Tenants can isolate workloads, connect workloads on their networks, and control inbound and outbound external access.
- Cloud Infrastructure Admins choose the provider networking implementation without requiring tenants to understand or select it.
- Cloud Provider Admins can control whether OSAC performs provider networking operations during installation or upgrade while ordinary workload provisioning remains available.
- Tenant Admins and Tenant Users can manage the shared networking model through the unified UI and the supported API and CLI surfaces.

### 2.2 Success Metrics

| Metric | Target | Baseline |
|--------|--------|----------|
| Workload service types using the shared networking model | 3/3 (VMaaS, CaaS, BMaaS) | 1/3 |
| Workload service types using a separate networking model | 0/3 | 2/3 (CaaS, BMaaS) |

### 2.3 Non-Goals

- IPv6 or dual-stack networking.
- More than one tenant network attachment per workload.
- VPC peering or other cross-VirtualNetwork connectivity.
- Tenant-managed DNS zones, load balancers, or Internet gateways.
- Advanced bare-metal interface configuration such as NIC bonding or VLAN trunking.
- Quota enforcement for networking resources.

## User Stories

### Tenant Admin

- As a Tenant Admin, I want to create isolated VirtualNetworks and Subnets for VMs, clusters, and bare-metal servers, so that the network layout is consistent across workload types.
- As a Tenant Admin, I want to define SecurityGroups and manage networking resources through the unified UI, so that I can govern traffic and resource lifecycle for my tenant.
- As a Tenant Admin, I want clear errors when a resource is not ready or still has dependents, so that I can correct a request without leaving resources stuck or breaking a workload.

### Tenant User

- As a Tenant User, I want to attach a workload to a ready Subnet and SecurityGroups, so that I can use the tenant's shared networking without choosing a provider implementation.
- As a Tenant User, I want to request an ExternalIP for inbound access or a NATGateway for outbound access, so that I can use the access direction I need.
- As a Tenant User, I want the unified UI to show resource relationships and whether external access is available, so that I can manage networking across VMaaS, CaaS, and BMaaS in one place.

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want to choose a provider networking service that meets OSAC's requirements and manage shared external address capacity, so that the provider can change implementations without changing the tenant resource model.
- As a Cloud Infrastructure Admin, I want to view NetworkClasses and ExternalIP pools in the unified UI, so that I can inspect configured manager capabilities and address capacity.

### Cloud Provider Admin

- As a Cloud Provider Admin, I want to enable or disable provider networking during installation or upgrade, so that I can control networking operations while keeping ordinary workload provisioning available.

The unified UI is in scope as a product outcome and is tracked by [OSAC-2226](https://redhat.atlassian.net/browse/OSAC-2226). This PRD defines resource and workflow outcomes; screen-level UX is covered by that UI work. User documentation must explain resource creation and deletion, workload attachment, ExternalIP reachability, and disabled-provider behavior.

The target release is OSAC 0.2. The provider networking setting is included because it defines the operating mode for the shared resources across VMaaS, CaaS, and BMaaS, including behavior when provider networking is disabled.

## 3. Requirements

### 3.1 Functional Requirements

#### FR-1: Network isolation and connectivity (R1)

Tenants must be able to create isolated VirtualNetworks. Workloads on the same Subnet share a local network segment and can communicate directly when their attached SecurityGroups permit the traffic. Workloads on different Subnets in the same VirtualNetwork can communicate through routing when their SecurityGroups permit the traffic. Workloads in separate VirtualNetworks remain isolated. These outcomes apply across VMaaS, CaaS, and BMaaS.

#### FR-2: Infrastructure-agnostic networking resources (R2)

Every networking resource uses the same model across VMs, cluster nodes, and bare-metal servers. The shared model includes tenant resources and provider-managed NetworkClasses and ExternalIP pools. Workload-specific connection details, such as a selected bare-metal interface, do not require a separate networking resource model.

#### FR-3: Uniform networking across all service types (R3)

Tenant Admins and Tenant Users can use the shared VirtualNetwork, Subnet, SecurityGroup, ExternalIP, ExternalIPAttachment, and NATGateway capabilities with VMaaS, CaaS, and BMaaS. Provider-managed address pools make ExternalIPs available to all three workload types.

#### FR-4: ExternalIP is external to the VirtualNetwork (R4)

An ExternalIP provides an address outside a tenant's VirtualNetwork. The provider determines where that address is reachable; OSAC does not promise that it is reachable from the public Internet.

#### FR-5: Clear ingress/egress separation (R5)

Tenants can configure inbound access to a workload with an ExternalIPAttachment and optional outbound access with a NATGateway. These capabilities have distinct purposes, and a NATGateway is not required for a workload to have basic connectivity.

#### FR-6: Pluggable networking backends with transparent selection (R6)

Cloud Infrastructure Admins can use any provider networking implementation that meets OSAC's requirements for the capabilities it supplies. The same resource model works with any conforming implementation; tenants do not select or need to understand the provider choice.

#### FR-7: Single network attachment per workload (R7)

Each ComputeInstance, Cluster, and BaremetalInstance can use at most one tenant network attachment. For bare-metal workloads, the Tenant User can choose one available physical interface when the selected template exposes interface choices. Multiple tenant attachments per workload are outside this proposal's scope.

#### FR-8: Create/read/delete networking contract (R8)

Tenant Admins and Tenant Users can create, list, view, and delete tenant-managed networking resources within their tenant. Cloud Infrastructure Admins can create, list, view, and delete provider-managed NetworkClasses and ExternalIP pools according to their role. Resource access follows ownership and authorization. No networking resource can be changed in place; a change requires replacing it. A workload's network attachment is also fixed when the workload is created and can be changed only by replacing the workload.

#### FR-9: Strict resource lifecycle enforcement (R9)

OSAC prevents users from creating a networking resource or workload before its referenced resources complete their required lifecycle, and prevents deletion of a resource that active resources depend on. OSAC returns a clear explanation of the unmet prerequisite or blocking resource so users can correct the request.

#### FR-10: Provider networking control at installation or upgrade

A Cloud Provider Admin can enable or disable OSAC provider networking through Helm or the Enclave Wizard during installation or upgrade. The setting defaults to enabled and takes effect after the rollout. Networking APIs remain available through supported API, CLI, and UI surfaces, and retain their normal authorization, validation, defaults, and dependency rules.

When disabled, OSAC continues to manage networking objects but does not apply provider network configuration, allocate provider addresses, or clean up provider networking. Logical object status does not promise network connectivity or provider-side changes; existing provider rules or resources may remain until the provider cleans them up. Ordinary workload provisioning remains available when its other prerequisites are met: VMs use platform default networking, new bare-metal hosts remain on provisioning connectivity, and CaaS still requires baseline connectivity for cluster installation and control-plane services. Tenant network connectivity, ExternalIP allocation, and outbound NAT are not provided by OSAC in this mode. [User]

#### FR-11: Unified networking UI and documentation

Tenant Admins and Tenant Users can manage VirtualNetworks, Subnets, SecurityGroups, ExternalIPs, ExternalIPAttachments, and NATGateways through the unified UI, and can use shared resource pickers from VMaaS, CaaS, and BMaaS workflows. Cloud Infrastructure Admins can inspect provider NetworkClasses and ExternalIP pools. User documentation explains the supported resource workflows, workload attachments, ExternalIP reachability, and provider networking control.

### 3.2 Non-Functional Requirements

No non-functional requirements were specified for this proposal.

## 4. Acceptance Criteria

### Shared tenant networking

- [ ] Tenant Admins and Tenant Users can use the same networking resource model with VMaaS, CaaS, and BMaaS.
- [ ] Workloads on one Subnet share a local network segment; traffic between them is allowed only when their attached SecurityGroups permit it. Workloads on different Subnets in the same VirtualNetwork can communicate through routing when their SecurityGroups permit the traffic.
- [ ] Workloads in separate VirtualNetworks are isolated.
- [ ] VMs, cluster nodes, and bare-metal servers can be placed on the same Subnet without selecting a workload-specific network.
- [ ] SecurityGroups control permitted traffic consistently for each workload type when provider networking is enabled.
- [ ] A workload can have at most one tenant network attachment; a bare-metal workload can select one exposed physical interface.

### External access and provider choice

- [ ] An ExternalIP can provide inbound access to a VM, a bare-metal server, or a cluster endpoint.
- [ ] A NATGateway provides optional outbound access with a stable source identity, independently of inbound ExternalIP attachments.
- [ ] Documentation and interfaces do not imply that an ExternalIP is necessarily Internet-routable.
- [ ] Tenants do not need to select a provider networking implementation to create or use tenant networking resources.
- [ ] Cloud Infrastructure Admins can configure a supported provider implementation without changing the tenant networking model.

### Resource lifecycle

- [ ] Tenant Admins and Tenant Users can create, list, view, and delete tenant-managed networking resources within their tenant. Cloud Infrastructure Admins can create, list, view, and delete provider-managed NetworkClasses and ExternalIP pools according to their role. Resource access follows ownership and authorization, and no role can change resource configuration in place.
- [ ] Changing a workload's network attachment requires replacing the workload.
- [ ] A request that depends on a missing resource or one that has not completed its required lifecycle is rejected with an explanation of the prerequisite.
- [ ] A deletion that would leave active resources without a dependency is rejected with an explanation of what blocks deletion.

### Provider networking control (FR-10)

- [ ] A Cloud Provider Admin can enable or disable provider networking during installation or upgrade through Helm or the Enclave Wizard; the default is enabled.
- [ ] After rollout with provider networking disabled, the networking APIs remain available with the same authorization and validation behavior, while OSAC does not apply or clean up provider networking or allocate provider addresses.
- [ ] Disabled-mode readiness does not lead users to believe that provider connectivity, ExternalIP allocation, or outbound NAT is available.
- [ ] Ordinary VM, cluster, and bare-metal provisioning remains available when its non-networking prerequisites are met, with the limitations described in FR-10.
- [ ] Provider-side resources or rules left by disabling networking are identified as requiring provider-side or manual cleanup.

### Unified networking UI and documentation (FR-11)

- [ ] The unified UI supports create, list/view, and delete for tenant networking resources and shows their VirtualNetwork, Subnet, and SecurityGroup relationships.
- [ ] Shared networking pickers are available to VMaaS, CaaS, and BMaaS workload workflows and show only resources the caller is authorized to use.
- [ ] Cloud Infrastructure Admins can inspect provider NetworkClasses and ExternalIP pools, including configured capabilities and available address capacity.
- [ ] The UI does not offer in-place configuration updates or imply that an ExternalIP is Internet-routable.
- [ ] User documentation covers resource creation/deletion, workload attachment, ExternalIP reachability, lifecycle errors, and disabled-provider behavior.

## 5. Dependencies

- **Unified Networking Design:** [/enhancements/OSAC-1433-unified-networking](/enhancements/OSAC-1433-unified-networking) defines the technical approach for these requirements.
- **Default Networking:** [/enhancements/OSAC-1433-default-networking](/enhancements/OSAC-1433-default-networking) defines tenant default-resource automation.
- **BareMetal Instance API:** [/enhancements/OSAC-1118-baremetal-instance-api](/enhancements/OSAC-1118-baremetal-instance-api) defines the BaremetalInstance resource used by BMaaS.
- **Per-service networking proposals:** [VMaaS](/enhancements/OSAC-1435-vmaas-networking), [CaaS](/enhancements/OSAC-1436-caas-networking), and [BMaaS](/enhancements/OSAC-1437-bmaas-networking) define how each service consumes the shared networking model.
- **Network Manager Integration Contract:** Its [PRD](/enhancements/OSAC-1433-network-manager-integration-contract-networking/prd.md) defines how providers determine whether manager implementations and selected pairs meet OSAC's requirements while preserving the shared tenant networking model.
- **Unified Networking UI (OSAC-2226):** Tracks the standalone networking UI and shared resource pickers required by FR-11.
- **User documentation:** API, CLI, and UI guidance must reflect the shared resource lifecycle and the support limits in this PRD.
- **Three-Layer Networking Model:** [Architecture reference](https://docs.google.com/document/d/1MwBjpmYoZoUN3PVjeIRZ2Y6mBuf0lu1uvTtN6XXPPTM).

## Support Boundaries

### Deployment support boundary

Networking supports connected deployments only. The provider-owned hub,
selected networking services, and provider-controlled address infrastructure
must be reachable within that deployment.

### Networking hub support boundary

Each deployment has one provider-owned networking hub. Multiple hosting
clusters remain supported where a workload feature calls for them, but
networking resources are not coordinated across multiple hubs.

---

## Provenance

Committed: commit @ prd 0.11.3 - 2bd6607, workspace docs/unified-networking-docs-structure @ 3d0f65e

> Authoring phases not recorded this session (commit-time snapshot only).

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"commit_only","workflow":"prd","workflow_version":"0.11.3","ai_workflows":"2bd6607","source_repo":"3d0f65e","source_repo_branch":"docs/unified-networking-docs-structure","commits_behind_main":0,"commits_ahead_main":19,"main_ref":"main","phases":["commit","commit","commit","commit"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
