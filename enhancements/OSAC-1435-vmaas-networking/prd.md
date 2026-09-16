# VMaaS Networking — Multi-Interface VMs and Auto External Access

| Field       | Value   |
|-------------|---------|
| Author(s)   | Dan Manor (dmanor@redhat.com) |
| Jira        | https://redhat.atlassian.net/browse/OSAC-1435 |
| Date        | 2026-07-08 |

> This PRD is an expansion of the [Unified Networking PRD](/enhancements/OSAC-1433-unified-networking/prd.md), scoped to the specific service type. The unified PRD defines shared product outcomes; the companion designs define shared and service-specific technical behavior; this document defines the service-specific product requirements and user stories.

## 1. Problem Statement

Tenants cannot create VMs with multiple network interfaces or designate which interface provides the default gateway. Creating a VM with external access requires manual IP allocation and NAT configuration, forcing tenants to understand inbound and outbound routing before provisioning their first reachable VM. The default networking experience varies across resource types — some resources have simplified creation flows while VMs require explicit networking details on every create.

## 2. Goals and Non-Goals

### 2.1 Goals

- A tenant can create a VM with multiple network interfaces on different subnets, designating one as primary
- A tenant can create a VM with external access enabled and have the system allocate and attach an external IP for inbound access
- A tenant can create a VM without specifying networking details — the system uses the tenant's default subnet and security group
- The platform prevents VM creation in deployments that do not support virtualization

### 2.2 Non-Goals

- Cluster or bare-metal server networking (this PRD covers VMs only; clusters and bare-metal servers are addressed in separate enhancements)
- Multiple network interfaces for bare-metal servers (bare-metal multi-interface support is out of scope)

## 3. User Stories

### Tenant User Stories

- As a Tenant User, I want to create a VM with multiple network interfaces, so that the VM can communicate on multiple subnets
- As a Tenant User, I want to designate one network interface as primary, so that it provides the VM's default gateway and DNS configuration
- As a Tenant User, I want to create a VM with external access enabled, so that the VM is externally reachable without manually allocating an IP
- As a Tenant User, I want to create a VM without specifying network details, so that the system uses my default subnet and security group and I can get started quickly
- As a Tenant User, I want clear error messages when I try to create a VM in a deployment that only supports bare-metal servers, so that I understand the limitation and can choose a different deployment

### Tenant Admin Stories

- As a Tenant Admin, I want to inspect and modify the default networking resources (subnet, security group) used when VMs are created without explicit network configuration
- As a Tenant Admin, I want to see which subnet and security groups each VM is attached to, and the IP address allocated to each interface, so I can audit my organization's network topology

### Cloud Infrastructure Admin Stories

- As a Cloud Infrastructure Admin, I want to configure which deployments support VM provisioning, so that VM creation is rejected with a clear error in BM-only deployments

### Cloud Provider Admin Stories

- As a Cloud Provider Admin, I want visibility into auto-provisioned networking resources (external IPs), so I can monitor capacity and troubleshoot connectivity issues

## Design Boundary

The companion [VMaaS Networking Design](design.md) is the normative home for attachment behavior, primary
interface behavior, address discovery, compatibility handling, provisioning
validation, cleanup, failure handling, and test strategy.

The product requirements above are implemented according to the companion
design; this PRD does not duplicate the technical contract.

## Product Acceptance Criteria

- [ ] A tenant can create a VM with multiple network connections and designate the primary connection.
- [ ] A tenant can create an externally reachable VM without manually allocating an external address.
- [ ] A VM created without explicit networking uses the tenant's default network.
- [ ] VM status exposes the network connections and addresses needed to use and troubleshoot the VM.
- [ ] Existing VM clients and configurations continue to work when the networking enhancement is enabled.
- [ ] Unsupported VM provisioning returns a clear, user-understandable error.

## Assumptions

- The tenant has default networking resources (virtual network, subnet, security group) pre-created by the platform (see Default Networking PRD). If defaults are not configured, creating a VM without explicit network configuration fails with a clear error.
- The target deployment supports virtualization. Bare-metal-only deployments do not support VMs.

## Dependencies

- **Unified Networking Design** — shared networking resource model and provider behavior defined in the [Unified Networking Design](/enhancements/OSAC-1433-unified-networking/design.md)
- **Default Networking Design** — default subnet and security group behavior defined in the [Default Networking Design](/enhancements/OSAC-1433-default-networking/design.md)
- **Virtualization platform integration** — required for the platform to provision
  VMs and connect them to tenant networking

## Design Notes

Technical risks, resolution behavior, and test scenarios are defined in the
companion design.
