---
title: Unified Networking Requirements for VMaaS, CaaS, and BMaaS
authors:
  - dmanor@redhat.com
creation-date: 2026-06-03
last-updated: 2026-09-24
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-1433
see-also:
  - Unified Networking Design: /enhancements/OSAC-1433-unified-networking
  - BareMetal Instance API: /enhancements/OSAC-1118-baremetal-instance-api
replaces:
  - N/A
superseded-by:
  - N/A
---

# Unified Networking Requirements for VMaaS, CaaS, and BMaaS

| Field       | Value                                      |
|-------------|--------------------------------------------|
| Author(s)   | Dan Manor (dmanor@redhat.com)              |
| Jira        | https://redhat.atlassian.net/browse/OSAC-1433 |
| Date        | 2026-09-24                                 |

## 1. Problem Statement

VMaaS, CaaS, and BMaaS need one networking model so tenants can connect workloads across service types without learning deployment-specific networking implementations. Today, service teams use separate workflows, and workload-level traffic policies do not provide a consistent subnet-wide boundary. This leaves tenants without a shared way to define and inspect network placement and traffic access.

## 2. Goals and Non-Goals

### 2.1 Goals

- Tenants use the same VirtualNetwork, Subnet, NetworkACL, and external-access resources across VMaaS, CaaS, and BMaaS.
- Tenants control ingress and egress traffic for all workloads on a Subnet through stateless NetworkACL rules evaluated by match specificity.
- Workloads in the three services can share a VirtualNetwork and Subnet while retaining at most one tenant network attachment per workload.
- Providers select and operate networking implementations without exposing those implementation choices to tenants.
- Tenants use provider-routable IPv4 addressing in connected deployments.

### 2.2 Success Metrics

| Metric | Target | Baseline |
|--------|--------|----------|
| Service types using the networking API | 3/3 (VMaaS, CaaS, BMaaS) | 1/3 (VMaaS only) |
| Service types bypassing the networking API for network configuration | 0/3 | 2/3 (CaaS, BMaaS) |
| API changes required to add a networking implementation | 0 | Requires API and operator changes |

### 2.3 Non-Goals

- VPC peering or communication between VirtualNetworks
- Tenant-managed DNS zones
- Load balancer or Internet Gateway APIs
- IPv6 and dual-stack networking
- Multi-hub networking placement or cross-hub connectivity
- Multiple tenant network attachments per workload
- Quota enforcement for networking resources
- Per-physical-interface configuration beyond selecting one BMaaS interface

## 3. Requirements

### 3.1 Functional Requirements

- **FR-1:** Tenants can create isolated VirtualNetworks and Subnets. Resources in different VirtualNetworks cannot communicate, and resources in the same Subnet share a broadcast domain. [Jira: OSAC-1433]
- **FR-2:** Tenants can create a NetworkACL within a VirtualNetwork and define separate ingress and egress rules. Each rule specifies whether matching traffic is allowed or denied, protocol, an optional TCP or UDP destination-port range, and a canonical IPv4 CIDR. In each direction, the effective order is derived from the match fields: longest CIDR prefixes first, then protocol (`ICMP`, `UDP`, `TCP`, `ALL`), then destination-port ranges from smallest to largest, with rules matching all ports after port-specific rules. Action and request order do not determine precedence. The first matching rule decides; identical match fields in one direction are rejected. If no ACL rule matches, the deployment's default ACL policy decides the result. [User]
- **FR-3:** NetworkACLs are stateless. Ingress and egress are evaluated independently, and return traffic requires a matching rule in the reverse direction. [User]
- **FR-4:** A Subnet may have zero or one associated NetworkACL. If the ACL is
  omitted at Subnet creation, the association remains unset; an explicit ACL
  must be READY and belong to the same VirtualNetwork, and requests with more
  than one ACL are rejected. A NetworkACL is scoped to one VirtualNetwork and
  may be referenced by multiple Subnets in that VirtualNetwork. For each
  direction, the associated ACL's rules are evaluated by match specificity;
  when no rule matches—or no ACL is associated—the deployment's default ACL
  policy decides the result. That policy is the final catch-all and is not
  replaced by an associated ACL. The effective policy applies uniformly to
  every workload attached to the Subnet. Traffic between workloads on the same
  Subnet is not filtered by the Subnet's NetworkACL. For traffic between
  Subnets, source egress and destination ingress rules are evaluated
  independently. [User]
