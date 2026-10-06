# Testplan — OSAC-1433-network-manager-integration-contract

## Overview

- **Feature:** Network Manager Integration Contract
- **Total test cases:** 10
- **Requirements covered:** 4 of 4 functional requirements
- **Interface changes covered:** 5 of 5

## Test Cases

### FR-1: Source-neutral manager conformance

#### TC-FR1-01: Invoke a conforming collection outside the OSAC collection

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-1, IC-2 | critical | automated |

##### Preconditions

- A test Fabric Manager registration declares contract v1 and an installed collection role using a new fully qualified name outside the OSAC collection.
- The role implements every Fabric operation and target assigned by contract v1.
- The AAP execution environment contains the test collection.

##### Steps

1. Select the test manager by its logical name in NetworkClass.
2. Create a VirtualNetwork through the existing tenant API.
3. Observe the AAP job and test backend state.

##### Expected Results

- OSAC invokes the role named by `implementationRef`, regardless of the manager's logical name or collection publisher.
- The task receives the fixed operation envelope and OSAC resource.
- The test manager reconciles the resource and OSAC reports success after AAP completes.
- No manager-specific Go build or tenant API change is needed for the test implementation.

### FR-2: Exact manager implementation requirements

#### TC-FR2-01: Reject an invalid manager registration

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-1 | critical | automated |

##### Preconditions

- The operator watches manager registration ConfigMaps.
- A test AAP provider records whether an AAP job was created.

##### Steps

1. Create registrations with an unsupported or missing contractVersion, malformed or missing implementationRef, missing capabilities or compatibleManagers, an unknown role label, an invalid capability such as `ipv6` or `dualStack`, a malformed compatibleManagers value, or duplicate logical names within a role.
2. Select each registration from a NetworkClass.
3. Observe NetworkClass status and AAP job count.

##### Expected Results

- OSAC identifies the ConfigMap and invalid field in its diagnostic.
- The NetworkClass is not Ready while a selected registration is invalid.
- OSAC creates no AAP job for the invalid registration.

#### TC-FR2-02: Derive effective NetworkClass capabilities

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-1 | high | automated |

##### Preconditions

- A Fabric Manager and a Kubernetes Manager declare `ipv4`; the Fabric Manager declares `dpuSupport` and the Kubernetes Manager does not.
- The operator exposes NetworkClass capability output and provider `disable_capabilities` input.

##### Steps

1. Create a NetworkClass using both managers and inspect its capability output.
2. Set `disable_capabilities.dpuSupport` and inspect the output again.
3. Try to disable IPv4 and observe NetworkClass readiness.

##### Expected Results

- OSAC intersects capabilities from selected managers: `supportsIpv4` is true, `supportsIpv6` and `supportsDualStack` are false, and `dpuSupport` is false because the Kubernetes Manager does not declare it.
- Disabling DPU support keeps `dpuSupport` false and does not alter operation routing.
- A NetworkClass that disables contract-required IPv4 is invalid and is not Ready.

#### TC-FR2-03: Verify complete role operation and target coverage

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-2, IC-4 | critical | automated |

##### Preconditions

- A contract conformance harness can load candidate Fabric and Kubernetes Manager collections.
- The v1 operation table and target combinations are available to the harness.

##### Steps

1. Enumerate every operation and target combination assigned to each candidate manager role.
2. Invoke every required task entry point with a valid resource fixture.
3. Check each task's desired-state behavior, required result, retry safety, and error behavior.

##### Expected Results

- Each collection provides every operation and workload target assigned to its role.
- Repeated create/apply converges without duplicate backend objects; repeated delete of absent state succeeds.
- Each successful task returns an `osac_result` artifact whose schema version, operation, resource UID, and observed generation match the dispatched operation and whose `data` is empty, except `dhcp_lease.query`, which returns the required `leases` artifact.
- Missing, malformed, stale, or mismatched result envelopes are rejected and do not advance readiness or release capacity.
- Each task meets the v1 input, result, tenancy, and failure requirements.

#### TC-FR2-04: Validate ExternalIP and DHCP lease results

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-4 | high | automated |

##### Preconditions

- A conforming Fabric Manager implements ExternalIP allocation and DHCP lease lookup for a supported target.
- The test backend returns a stable address and a matching lease.

##### Steps

1. Reconcile the same ExternalIP twice.
2. Run `dhcp_lease.query` for an attachment with a known lease.
3. Repeat with a missing and an ambiguous lease result.
4. Observe the annotation, AAP artifacts, and workload status.

##### Expected Results

- The manager writes the same durable address to `osac.openshift.io/allocated-address` on both ExternalIP reconciliations.
- The `leases` artifact contains `subnet_ref`, `interface`, `ip_address`, and `mac_address`.
- A missing or ambiguous match fails with a diagnostic and is not reported as a successful lookup.

### FR-3: Provider selection with one tenant networking API

#### TC-FR3-01: Select mutually compatible managers from different sources

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-1, IC-2, IC-3 | high | automated |

