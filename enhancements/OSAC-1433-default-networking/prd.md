# Simplified Resource Creation — Default Networking and Auto ExternalIP

| Field       | Value   |
|-------------|---------|
| Author(s)   | Dan Manor |
| Jira        | https://redhat.atlassian.net/browse/OSAC-1433 |
| Date        | 2026-07-02 |

## 1. Problem Statement

Creating a reachable resource in OSAC requires 6+ sequential API calls:
VirtualNetwork, Subnet, SecurityGroup, the resource itself, ExternalIP,
and ExternalIPAttachment. Every tenant must understand the full networking
resource model before provisioning their first VM, cluster, or bare-metal
server. This friction slows onboarding, increases the chance of
misconfiguration, and makes OSAC harder to adopt compared to platforms
where a single create command produces a reachable instance.

## 2. Goals and Non-Goals

### 2.1 Goals

- A tenant can create a fully connected VM, bare-metal server, or cluster
  (inbound + outbound) with a single create action, without pre-creating any
  networking resources
- Tenants who need custom networking retain the full explicit workflow —
  simplified creation is additive, not a replacement
- Auto-provisioned networking resources are visible, editable, and follow
  the same lifecycle as manually created ones

### 2.2 Non-Goals

- Custom default configurations per tenant (all tenants in a deployment share
  the provider's configured defaults)
- Auto-provisioning of VirtualNetworks or Subnets beyond the initial
  default (tenants create additional VNs manually)
- UI support for simplified creation (deferred — API and CLI only for now)
- Automatically enrolling existing tenants (only new tenants receive defaults
  during onboarding)

## 3. User Stories

### Tenant User Stories

- As a Tenant User, I want to create a resource (VM, cluster, or
  bare-metal server) without pre-creating networking resources, so that
  the system provides sensible defaults and I can get started quickly
- As a Tenant User, I want to create a resource with external access enabled
  and have it externally reachable in a single create action, without
  manually creating supporting networking resources
- As a Tenant User, I want auto-provisioned ExternalIPs to be
  automatically cleaned up when I delete the parent resource, so that I do
  not accumulate orphaned resources
- As a Tenant User, I want to create a Cluster with external access enabled
  and have the system automatically provide access for both the API server and
  ingress endpoints before cluster provisioning begins

### Tenant Admin Stories

- As a Tenant Admin, I want to inspect and customize my default networking
  resources (e.g., modify SecurityGroup rules) after they are auto-created

### Cloud Infrastructure Admin Stories

- As a Cloud Infrastructure Admin, I want to configure the defaults used for
  tenant networking, so that the system can provide a ready-to-use network at
  onboarding

### Cloud Provider Admin Stories

- As a Cloud Provider Admin, I want visibility into whether a tenant's
  default networking resources were successfully provisioned, so I can
  troubleshoot onboarding failures

## Design Boundary

The companion [Default Networking Design](design.md) is the normative home for
default-resource schemas, onboarding behavior, resource selection, attachment
resolution, cleanup ordering, validation, failure handling, and test strategy.

Default networking makes these product capabilities available at tenant
onboarding and when a tenant creates a workload without explicit networking.

## Product Acceptance Criteria

- [ ] A new tenant receives a ready-to-use default network for supported workloads.
- [ ] A tenant can create a workload without pre-creating networking resources.
- [ ] A tenant can request external access without manually creating supporting networking resources.
- [ ] Explicit custom networking remains available and is not replaced by defaults.
- [ ] Auto-provisioned networking resources are visible and cleaned up with their parent workload.

## Dependencies

- **Unified Networking Design** — the shared networking resource model and
  provider architecture
- **VMaaS, CaaS, and BMaaS networking designs** — service-specific use of
  default attachments and external access
- **Tenant onboarding** — the product workflow that makes default networking
  available to a tenant

## Design Notes

Failure modes, recovery behavior, and service-specific test coverage are
defined in the companion design.
