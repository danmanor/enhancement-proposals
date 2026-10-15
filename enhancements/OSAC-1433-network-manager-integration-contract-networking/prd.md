---
title: network-manager-integration-contract
authors:
  - dmanor@redhat.com
creation-date: 2026-10-04
last-updated: 2026-10-15
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
| Date | 2026-10-15 |

## Contents

- [1. Problem Statement](#1-problem-statement)
- [2. Goals and Non-Goals](#2-goals-and-non-goals)
- [User Stories](#user-stories)
- [3. Requirements](#3-requirements)
  - [3.1 Functional Requirements](#31-functional-requirements)
  - [3.2 Non-Functional Requirements](#32-non-functional-requirements)
- [4. Dependencies](#4-dependencies)

## 1. Problem Statement

OSAC networking relies on provider-selected Fabric and Kubernetes Managers, but manager authors and providers need a clear statement of what each role must do and how OSAC uses it. Without that contract, providers must infer task responsibilities and workload sequencing, and a successful task response can be mistaken for proof that connectivity, addressing, isolation, or cleanup works. Providers need a published contract for each role and repeatable conformance tests that show whether an implementation fulfills it. [User]

## 2. Goals and Non-Goals

### 2.1 Goals

- A manager author can implement a role from its published contract alone.
- A provider can supply a Fabric Manager, a Kubernetes Manager, or both without provider-specific OSAC code, provided each role fulfills its contract.
- A conforming manager works with the workload service integrations that use the shared networking model.
- Providers can configure the backend settings and credentials their managers need.
- Managers can pass provider-defined values through OSAC without OSAC interpreting their meaning.
- OSAC reports clearly when a selected role does not enforce SecurityGroups.
- Providers can qualify manager behavior with repeatable tests rather than treating registration or a successful task response as proof of conformance.

### 2.2 Non-Goals

- Validating that a selected Fabric Manager and Kubernetes Manager are compatible.
- Schemas or validation for values exchanged between managers; `provider_values` are opaque to OSAC.
- Supporting more than one hosting cluster.
- Supporting more than one tenant attachment per workload.

## User Stories

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want one published contract for each manager role, so that I can implement or select a manager without guessing its tasks, inputs, outputs, or lifecycle requirements.
- As a Cloud Infrastructure Admin, I want to register managers and select them in NetworkClass, so that OSAC can dispatch networking work through the shared contract.
- As a Cloud Infrastructure Admin, I want to configure backend settings and credentials for networking fulfillment, so that selected managers can reach their backend services.
- As a Cloud Infrastructure Admin, I want a clear report when a selected role does not enforce SecurityGroups, so that OSAC does not imply that traffic policy is active.
- As a Cloud Infrastructure Admin, I want to run the published conformance suite, so that I can distinguish registration from working connectivity, allocation, policy, and cleanup.

### Tenant User

- As a Tenant User, I want to use the shared networking resources across supported manager implementations, so that changing provider implementations does not change my networking workflow.

## 3. Requirements

### 3.1 Functional Requirements

- **FR-1:** A provider can select a conforming Fabric Manager or Kubernetes Manager regardless of who supplies it; every implementation follows the same role contract. [User]
- **FR-2:** A manager author can determine from one published contract what each role requires and how OSAC uses that role. [User]
- **FR-3:** Tenants can use the same OSAC networking resources across workload services without selecting a provider backend. [Unified Networking PRD: FR-2, FR-6] [User]
- **FR-4:** An unknown manager name, wrong role, unsupported contract version, unknown capability, or request outside the role's task set fails with a clear diagnostic and is not routed to another manager. [User]
- **FR-5:** Managers can exchange provider-defined values through OSAC without OSAC interpreting them; values are write-once per owner and available to later tasks of both roles. [User]
- **FR-6:** OSAC rejects malformed manager results, including identity mismatches, malformed typed outputs such as the attachment descriptor, and changed provider values, before dependent work starts. [User]
- **FR-7:** Cloud Infrastructure Admins register managers with registration ConfigMaps that identify the manager name, role, contract version, capabilities, and optional limits. [User]
- **FR-8:** Cloud Infrastructure Admins can provide backend settings and credentials through the networking fulfillment group's ConfigMap and Secret, so manager jobs can reach their backend services. [User]
- **FR-9:** A Cloud Infrastructure Admin can distinguish manager registration and task success from behavioral conformance; qualification checks the selected implementation against shared networking behavior. [User]
- **FR-10:** Manager tasks are idempotent by stable API ID. OSAC runs at most one job at a time per resource and per workload interface, and starts no opposite action while a job may still be running. Failed work is retried with backoff. [User]
- **FR-11:** A Cloud Infrastructure Admin can run the published conformance suite and receive pass, fail, or incomplete results; a skipped required case is incomplete, not a pass. [User]
- **FR-12:** Supported workload interfaces receive tenant IPv4 configuration through DHCP. The role that owns an interface provides its addressing path, and providers qualify this behavior for their selected managers. [Unified Networking PRD: FR-1] [User]
- **FR-13:** A role declares whether it enforces SecurityGroups. When it does not, OSAC reports enforcement as `NotEnforced` and never reports it as enforced. [User]
- **FR-14:** VM provisioning starts only after the tenant network attachment is prepared. VMaaS creates the VM with its requested network already configured, so its first boot uses that network. [User]

### 3.2 Non-Functional Requirements

No separate non-functional requirements were specified for this work.

## 4. Dependencies

- **Unified Networking:** Its PRD and design define the shared networking resources, tenant-visible behavior, and SecurityGroup semantics that conforming implementations preserve.
- **Manager designs:** Each manager design lists the provider values it emits and requires. Providers document the selected pair's matching values and demonstrate the pair through conformance tests.
- **Workload service extensions:** Each extension defines when its controller creates and deletes SubnetAttachment requests and how it uses manager results.
- **Provider manager implementations:** Selected Fabric and Kubernetes Managers fulfill their assigned task sets and pass conformance for the behaviors and capabilities they declare.

---

## Provenance

Authored: draft @ prd 0.11.3 - 2bd6607, workspace main @ 1f3b63b82 (52 behind origin/main)
Final: manual-edit [manual] @ prd 0.11.5 - 0c7ea35, workspace main @ 14c4d6f1c (17 behind origin/main)

> Context changed between draft and manual-edit.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"prd","workflow_version":"0.11.5","ai_workflows":"0c7ea35","source_repo":"14c4d6f1c","source_repo_branch":"main","commits_behind_main":17,"commits_ahead_main":0,"main_ref":"main","phases":["draft","manual-edit","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","manual-edit","manual-edit","revise","manual-edit"],"authoring_modes":["manual","skill"],"context_changed":true,"origin_untracked":false} -->