- **FR-5:** NetworkACL rules and a Subnet's NetworkACL association are fixed at creation. Changing them requires deleting and recreating the affected networking resources. A VirtualNetwork's and Subnet's address configuration and a workload's network attachment also remain fixed after creation. [User]
- **FR-6:** ComputeInstance, Cluster, and BaremetalInstance attachments identify a Subnet and do not carry traffic-policy references. A BaremetalInstance attachment may also identify one physical interface. Each workload supports at most one tenant network attachment. [User; OSAC-1433]
- **FR-7:** The networking API provides read (list/get), create, and delete operations for networking resources. NetworkACL rules and Subnet-to-NetworkACL associations cannot be updated after creation. Workload attachments remain immutable after workload creation. [User; OSAC-1433]
- **FR-8:** All three service types use the same network resource model and can place workloads on any compatible Subnet. A Subnet and its associated NetworkACL belong to the same VirtualNetwork. [Jira: OSAC-1433]
- **FR-9:** Tenants can allocate ExternalIPs and attach them to ComputeInstances, Clusters, and BaremetalInstances for inbound traffic. ExternalIP means external to the VirtualNetwork and does not promise Internet reachability. [Jira: OSAC-1433]
- **FR-10:** A NATGateway can provide a stable egress identity for a VirtualNetwork. It is optional and handles outbound external access; ExternalIPAttachment handles inbound access. [Jira: OSAC-1433]
- **FR-11:** Providers configure networking implementations. Tenants do not select implementation backends, and adding a supported backend does not require a tenant API change. [Jira: OSAC-1433]
- **FR-12:** The supported deployment profile is connected networking with one provider-owned hub. ExternalIPPool accepts exactly one canonical IPv4 CIDR. IPv6, dual-stack, disconnected deployment, and multi-hub networking requests are rejected or reported unsupported. [Jira: OSAC-1433]
- **FR-13:** At tenant onboarding, the system creates a default VirtualNetwork,
  a default Subnet, and a NATGateway from provider-configured defaults. It does
  not create a tenant default NetworkACL. The default Subnet has no ACL
  association unless an ACL is explicitly selected; the deployment's default
  ACL policy applies to unmatched traffic, including all traffic when no ACL
  is associated. Workload creation can omit network attachment details to use
  the default Subnet. Tenant readiness waits for the default VirtualNetwork,
  Subnet, and NATGateway to become READY, with no per-tenant ACL resource or
  association prerequisite. [User; OSAC-1433]
- **FR-14:** Existing workload traffic policies require tenant-assisted migration where their scope or stateful behavior cannot be represented by a Subnet-level stateless ACL. Tenants can group workloads by intended policy, place each group on a Subnet with the corresponding shared NetworkACL, and add reverse-direction rules where return traffic is required. [User]
- **FR-15:** The provider must configure a deployment-wide default ACL policy of `PERMIT` or `DENY`. ACL rules are evaluated by match specificity before this policy; the policy is the final catch-all for each direction and applies whether or not a Subnet has an associated NetworkACL. [User]

### 3.2 Non-Functional Requirements

- **NFR-1:** Network isolation, ACL enforcement, address assignment, and external access behave consistently for VMaaS, CaaS, and BMaaS workloads. [Jira: OSAC-1433]
- **NFR-2:** Tenants receive validation errors before persistence or
  provisioning when a request violates the IPv4-only, same-VirtualNetwork
  association, at-most-one-ACL-per-Subnet, single-attachment, or single-hub
  constraints. [OSAC-1433; User]

