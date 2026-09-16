# BMaaS Networking — Network Attachments and Auto External Access

| Field       | Value   |
|-------------|---------|
| Author(s)   | Dan Manor (dmanor@redhat.com) |
| Jira        | https://redhat.atlassian.net/browse/OSAC-1437 |
| Date        | 2026-07-08 |

> This PRD is an expansion of the [Unified Networking PRD](/enhancements/OSAC-1433-unified-networking/prd.md), scoped to the specific service type. The unified PRD defines shared product outcomes; the companion designs define shared and service-specific technical behavior; this document defines the service-specific product requirements and user stories.

## 1. Problem Statement

Provisioning bare-metal servers requires manual switch configuration outside the OSAC API. Tenants cannot attach bare-metal servers to subnets, apply security groups, or configure external access through the API. The system does not expose which physical network interfaces are available on a bare-metal server, forcing tenants to discover interface names through out-of-band documentation. Creating a reachable bare-metal server with both inbound and outbound connectivity requires sequential API calls to create networking resources and manual coordination with infrastructure administrators for switch port configuration.

## 2. Goals and Non-Goals

### 2.1 Goals

- A tenant can provision a bare-metal server with explicit network connections to selected subnets
- A tenant can create a bare-metal server with external access enabled and have the system allocate the required external address automatically
- Network attachments are optional — when omitted, the system attaches the server to the tenant's default subnet and security group
- Tenants can discover suitable connection options for bare-metal servers
- Network connectivity for each attachment is established before bare-metal OS provisioning begins
- External IP attachments support bare-metal servers as a target type

### 2.2 Success Metrics

| Metric | Target | Baseline |
|--------|--------|----------|
| BM provisioning time with networking | <5 min | N/A (no baseline) |
| Network connectivity configuration success rate | >95% | N/A |

### 2.3 Non-Goals

- Cluster or VM networking (this PRD covers bare-metal servers only; clusters and VMs are addressed in separate enhancements)
- Multi-interface failover or bonding (out of scope for initial implementation)

## 3. User Stories

### Tenant User Stories

- As a Tenant User, I want to create a bare-metal server with explicit network attachments so that I can connect specific physical interfaces to specific subnets
- As a Tenant User, I want to see which connection options are available for a host type so that I can select the appropriate network when creating attachments
- As a Tenant User, I want to create a bare-metal server with external access enabled and have it externally reachable in a single API call, without manually creating supporting networking resources
- As a Tenant User, I want to create a multi-homed bare-metal server (multiple network attachments) and designate which interface provides the default gateway
- As a Tenant User, I want auto-provisioned external IPs to be automatically cleaned up when I delete the server, so that I do not accumulate orphaned resources
- As a Tenant User, I want network interface validation when creating attachments so that I get clear errors if I specify an interface that doesn't exist or attach the same interface to multiple subnets

### Tenant Admin Stories

- As a Tenant Admin, I want visibility into which physical interfaces are connected to which subnets for a bare-metal server so that I can troubleshoot network connectivity issues

### Cloud Infrastructure Admin Stories

- As a Cloud Infrastructure Admin, I want to describe the connection options for each host type so that tenants can discover and use the appropriate networks

### Cloud Provider Admin Stories

- As a Cloud Provider Admin, I want to see which IP addresses were allocated to each network interface on a bare-metal server so that I can troubleshoot connectivity and external access configuration

## 4. Design Boundary

The companion [BMaaS Networking Design](design.md) is the normative home for attachment behavior, physical-interface mapping and primary-interface rules, address discovery, provisioning gates, external access activation, cleanup ordering, failure handling, and test strategy.

The product requirements above are implemented according to that design; this PRD does not duplicate the technical contract.

## Product Acceptance Criteria

- [ ] A tenant can create a bare-metal server with explicit network connections.
- [ ] A tenant can discover suitable host connection options before selecting network connections.
- [ ] A tenant can create an externally reachable bare-metal server without manually creating supporting networking resources.
- [ ] A multi-homed server can use a designated primary connection for default access.
- [ ] Auto-provisioned networking resources are cleaned up when the server is deleted.

## Assumptions

- The tenant has default networking resources (virtual network, subnet, security group) pre-created at onboarding (see Default Networking PRD). If defaults are not configured, creating a server without explicit network attachments fails with a clear error.
- Bare-metal host types expose the connection options needed by tenants to select network attachments.
- Out-of-band provisioning connectivity remains reserved for system use.

## Dependencies

- **Unified Networking Design** — shared networking resource model and provider architecture
- **Default Networking Design** — default subnet and security-group behavior

## Design Notes

Failure modes, recovery behavior, and service-specific test coverage are defined in the companion design.