##### Preconditions

- A Fabric Manager and a Kubernetes Manager from different collection sources are installed in the AAP execution environment.
- Each registration uses contract v1 and names the other in `compatibleManagers`.

##### Steps

1. Configure NetworkClass with both logical manager names.
2. Create a VirtualNetwork and Subnet through the existing tenant API.
3. Observe the selected AAP roles and backend state.

##### Expected Results

- OSAC accepts the mutually declared pair and invokes each registered implementation for its assigned role.
- Both managers receive the same shared resource shape; the Kubernetes Manager receives `l2_vni`, `l3_vni`, and `fabric_reserved_range` for the Subnet.
- The tenant API does not expose or require a backend selector.

#### TC-FR3-02: Create and delete a Subnet with shared Fabric outputs

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-5 | critical | automated |

##### Preconditions

- The test Fabric task writes the namespaced output ConfigMap with string values for `l2_vni`, `l3_vni`, and `fabric_reserved_range`.
- The Kubernetes test role records its inherited AAP variables and operation order.

##### Steps

1. Create the Subnet and inspect the Fabric output ConfigMap, Kubernetes AAP input variables, and readiness.
2. Delete the Subnet and inspect Kubernetes and Fabric task order, ConfigMap lifetime, and their inputs.

##### Expected Results

- The Fabric role writes the exact flat ConfigMap keys `l2_vni`, `l3_vni`, and `fabric_reserved_range`; no VLAN field or Subnet status handoff is used.
- OSAC requires all three keys, validates and normalizes both VNI values, and passes those values as top-level AAP extra variables to Kubernetes.
- Kubernetes receives the exact values returned by Fabric and does not allocate replacement VNIs.
- The Subnet becomes Ready only after both create stages succeed.
- Delete runs Kubernetes cleanup before Fabric cleanup and passes the same three values to Kubernetes while the ConfigMap still exists.
- Fabric removes the output ConfigMap only after Kubernetes cleanup has succeeded.

#### TC-FR3-03: Recover from partial Subnet create and delete failures

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-5 | high | automated |

##### Preconditions

- Fabric create succeeds and writes a valid output ConfigMap.
- The AAP test provider can inject Kubernetes create and Fabric delete failures independently.

##### Steps

1. Fail the Kubernetes create task after Fabric success, then allow a retry to succeed.
2. Begin deletion and fail Kubernetes detach; later allow detach and Fabric delete to succeed.
3. Observe output ConfigMap lifetime, job history, role order, and final resource state.

##### Expected Results

- The output ConfigMap remains available after Kubernetes create failure, and retry uses the same values without creating a second Fabric segment.
- Fabric deletion does not begin until Kubernetes detach succeeds.
- The output ConfigMap remains available while Kubernetes cleanup retries; Fabric removes it after both roles succeed.
- All task retries are idempotent and the resource does not report Ready or deleted prematurely.

### FR-4: Reject incompatible or unavailable work before dispatch

#### TC-FR4-01: Reject a manager pair without mutual compatibility declarations

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-1, IC-3 | critical | automated |

##### Preconditions

- The Fabric registration lists `cudn_evpn` and the Kubernetes registration lists `netris`.
- A second configuration selects `agentless_net` with `cudn_evpn`, and a third declares compatibility in only one direction.
- The test AAP provider records created jobs.

##### Steps

1. Select each pair in NetworkClass.
2. Observe readiness diagnostics and AAP job count.

##### Expected Results

- Netris and CUDN EVPN are accepted when both declarations are present.
- Agentless VLAN and CUDN EVPN are rejected when either side does not list the other.
- A one-sided declaration is rejected.
- Diagnostics name the two manager registrations and missing compatibility declaration; no AAP job starts.

#### TC-FR4-02: Reject work requiring an unselected or unsupported role/target

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-3, IC-4 | critical | automated |

##### Preconditions

- NetworkClass has a valid Fabric Manager but no Kubernetes Manager.
- The test AAP provider records whether an operation job was created.

##### Steps

1. Submit a VM request whose selected networking path requires Kubernetes-side Subnet networking.
2. Separately submit a role/target combination not assigned by the v1 operation table.
3. Observe the request result and AAP job count.

##### Expected Results

- OSAC reports the missing role or unsupported operation-target combination with a specific diagnostic.
- OSAC does not send that work to the Fabric Manager or another unrelated implementation.
- No AAP job is created for the rejected work.

## Gaps

### Requirement Coverage Gaps

All PRD functional requirements have test cases.

### Interface Change Coverage Gaps

All design interface changes are exercised by test cases.

## Summary

| Metric | Count |
|--------|-------|
| Total test cases | 10 |
| Critical | 6 |
| High | 4 |
| Medium | 0 |
| Low | 0 |
| Automated | 10 |
| Manual | 0 |
| Requirements with test cases | 4 / 4 |
| Interface changes with test cases | 5 / 5 |