## 4. Acceptance Criteria

- [ ] Resources in different VirtualNetworks cannot communicate; Subnets in the same VirtualNetwork retain Layer 3 connectivity subject to their ingress and egress NetworkACL rules.
- [ ] A tenant can create a NetworkACL with ingress and egress rules that include ALLOW or DENY, protocol, optional TCP/UDP destination ports, and an IPv4 CIDR.
- [ ] Each direction derives rule precedence from CIDR prefix length, protocol, and destination-port range; action and request order do not affect precedence, identical match fields are rejected, the first matching rule determines the result, and unmatched traffic uses the deployment's default ACL policy.
- [ ] Reply traffic is evaluated independently and passes only when the reverse direction has a matching rule.
- [ ] A Subnet has zero or one NetworkACL association. An omitted reference
  remains unset; an explicit ACL must be READY and in the same VirtualNetwork,
  and multiple ACLs are rejected. One NetworkACL can be shared by multiple
  Subnets in the same VirtualNetwork. Unmatched traffic uses the deployment's
  default ACL policy.
- [ ] NetworkACL rules and Subnet-to-NetworkACL associations are set at creation and cannot be updated; changes require deleting and recreating affected networking resources.
- [ ] All resources on one Subnet receive the same ACL policy; same-Subnet traffic is not filtered by that ACL.
- [ ] Cross-Subnet traffic must pass the source Subnet's egress rules and the destination Subnet's ingress rules.
- [ ] ComputeInstance, Cluster, and BaremetalInstance network attachments refer to a Subnet and contain no traffic-policy references; BMaaS can additionally select one interface.
- [ ] Workloads of all three service types can use the same networking resources, and each workload has at most one tenant network attachment.
- [ ] ExternalIPAttachment supports all three service types for inbound traffic, and NATGateway remains optional for outbound traffic.
- [ ] Default tenant readiness waits for the default VirtualNetwork, Subnet,
  and NATGateway, with no tenant default ACL resource.
- [ ] The deployment has a configured `PERMIT` or `DENY` default ACL policy.
  More-specific matching ACL rules take precedence over the policy, which is
  the final catch-all; reply traffic passes only when the reverse-direction
  evaluation allows it.
- [ ] Default-based workload creation stores and returns the resolved Subnet attachment.
- [ ] Unsupported IPv6, dual-stack, disconnected, and multi-hub configurations are rejected before provisioning.
- [ ] Existing policies that cannot be represented exactly are migrated through tenant-directed workload grouping and explicit reverse-direction ACL rules.
- [ ] Networking implementations remain hidden from tenant-facing APIs.

## 5. Dependencies

- **Unified Networking Design:** [/enhancements/OSAC-1433-unified-networking](/enhancements/OSAC-1433-unified-networking) defines the resource and API contract.
- **Default Networking:** [/enhancements/OSAC-1433-default-networking](/enhancements/OSAC-1433-default-networking) defines tenant defaults and simplified workload creation.
- **VMaaS, CaaS, and BMaaS networking enhancements:** define the service-specific attachment and provisioning behavior.
- Networking API, fulfillment, operator, fabric, and K8s networking components must implement the shared contract.

---

## Provenance

Authored: revise @ prd 0.11.3 - cc0daa6, workspace main @ 06d340f90 (43 behind origin/main)
Final: revise @ prd 0.11.3 - 2bd6607, workspace main @ 2293f9140 (3 behind origin/main)

> Context changed between revise and revise.

> This document's phase history does not include an initial /draft — structure was not verified against the template from origin.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"prd","workflow_version":"0.11.3","ai_workflows":"2bd6607","source_repo":"2293f9140","source_repo_branch":"main","commits_behind_main":3,"commits_ahead_main":0,"main_ref":"main","phases":["revise","respond","revise","revise","manual-edit","revise","manual-edit","revise","manual-edit","revise"],"authoring_modes":["manual","skill"],"context_changed":true,"origin_untracked":true} -->
