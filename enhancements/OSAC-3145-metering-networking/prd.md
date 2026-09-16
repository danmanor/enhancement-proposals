# Metering and Usage Tracking — Part 2c: Networking

| Field       | Value                |
|-------------|----------------------|
| Author(s)   | masayag@redhat.com   |
| Jira        | [OSAC-3145](https://redhat.atlassian.net/browse/OSAC-3145) |
| Date        | 2026-07-26           |

## Glossary

Terms defined in the [Part 1 PRD](/enhancements/metering-and-usage-tracking/prd.md) apply here. Additional terms:

| Term | Definition |
|------|-----------|
| **Allocation metering** | Metering that runs from the point a resource is allocated until deletion, regardless of whether the resource is actively in use. Reflects the provider's physical capacity reservation. |
| **Deployment** | A discrete OSAC installation identified by the metering configuration. |

## 1. Problem Statement

OSAC provisions ExternalIPs and NAT Gateways that consume scarce provider infrastructure from allocation until deletion, but has no mechanism to report that consumption to billing. An ExternalIP consumes finite address pool space whether it is attached to a resource or not — the provider's pool is finite and each allocation reduces availability. A NAT Gateway consumes dedicated gateway capacity for as long as it exists. Metering exists to supply billing with usage data for resources that can incur cost — not to enforce quota, and not to inventory every networking object a tenant holds.

VirtualNetworks, Subnets, and SecurityGroups are configuration metadata that will not incur cost — they are free across all surveyed hyperscalers and GPU/AI clouds, none of which meter them on an allocation basis — so metering does not report them.

Without metering for the billable networking resources, Cloud Provider Admins have no usage data to account for the scarce network infrastructure tenants hold, and Tenant Admins have no visibility into their billable networking footprint across projects.

## 2. In Scope

### 2.1 Services

Metered networking resources are service-agnostic — an ExternalIP or NATGateway is metered regardless of which service (VMaaS, CaaS, BMaaS) consumes it.

| Resource | VMaaS | CaaS | BMaaS |
|----------|-------|------|-------|
| NATGateway | Yes | Yes | Yes |
| ExternalIP | Yes | Yes | Yes |

ExternalIP resources support all three services and can be attached to ComputeInstances, Clusters, and BareMetalInstances. Usage reporting distinguishes whether an ExternalIP is attached, without metering the attachment as a separate resource.

VirtualNetwork, Subnet, and SecurityGroup are available on all three services but are not metered — they are configuration metadata that will not incur cost, so metering does not report them.

### 2.2 Capabilities

- Billing-bound reporting — metering reports only networking resources that can incur cost; it is not a quota feed and not a complete inventory of the networking objects a tenant or user holds
- Networking resource allocation metering — metering for ExternalIPs and NATGateways from allocation to deletion
- Unattached IP metering — ExternalIPs generate usage data regardless of whether they are attached
- Parent-child attribution — extending the existing metering attribution model so that ExternalIPs attached to a parent resource can be attributed to it in a unified usage view

## 3. Out of Scope

- Metering of VirtualNetwork, Subnet, and SecurityGroup — these are configuration metadata that will not incur cost (free across all surveyed hyperscalers and GPU/AI clouds); metering does not report them (per PR #159 review, [comment 5204380439](https://github.com/osac-project/enhancement-proposals/pull/159#issuecomment-5204380439))
- Quota enforcement and a complete inventory of the networking resources a tenant or user holds — these are not purposes of metering
- BMaaS compute metering — tracked separately ([OSAC-2506](https://redhat.atlassian.net/browse/OSAC-2506)); ExternalIPs and NATGateways consumed by BMaaS are in scope here
- Storage metering — tracked separately ([OSAC-3141](https://redhat.atlassian.net/browse/OSAC-3141))
- Network bandwidth metering (ingress/egress traffic) — tracked separately ([OSAC-3149](https://redhat.atlassian.net/browse/OSAC-3149))
- Costing, billing, rate schedules, invoicing, and budget alerts — deferred to a separate billing PRD; this PRD supplies the usage data that billing consumes
- UI for viewing networking usage — metering data is consumed by the billing system, which provides the user-facing usage views
- Workload-level metering inside tenant environments

## 4. User Stories

### Cloud Provider Admin

- As a Cloud Provider Admin, I want networking resource usage data across all tenants to be available broken down by resource type (ExternalIP, NATGateway), so that downstream systems can track the scarce network infrastructure each tenant consumes.

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want to add meters for new networking resource types that consume scarce infrastructure (e.g., LoadBalancer, VPN Gateway) via configuration without redeployment, extending Part 1 CAP-6 to networking resources.

### Tenant Admin

- As a Tenant Admin, I want my organization's networking resource usage data to be available broken down by project, including the count and duration of ExternalIPs and NATGateways, so that downstream systems can attribute networking consumption to the teams that provisioned them.

### Tenant User

- As a Tenant User, I want networking resource usage data for the projects I belong to — including ExternalIP allocation duration and NATGateway uptime — to be available so that downstream systems can report the networking resource consumption of my deployments.

## 5. Product Capabilities

- Usage reporting covers billable ExternalIP and NATGateway resources for all supported services.
- Usage accrues for the lifetime of an allocated resource, including an ExternalIP that is not attached to a workload.
- Reports identify resource type, IP family where applicable, attachment status, deployment ownership, tenant, and project.
- ExternalIP usage can be attributed to its parent workload when attached.
- Networking usage extends the existing metering service without requiring a separate metering deployment.
- Usage has per-second granularity and follows the existing metering service's deduplication and retention expectations.

## 6. Design Boundary

The companion [Networking Metering Design](design.md) is the normative home for lifecycle boundaries, measurement and attribution contracts, correction behavior, feature gates, failure recovery, and test strategy.

The product capabilities above are implemented according to that design; this PRD does not duplicate the technical contract.

## 7. Acceptance Criteria

- [ ] Usage data is available for ExternalIP and NATGateway resources across VMaaS, CaaS, and BMaaS.
- [ ] Usage includes ExternalIPs that are allocated but not attached to a workload.
- [ ] Configuration-only networking resources do not produce billable usage data.
- [ ] Usage can be reported by resource type, IP family where applicable, attachment status, deployment, tenant, project, and attached parent where applicable.
- [ ] Networking usage has per-second granularity and follows the existing metering service's deduplication, retention, and reliability expectations.

## 8. Assumptions

- Part 1 metering infrastructure is deployed and operational.
- The Part 1 metering infrastructure supports duration-based usage for networking resources.

## 9. Dependencies

- **Part 1 metering infrastructure:** The metering infrastructure established by [Part 1](/enhancements/metering-and-usage-tracking/prd.md) is a prerequisite. Part 2c extends but does not replace it.

## 10. Design Notes

The companion design records implementation risks, rollout prerequisites, lifecycle boundaries, correction behavior, and detailed validation.

## Related PRDs

This PRD is part of the Metering Part 2 family:

- **Part 2a: BMaaS** — [OSAC-2506](https://redhat.atlassian.net/browse/OSAC-2506)
- **Part 2b: Storage** — [OSAC-3141](https://redhat.atlassian.net/browse/OSAC-3141)
- **Part 2c: Networking** — this document (OSAC-3145)
- **Part 2d: Network Bandwidth** — [OSAC-3149](https://redhat.atlassian.net/browse/OSAC-3149)

---

## Provenance

Committed: commit @ prd 0.9.0 - 562b610, workspace main @ 1095dc5d3

> Authoring phases not recorded this session (commit-time snapshot only).

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"commit_only","workflow":"prd","workflow_version":"0.9.0","ai_workflows":"562b610","source_repo":"1095dc5d3","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["commit"],"authoring_modes":["skill"],"context_changed":false,"origin_untracked":false} -->
