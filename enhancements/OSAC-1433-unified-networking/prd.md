---
title: Unified Networking Requirements for VMaaS, CaaS, and BMaaS
authors:
  - dmanor@redhat.com
creation-date: 2026-06-03
last-updated: 2026-09-16
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-1433
see-also:
  - Unified Networking Design: /enhancements/OSAC-1433-unified-networking
  - Original Networking API: /enhancements/OSAC-356-networking
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
| Date        | 2026-06-03 |

The companion [Unified Networking Design](design.md) defines the networking
architecture and implementation terminology. This PRD focuses on the product
outcomes for tenants and providers across VMaaS, CaaS, and BMaaS.

## 1. Problem Statement

The OSAC Networking API must serve as a foundational service across all three
OSAC service types — VMaaS, CaaS, and BMaaS — with a single, consistent
resource model. The technical design that fulfills these requirements is
described in the companion [Unified Networking Design](design.md).

The original networking work focused on virtual machines, leaving cluster and
bare-metal users with different experiences. Tenants therefore cannot manage
networking consistently across their workloads, and providers must support
different operational paths for each service type.

The result is fragmented networking with no consistent tenant-facing
abstraction. Users need one way to create isolated networks, connect all
supported workload types, and control inbound and outbound access. Providers
need to operate that experience consistently across deployments, including
air-gapped environments.

The remaining architecture and implementation gaps are addressed in the
companion design and the service-specific networking designs.

## 2. Goals and Non-Goals

### 2.1 Goals

- Provide a unified networking API across VMaaS, CaaS, and BMaaS with a single, consistent resource model
- Enable tenants to manage networking resources (VirtualNetworks, Subnets, SecurityGroups, ExternalIPs) without choosing how networking is implemented
- Allow providers to offer the same tenant networking experience across supported deployments
- Enable VMs, clusters, and bare-metal servers to coexist in the same VirtualNetwork
- Work in air-gapped environments using data-center-routable IPs
- Support network attachment for bare-metal servers with multiple physical interfaces

### 2.2 Non-Goals

- VPC Peering / cross-VN communication (separate enhancement)
- DNS API for tenant-managed DNS zones (separate enhancement)
- Advanced physical network topology beyond basic network attachment
- Load Balancer API
- Internet Gateway API
- Quota enforcement for networking resources

## 3. User Stories

### Tenant Stories (All Services)

- As a tenant, I want to create isolated VirtualNetworks and Subnets for my
  workloads without choosing a networking backend
- As a tenant, I want to define SecurityGroups to control traffic to and
  from my resources
- As a tenant, I want to allocate ExternalIPs and attach them to my VMs,
  clusters, or bare-metal servers for inbound access
- As a tenant, I want to create a NATGateway for outbound access from my
  VirtualNetwork

### CaaS-Specific Stories

- As a tenant, I want to place my cluster's worker nodes on a Subnet in my
  VirtualNetwork
- As a tenant, I want to attach ExternalIPs to my cluster's API server and
  ingress endpoints before provisioning
- As a tenant, I want my cluster to work in air-gapped environments using
  data-center-routable IPs

### BMaaS-Specific Stories

- As a tenant, I want to place my BaremetalInstance on Subnets in my
  VirtualNetwork
- As a tenant, I want to see the available physical interfaces on a bare-metal
  template so I can decide how to attach networks
- As a tenant, I want to attach different physical interfaces of my
  BaremetalInstance to different Subnets (e.g., data interface to a data
  subnet, management interface to a management subnet)
- As a tenant, I want to attach an ExternalIP to my bare-metal server for
  inbound access

### Provider Stories

- As a provider, I want to configure the networking experience without exposing
implementation choices to tenants, so that tenants use the same workflow
across deployments.

## Design Boundary

The companion [Unified Networking Design](design.md) is the normative home for
the shared [API specification](design.md#api-specification), resource
definitions, provider behavior, attachment and address assignment rules,
lifecycle behavior, validation, and test strategy.

## Product Acceptance Criteria

- [ ] Tenants can use one networking resource model across VMaaS, CaaS, and BMaaS.
- [ ] VMs, clusters, and bare-metal servers can use the same isolated tenant network.
- [ ] Tenants can control inbound and outbound access for supported workloads.
- [ ] Providers can offer the networking experience without exposing implementation choices to tenants.
- [ ] Bare-metal tenants can use multiple physical network interfaces when supported by the host.

## Dependencies

- **Unified Networking Design**: [design.md](design.md) — technical design fulfilling these requirements
- **Default Networking**: [/enhancements/OSAC-1433-default-networking](/enhancements/OSAC-1433-default-networking) — related onboarding experience
- **VMaaS, CaaS, and BMaaS networking designs** — service-specific technical flows built on this product requirement
- **BareMetal Instance API**: [/enhancements/OSAC-1118-baremetal-instance-api](/enhancements/OSAC-1118-baremetal-instance-api) — bare-metal workload dependency
