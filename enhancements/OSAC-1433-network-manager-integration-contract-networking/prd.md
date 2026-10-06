---
title: network-manager-integration-contract
authors:
  - dmanor@redhat.com
creation-date: 2026-10-04
last-updated: 2026-10-06
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-1433
see-also:
  - Unified Networking PRD: /enhancements/OSAC-1433-unified-networking/prd.md
  - Unified Networking Design: /enhancements/OSAC-1433-unified-networking/design.md
  - Network Manager Integration Contract Design: /enhancements/OSAC-1433-network-manager-integration-contract-networking/design.md
replaces:
  - N/A
superseded-by:
  - N/A
---

# Network Manager Integration Contract

| Field | Value |
|-------|-------|
| Author(s) | Dan Manor (dmanor@redhat.com) |
| Jira | https://redhat.atlassian.net/browse/OSAC-1433 |
| Date | 2026-10-06 |

## Contents

- [1. Problem Statement](#1-problem-statement)
- [2. Goals and Non-Goals](#2-goals-and-non-goals)
- [3. Requirements](#3-requirements)
  - [3.1 Functional Requirements](#31-functional-requirements)
  - [3.2 Non-Functional Requirements](#32-non-functional-requirements)
- [4. Acceptance Criteria](#4-acceptance-criteria)
- [5. Dependencies](#5-dependencies)

## 1. Problem Statement

OSAC networking uses provider-selected Fabric and Kubernetes (K8s) managers, but providers do not have one clear way to tell which combinations work together or whether a new implementation meets OSAC's requirements. An incompatible combination can prevent workloads from joining the intended network. Providers and implementation authors must infer the integration expectations, making source-neutral onboarding and consistent networking across virtual machines (VMs), managed Kubernetes clusters, and bare-metal workloads difficult to assess. [User]

## 2. Goals and Non-Goals

### 2.1 Goals

- Cloud Infrastructure Admins can determine which Fabric and K8s manager implementations can work together before selecting them.
- Providers can assess whether a manager implementation meets OSAC's complete role requirements using one published contract.
- Tenants can use the same networking resources and workflows for VMs, managed clusters, and bare-metal workloads across compatible manager combinations.
- Providers can select an implementation regardless of who supplies it, without changing tenant networking workflows.

### 2.2 Non-Goals

- Changing the shared tenant networking resources or their meaning.
- Requiring every Fabric Manager to work with every K8s Manager.
- Exposing provider manager selection to tenants.
- Defining vendor-specific configuration for Netris, Agentless VLAN, CUDN, or another backend.

## 3. Requirements

### 3.1 Functional Requirements

- **FR-1:** A provider can select a conforming Fabric Manager or K8s Manager implementation regardless of who supplies it; OSAC-provided implementations meet the same requirements. [User]
- **FR-2:** A Cloud Infrastructure Admin must be able to determine from one published contract what a manager must provide and whether a selected Fabric and K8s Manager pair can interoperate. [User]
- **FR-3:** Tenants can use the same OSAC networking resources and workflows for VMs, managed Kubernetes clusters, and bare-metal workloads across compatible manager combinations, without selecting a provider backend. [Unified Networking PRD: FR-2, FR-6] [User]
- **FR-4:** When a selected manager pair is incompatible, a required role is unavailable, or a request is outside the supported manager contract, the Cloud Infrastructure Admin must receive a clear diagnostic and OSAC must reject the request rather than route it through an unrelated manager. [User]

### 3.2 Non-Functional Requirements

No separate non-functional requirements were specified for this work.

## 4. Acceptance Criteria

- [ ] A provider can determine the complete requirements for each manager role and identify which selected manager combinations are supported from one published contract.
- [ ] Implementations from any source, including those distributed with OSAC, are eligible when they meet the same published contract.
- [ ] An incompatible manager pair, missing required role, or unsupported request produces a clear provider-facing diagnostic; OSAC does not route the work to an unrelated manager.
- [ ] Tenants use the same networking resources and workflows for VMs, managed clusters, and bare-metal workloads across compatible manager combinations.
- [ ] The Unified Networking design defines the shared resource semantics and OSAC orchestration, and references the standalone contract for exact manager integration requirements.

## 5. Dependencies

- **Unified Networking:** Its PRD and design define the shared networking resources, provider manager roles, and tenant-visible behavior that conforming implementations must preserve.
- **Provider manager implementations:** Fabric and K8s managers selected by a provider must meet the published requirements and declare compatibility with each other when both roles are selected.

---

## Provenance

Committed: commit @ prd 0.11.3 - 2bd6607, workspace docs/unified-networking-docs-structure @ 3d0f65e

> Authoring phases not recorded this session (commit-time snapshot only).

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"commit_only","workflow":"prd","workflow_version":"0.11.3","ai_workflows":"2bd6607","source_repo":"3d0f65e","source_repo_branch":"docs/unified-networking-docs-structure","commits_behind_main":0,"commits_ahead_main":19,"main_ref":"main","phases":["commit","commit","commit","commit"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
