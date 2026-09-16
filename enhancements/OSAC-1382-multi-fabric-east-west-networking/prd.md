# Multi-Fabric East-West Networking

| Field       | Value   |
|-------------|---------|
| Author(s)   | Vladik Romanovsky |
| Jira        | [OSAC-1382](https://redhat.atlassian.net/browse/OSAC-1382) |
| Date        | 2026-07-14 |

## Problem Statement

High-performance workloads — AI training and inference, storage clusters, HPC — running in shared, multi-tenant sovereign clouds require high-bandwidth, low-latency east-west connectivity between servers. Distributed AI workloads (training and inference) drive massive east-west traffic through collective operations (RDMA over InfiniBand/RoCE), and storage replication and other scale-out workloads have similar requirements. At the same time, hard tenant isolation must be enforced across all fabric types to prevent data leakage.

Today, provisioning east-west connectivity across heterogeneous fabrics (Ethernet/Spectrum-X, InfiniBand, NVLink) requires manual, error-prone coordination. This results in slow tenant onboarding, high operational overhead, risk of misaligned isolation boundaries across fabrics, and inability to offer predictable high-performance east-west out of the box.

OSAC's unified networking model (EP #50) provides north-south connectivity and general workload support. Per-service networking extensions (CaaS in EP #107, with VMaaS and BMaaS planned) build on this foundation. However, none of these address automated east-west provisioning or unified multi-fabric tenant isolation.

## In Scope (Phase 1)

- Declarative east-west connectivity on Ethernet-based fabrics.
- Automated multi-tenant isolation on east-west paths, enforced at the fabric level.
- Ability to create east-west isolation domains for a group of servers — as part of tenant onboarding, as an explicit admin operation, or when resizing an existing deployment.
- Visibility into tenant isolation boundaries for auditing and troubleshooting.
- Integration with the unified networking primitives (EP #50) and compatible with per-service extensions (EP #107).

## Out of Scope

- Full InfiniBand east-west support (PKey management, SHARP, UFM integration) — planned for Phase 2+.
- NVLink Multi-Node partition management and alignment with other fabrics — planned for Phase 2+.
- High-performance east-west storage access (GPU-to-storage over east-west paths).
- Cross-tenant east-west connectivity (explicitly forbidden).
- North-south connectivity enhancements (covered by base unified networking in EP #50).
- DPU/HBN Virtual Function assignment and software-based host segmentation.
- Layer-4 load balancing for tenant services.

## Future Phases / Roadmap (for awareness)

Phase 2+ will expand support to InfiniBand (PKey + UFM) and NVLink Multi-Node partitions, along with tighter alignment across all three fabrics and high-performance east-west storage access patterns.

## Product Flow

An east-west isolation domain can be created during onboarding, as an explicit administrator action, or while resizing a deployment. The companion [Multi-Fabric East-West Networking Design](design.md) defines how membership, isolation, connectivity, and lifecycle are implemented.

## User Stories

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want to integrate OSAC with a supported fabric provider so that tenant network isolation is automatically enforced without manual network configuration.

- As a Cloud Infrastructure Admin, I want to create east-west isolation domains for a group of servers so that tenants get high-performance, isolated connectivity for their workloads.

- As a Cloud Infrastructure Admin, I want to add or remove servers from an existing east-west isolation domain so that I can resize tenant deployments without recreating the isolation domain.

### Cloud Provider Admin

- As a Cloud Provider Admin, I want visibility into tenant connectivity allocation and isolation boundaries so that I can audit the service and troubleshoot connectivity issues.

### Tenant Admin

- As a Tenant Admin, I want confidence that my tenant's east-west network isolation is enforced at the fabric level so that other tenants cannot access my data or traffic.

- As a Tenant Admin, I want to define SecurityGroup rules that control which resources can communicate east-west within my tenant's networks, and have those rules protect that communication.

### Tenant User

- As a Tenant User, I want my provisioned compute instances to connect to the correct east-west network so that distributed workloads can communicate without additional network configuration.

- As a Tenant User, I want my Kubernetes clusters to have east-west connectivity within my isolation boundary. (Note: namespace-level network isolation within clusters is provided by the k8s networking layer — see EP #107.)

## Design Boundary

The companion design is the normative home for isolation-domain definitions, membership and tenant-boundary validation, provider integration, connectivity behavior, lifecycle handling, failure recovery, and test strategy.

The product outcomes above are implemented according to that design; this PRD does not duplicate the technical contract.

## Product Acceptance Criteria

- [ ] An east-west isolation domain can be created during onboarding, by an administrator, or while resizing a deployment.
- [ ] Workloads in the same tenant isolation domain can communicate over the supported east-west network.
- [ ] Cross-tenant east-west communication is blocked.
- [ ] Administrators can inspect domain membership and isolation boundaries for troubleshooting.
- [ ] Supported compute workloads receive east-west connectivity without additional tenant network configuration.

## Assumptions

- Target deployments provide high-performance east-west connectivity on the supported fabric types.
- The unified networking model from EP #50 is the base layer.
- Per-service networking extensions (EP #107 for CaaS, others planned) will be merged before or in parallel with this work.

## Dependencies

- **Unified Networking (EP #50):** Shared networking primitives must be in place as the foundation layer.
- **Fabric provider:** A supported provider integration must expose the capabilities required for the requested east-west service.

## Design Notes

The companion design records implementation risks, provider limitations, recovery behavior, and control-plane and data-plane test coverage.
