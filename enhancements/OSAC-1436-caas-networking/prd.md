# CaaS Networking — Unified API with Auto External Access

| Field       | Value   |
|-------------|---------|
| Author(s)   | Dan Manor (dmanor@redhat.com) |
| Jira        | https://redhat.atlassian.net/browse/OSAC-1436 |
| Date        | 2026-07-08 |

> This PRD is an expansion of the [Unified Networking PRD](/enhancements/OSAC-1433-unified-networking/prd.md), scoped to the specific service type. The unified PRD defines shared product outcomes; the companion designs define shared and service-specific technical behavior; this document defines the service-specific product requirements and user stories.

## 1. Problem Statement

Cluster provisioning has no networking configuration. Tenants cannot choose which subnet their cluster nodes use, cannot place two clusters in the same virtual network, and cannot isolate them in separate networks. All clusters are placed on a single deployment-wide network configuration with zero tenant control. Cluster networking is completely divergent from VM and bare-metal server workflows, requiring separate knowledge and tools.

## 2. Goals and Non-Goals

### 2.1 Goals

- A tenant can create a cluster with explicit network configuration, specifying which subnet and security groups to use for cluster nodes
- A cluster's node sets use a consistent network connection, with the system selecting suitable host connectivity automatically
- Tenants can request automatic external access for cluster API server and ingress endpoints, without pre-creating external IP resources
- When network configuration is omitted, the system applies the tenant's default subnet and security group
- Cluster status exposes API server and ingress endpoint addresses after provisioning completes
- The system automatically selects suitable bare-metal hosts and configures network connectivity before cluster provisioning begins
- Auto-provisioned external IPs and external IP attachments are cleaned up when the cluster is deleted
- Host catalog information is sufficient for the system to connect cluster nodes

### 2.2 Non-Goals

- VM-based cluster node sets (deferred — bare-metal only for initial release)
- DNS API for cluster endpoints (DNS record creation remains template-based until DNS API is implemented)
- Per-node-set subnet placement (all node sets share the cluster's single network attachment)
- Multi-NIC cluster nodes (one attachment per cluster; the system automatically determines which physical interface to use for each node set based on its host type)

## 3. User Stories

### Tenant User Stories

- As a Tenant User, I want to create a cluster with explicit network configuration so that I can place it on a specific subnet with specific security group rules
- As a Tenant User, I want my cluster's node sets to automatically use suitable host connectivity so that networking is configured without manual connection selection
- As a Tenant User, I want to create a cluster with external access enabled so that the system provisions access for both the API server and ingress without manual networking setup
- As a Tenant User, I want to create a cluster without specifying network configuration and have it placed on my default subnet with my default security groups
- As a Tenant User, I want to see my cluster's API server and ingress endpoint addresses in the cluster status so that I can access the cluster
- As a Tenant User, I want auto-provisioned networking resources to be automatically cleaned up when I delete my cluster so that I do not accumulate orphaned resources

### Tenant Admin Stories

- As a Tenant Admin, I want to place multiple clusters in the same virtual network so that they can communicate privately with each other and with my VMs
- As a Tenant Admin, I want to isolate clusters in separate virtual networks so that I can enforce network boundaries between different projects or teams

### Cloud Infrastructure Admin Stories

- As a Cloud Infrastructure Admin, I want to define structured network interface metadata for bare-metal host types so that the system can automatically configure network connectivity when provisioning clusters

### Cloud Provider Admin Stories

- As a Cloud Provider Admin, I want visibility into whether cluster hosts were successfully selected and network connectivity configured so I can troubleshoot provisioning failures

## Design Boundary

The companion [CaaS Networking Design](design.md) is the normative home for attachment behavior, host
selection, interface resolution, endpoint discovery, external access
activation, cleanup, failure handling, and test strategy.

The product requirements above are implemented according to the companion
design; this PRD does not duplicate the technical contract.

## Product Acceptance Criteria

- [ ] A tenant can place a cluster on a selected subnet with the desired security rules.
- [ ] A tenant can create a cluster without explicit networking and use the tenant's default network.
- [ ] Cluster status exposes the API server and ingress addresses after provisioning.
- [ ] A tenant can request external access for both cluster endpoints without manually creating supporting resources.
- [ ] Networking resources created for a cluster are cleaned up when the cluster is deleted.

## Assumptions

- The tenant has default networking resources (virtual network, subnet, security group) pre-created. If defaults are not configured, creating a cluster without explicit network configuration fails with a clear error.
- The deployment has the networking capabilities required by this product.
- The host catalog contains enough information for the system to connect cluster nodes.

## Dependencies

- **Unified Networking Design** — shared networking resource model and provider behavior defined in the [Unified Networking Design](/enhancements/OSAC-1433-unified-networking/design.md)
- **Default Networking Design** — default subnet and security group behavior defined in the [Default Networking Design](/enhancements/OSAC-1433-default-networking/design.md)

## Design Notes

Technical risks, resolution behavior, and test scenarios are defined in the
companion design.
