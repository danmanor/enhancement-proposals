---
title: OSAC-1433 Network Manager Integration Contract Test Plan
last-updated: 2026-10-15
---

# Testplan — OSAC-1433 Network Manager Integration Contract

## Overview

- **Feature:** OSAC-1433 Network Manager Integration Contract
- **Requirements covered:** FR-1 through FR-14
- **Interface changes covered:** IC-1 through IC-7

This plan verifies manager registration, dispatch, fixed inputs/results, attachment descriptor mapping, and provider behavior. Structural checks do not substitute for backend conformance. A skipped required case is incomplete, never a pass.

### Test layers

| Layer | Purpose |
|---|---|
| Envelope | Exercise each fixed task's `osac_job_vars`, `osac_result`, typed outputs, and failure handling without a production backend. |
| Backend behavior | Verify isolation, L2/L3 reachability, DHCP, SecurityGroup policy when declared, ExternalIP/NAT behavior, idempotency, and cleanup on the provider backend. |
| Workload journeys | Exercise VM, bare-metal, and Cluster create/delete through their service extension flows. |

## Test Cases

### Manager registration and fixed dispatch

#### TC-REG-01: Reject invalid registration and NetworkClass selections

Verify missing manager names, wrong role labels, unsupported `contract_version`, unknown capabilities, missing required `ipv4`, and east-west enabled without `eastWestEthernet`. Each case makes NetworkClass unusable with a diagnostic and launches no AAP job. A role task outside the fixed task set is rejected rather than routed elsewhere.

#### TC-DISPATCH-01: Run every fixed role task through the generic playbook

Run a task against two implementations for a role. Verify OSAC selects the manager from NetworkClass, selects the task from the fixed dispatch table, and includes `osac.templates.<name>` with `tasks_from: <task>`. Both receive the same contract envelope; providers add no AAP job templates.

#### TC-CONTRACT-01: Review role requirements and task coverage

Manually verify that the contract lists every required task, trigger, input, typed output, failure rule, ordering constraint, settings source, capability meaning, and conformance behavior. Verify managers do not call one another or write OSAC workload objects.

### Attachment descriptor and VM creation

#### TC-DESC-01: Reject malformed attachment descriptors

For Kubernetes `apply_subnet_attachment`, reject: missing `namespace`; nonexistent namespace; namespace without the tenant label; role/binding pair not in the fixed table (including `primary` + `bridge` and `secondary` + `l2bridge`); `secondary` without `nad`; `primary` with non-empty `nad`; and any additional field such as `labels`, `annotations`, or raw `networks`. The SubnetAttachment stays non-ready and VMaaS creates no VM.

#### TC-DESC-02: Map valid descriptors for Linux and Windows VMs

For `primary`/`l2bridge` and `secondary`/`bridge`, verify VMaaS constructs exactly the `networks`, `interfaces`, and `autoattachPodInterface` fields in the contract mapping table in both Linux and Windows templates. Primary leaves `autoattachPodInterface` unset; secondary sets it to false and omits the default pod network.

#### TC-VM-01: Create the VM after the attachment is ready

Create an overlay SubnetAttachment with target containing only ComputeInstance kind and API UUID. While the request is being processed, verify no VirtualMachine exists. After descriptor validation and `NetworkAttachmentsReady=True`, VMaaS creates the VM with the mapped network fields and requested run strategy. The manager never reads or modifies the VM.

#### TC-POLICY-01: Select policy with the fixed ComputeInstance UUID label

Verify VMaaS stamps `osac.openshift.io/compute-instance-id: <uuid>` on the VM pod template and retains the existing hub-CR-name label. Verify manager policy selects the UUID label. A VM without that label receives no tenant traffic under the Subnet baseline.

#### TC-VM-DELETE-01: Delete the VM before attachment policy cleanup

Verify VMaaS deletes the VM and observes it gone before deleting SubnetAttachment. Kubernetes `delete_subnet_attachment` then removes policy and per-VM objects; a task failure retains the request finalizer and blocks further cleanup.

### Provider values, results, and capabilities

#### TC-VALUES-01: Persist and pass opaque provider values

Fabric returns `provider_values`; OSAC persists them in the owner ConfigMap and passes the merged values to a later Kubernetes task. An identical retry succeeds, a changed owner/key value fails without replacement, and the owner ConfigMap is deleted only after both role delete tasks succeed. Missing required keys are reported by the consuming task at runtime.

#### TC-RESULT-01: Validate result identity, typed outputs, and task errors

Reject a wrong resource ID/UID/generation, malformed `dhcp_leases[]`, malformed typed ExternalIP or ingress output, and malformed attachment descriptor before dependent work starts. Verify `osac_error.retryable=false` stops retries with its reason; true or absent follows configured backoff.

#### TC-SG-01: Report and verify SecurityGroup enforcement

When a role declares `securityGroups`, run the complete ingress/egress matrix on interfaces it owns and verify policy is installed before tenant traffic. When the capability is absent, status reports `NotEnforced`, never enforced. Cover ICMP and secondary-network constraints from the manager contract.

#### TC-IPAM-01: Avoid overlapping VM and bare-metal addresses on one Subnet

Attach one VM and one bare-metal host to one Subnet. Verify Fabric publishes its reserved DHCP ranges, Kubernetes allocation excludes `reserved_ipv4_cidrs`, and both roles exclude `osac_reserved_ipv4_cidrs`; addresses do not overlap.

#### TC-MANAGER-DELETE-01: Delete with the manager names captured at create

Change NetworkClass selection after resource creation, then delete. Verify the stored `status.networkManagers` names are used and no task is sent to the replacement manager.

#### TC-WORKLOAD-01: Exercise bare-metal and Cluster journeys

Run BMaaS and CaaS worker create/delete journeys with the same fixed manager contract. Verify SubnetAttachment is ready before the one host handoff reboot, `query_dhcp_lease` follows the reboot, and teardown waits for port restoration. Cluster flow uses the CaaS extension and does not add task fields to the shared envelope.

#### TC-BACKEND-01: Verify network behavior and cleanup

On each provider backend, verify overlapping CIDRs remain isolated across VirtualNetworks, same-VirtualNetwork Subnets route at L3 but remain separate L2 domains, workloads within one Subnet share its broadcast domain, DHCP works, declared SecurityGroups enforce their complete contract, ExternalIP/NAT work, and cleanup leaves no owned state.

### Non-Goals

- Validating that a selected Fabric Manager and Kubernetes Manager are compatible.
- Schemas or validation for values exchanged between managers; `provider_values` are opaque to OSAC.
- Supporting more than one hosting cluster.
- Supporting more than one tenant attachment per workload.

---

## Provenance

Authored: draft @ design 0.11.3 - 2bd6607, workspace main @ 1f3b63b82 (52 behind origin/main)
Final: manual-edit [manual] @ design 0.11.5 - 0c7ea35, workspace main @ 14c4d6f1c (17 behind origin/main)

> Context changed between draft and manual-edit.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.5","ai_workflows":"0c7ea35","source_repo":"14c4d6f1c","source_repo_branch":"main","commits_behind_main":17,"commits_ahead_main":0,"main_ref":"main","phases":["draft","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","manual-edit","revise","manual-edit"],"authoring_modes":["manual","skill"],"context_changed":true,"origin_untracked":false} -->
