# Testplan — OSAC-1433-network-manager-integration-contract

## Overview

- **Feature:** Network Manager Integration Contract
- **Total test cases:** 15
- **Requirements covered:** 7 of 7 functional requirements
- **Interface changes covered:** 7 of 7

## Test Cases

### FR-1: Source-neutral manager conformance

#### TC-FR1-01: Invoke a conforming collection outside the OSAC collection

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-1, IC-2 | E2E | critical | automated |

##### Preconditions

- A test Fabric Manager registration points to an installed collection role using a new fully qualified name outside the OSAC collection.
- The role implements every Fabric operation and target assigned by the contract.
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

#### TC-FR2-01: Reject invalid NetworkDataModel and NetworkManager objects

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-1, IC-7 | Integration | critical | automated |

##### Preconditions

- NetworkDataModel and NetworkManager CRDs and the fulfillment-service provider APIs are deployed.
- A test AAP provider records whether an AAP job was created.
- A controlled HTTP endpoint records whether schema validation attempts a network fetch.

##### Steps

1. Create valid OSAC and provider-defined NetworkDataModel objects through the fulfillment-service API, including a schema with the required `$schema` key and a local fragment reference. Read the object back and verify the schema is preserved unchanged.
2. Attempt models with an unsupported owner scope, malformed JSON Schema, missing or unsupported `$schema` dialect, unsupported vocabulary, remote or file `$ref`/`$dynamicRef`, an external `$id`, a schema beyond OSAC's size, nesting, or evaluation-work limits, or a name reserved to OSAC.
3. Create Fabric and Kubernetes NetworkManager objects through the fulfillment-service API, each referencing existing model names.
4. Attempt manager objects with an unknown or duplicate model name, duplicate (role, managerName), invalid role, role-inappropriate input/output field, missing required list, malformed implementation reference.
5. Create a NetworkClass that selects the valid pair; observe the create response and AAP job count.

##### Expected Results

- Valid model and manager objects are returned by provider API Get and List and appear as the same objects in the Kubernetes backing store.
- The fulfillment service rejects every invalid model or manager before it can be selected and identifies the object, invalid field, and model name when relevant. Kubernetes preserves the inline schema object and independently validates the outer CRD fields; it does not interpret JSON Schema keywords.
- Rejected remote references cause no request to the controlled endpoint.
- A role's required declaration may be an empty list; an omitted declaration is rejected.
- No AAP job is created for an invalid object or profile.

#### TC-FR2-02: Enforce IPv4-only behavior without per-manager family declarations

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-3, IC-4 | Integration | high | automated |

##### Preconditions

- Valid Fabric and optional Kubernetes NetworkManager registrations exist without any IP-family field.
- The tenant networking API and provider dispatch are available.

##### Steps

1. Create a NetworkClass using the registered managers, with no per-manager address-family declarations.
2. Inspect the NetworkClass API response for manager-derived IP-family output.
3. Submit valid IPv4 networking input.
4. Submit IPv6 and dual-stack values through the networking API.

##### Expected Results

- The NetworkClass is accepted based on valid role references and data-model compatibility; manager IP-family declarations are not required.
- The NetworkClass exposes no manager-derived IP-family fields.
- IPv4 input is accepted and reaches the assigned manager.
- IPv6 and dual-stack input is rejected before provider dispatch.
#### TC-FR2-03: Verify complete role operation and target coverage

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-2, IC-4, IC-6 | Integration | critical | automated |

##### Preconditions

- A contract conformance harness can load candidate Fabric and Kubernetes Manager collections.
- The fixed operation table and target combinations are available to the harness.

##### Steps

1. Enumerate every operation and target combination assigned to each candidate manager role.
2. Invoke every required task entry point with a valid resource fixture.
3. Check each task's desired-state behavior, required result, retry safety, and error behavior.

##### Expected Results

- Each collection provides every operation and workload target assigned to its role.
- Repeated create/apply converges without duplicate backend objects; repeated delete of absent state succeeds.
- Each successful task returns an `osac_result` artifact whose operation, resource UID, and observed generation match the dispatched operation. `data.network_data` is empty unless a Fabric task publishes values requested for its operation; `dhcp_lease.query` returns the required `leases` artifact instead.
- Missing, malformed, stale, or mismatched result envelopes are rejected and do not advance readiness or release capacity.
- Each task meets the input, result, tenancy, idempotency, and failure requirements.

#### TC-FR2-04: Validate ExternalIP and DHCP lease results

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-4 | Integration | high | automated |

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

#### TC-FR2-05: Apply SecurityGroup rules to resolved workload attachments

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-2, IC-6 | Integration | critical | automated |

##### Preconditions

- A test Fabric Manager and Kubernetes Manager record their operation inputs and can enforce test rules on physical and overlay interfaces.
- A tenant has a Ready VirtualNetwork, Subnet, and two Ready SecurityGroups with distinct ingress and egress rules.

##### Steps

1. Create a ComputeInstance selecting the Subnet and both SecurityGroups; inspect the Kubernetes Manager task input and traffic behavior.
2. Create a BaremetalInstance and a Cluster selecting the same resources; inspect the Fabric Manager input for the physical interface and each Cluster node-set interface.
3. Delete each workload; inject one policy-removal failure, then allow retry to succeed and inspect teardown ordering.

##### Expected Results

- OSAC sends ComputeInstance policy to the Kubernetes Manager and BaremetalInstance/Cluster policy to the Fabric Manager; managers do not call each other.
- Each payload includes the owning workload UID, resolved Subnet and interface data, SecurityGroup UIDs, and complete normalized ingress/egress rules. Rule references are not passed as labels or left for the manager to look up.
- The combined allow rules are a union, unmatched new traffic is denied, and established connections allow return traffic.
- The manager installs policy before enabling interface traffic. A failed apply leaves the workload non-ready and the attachment closed to workload traffic.
- Delete sends the same normalized attachment, removes policy and detaches before workload teardown, and retries idempotently. A failed delete retains the finalizer and blocks teardown.

### FR-3: Provider selection with one tenant networking API

#### TC-FR3-01: Select managers whose declared network data matches

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-1, IC-2, IC-3 | E2E | high | automated |

##### Preconditions

- Fabric and Kubernetes Manager collections from different sources are installed in the AAP execution environment.
- The Fabric registration declares VXLAN L3 VNI, VXLAN L2 VNI, and reserved IPv4 CIDR model names as outputs; the Kubernetes registration declares those same NetworkDataModel names as inputs.

##### Steps

1. Configure NetworkClass with both logical manager names.
2. Create a VirtualNetwork and Subnet through the tenant API.
3. Observe the selected AAP roles, resolved inputs, and backend state.

##### Expected Results

- OSAC accepts the pair because each Kubernetes input model name is declared by the Fabric Manager.
- Both managers receive the shared resource shape; the Kubernetes Manager receives schema-validated values from both the parent VirtualNetwork and current Subnet scopes.
- The tenant API does not expose or require a backend selector.

#### TC-FR3-02: Validate schema-checked, owner-scoped network outputs and inputs

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-5 | Integration | critical | automated |

##### Preconditions

- The test Fabric Manager declares VirtualNetwork- and Subnet-scoped output models, and the selected Kubernetes Manager declares the applicable model names as inputs.
- The Fabric task can return `osac_result.data.network_data`; it has no Kubernetes API or fulfillment-service write credential.
- The Kubernetes test role records its `osac_job_vars.network_data` and operation order.

##### Steps

1. Create the Subnet and inspect the resulting NetworkData records, Kubernetes inputs, and readiness.
2. Verify the parent VirtualNetwork VNI is stored under the VirtualNetwork UID and the L2 VNI and reserved CIDR array are stored under the Subnet UID.
3. Attempt to return an undeclared model, a duplicate model/owner key, an out-of-context owner UID, and a schema-invalid value; verify each result is rejected before any NetworkData write or Kubernetes dispatch.
4. Delete the Subnet and inspect Kubernetes and Fabric task order, NetworkData lifetime, and inputs.

##### Expected Results

- Each NetworkData object records one model name, owner kind and UID, and a JSON value that matches the registered schema.
- A manager cannot create or update NetworkData directly; OSAC persists values only after validating the complete result.
- OSAC validates model name, owner scope, and JSON Schema, then passes only applicable Kubernetes Manager inputs with their owner kind and UID.
- A Kubernetes Subnet operation receives both parent-VirtualNetwork and current-Subnet values when required; it does not receive unrelated Fabric outputs.
- The Subnet becomes Ready only after both create stages succeed.
- Delete runs Kubernetes cleanup before Fabric cleanup and supplies the same required inputs while the NetworkData records remain available.
- Subnet-scoped NetworkData is removed after successful Subnet Fabric deletion; the VirtualNetwork-scoped value remains until VirtualNetwork deletion.

#### TC-FR3-03: Recover from partial Subnet create and delete failures

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-5 | Integration | high | automated |

##### Preconditions

- Fabric create succeeds and OSAC stores valid outputs for both resource scopes.
- The AAP test provider can inject Kubernetes create and Fabric delete failures independently.

##### Steps

1. Fail the Kubernetes create task after Fabric success, then allow a retry to succeed.
2. Begin deletion and fail Kubernetes cleanup; later allow cleanup and Fabric delete to succeed.
3. Observe NetworkData lifetime, job history, role order, and final resource state.

##### Expected Results

- Both NetworkData records remain available after Kubernetes create failure, and retry uses the same values without creating duplicate Fabric resources.
- Fabric deletion does not begin until Kubernetes cleanup succeeds.
- Subnet and VirtualNetwork values remain available while their consumers may retry; Subnet data is removed after its resource is deleted and VirtualNetwork data persists until parent deletion.
- All task retries are idempotent and the resource does not report Ready or deleted prematurely.

### FR-4: Reject unsatisfied network data or unavailable work before dispatch

#### TC-FR4-01: Reject a NetworkClass with an unsatisfied network input

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-1, IC-3 | Integration | critical | automated |

##### Preconditions

- Two valid NetworkManager objects exist.
- The Kubernetes Manager requires a model name that the Fabric Manager does not declare.

##### Steps

1. Create NetworkClass selecting the incompatible Fabric and Kubernetes manager names.
2. Inspect the create response and NetworkClass store.
3. Verify the AAP job count.

##### Expected Results

- NetworkClass creation is rejected synchronously with the unsatisfied model name and both manager names.
- The invalid NetworkClass is not persisted.
- No resource operation or AAP job can use the invalid profile.

#### TC-FR4-02: Reject work requiring an unselected or unsupported role/target

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-3, IC-4 | Integration | critical | automated |

##### Preconditions

- NetworkClass has a valid Fabric Manager but no Kubernetes Manager.
- The test AAP provider records whether an operation job was created.

##### Steps

1. Submit a VM request whose selected networking path requires Kubernetes-side Subnet networking.
2. Separately submit a role/target combination not assigned by the fixed operation table.
3. Observe the request result and AAP job count.

##### Expected Results

- OSAC reports the missing role or unsupported operation-target combination with a specific diagnostic.
- OSAC does not send that work to the Fabric Manager or another unrelated implementation.
- No AAP job is created for the rejected work.

### FR-5: Add a provider-defined data model without manager-specific OSAC code

#### TC-FR5-01: Exchange a provider-defined structured model

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-1, IC-3, IC-7 | E2E | high | automated |

##### Preconditions

- Generic catalog loading, schema validation, and model resolution are available.
- A provider-defined `acme-networking-subnet-segment` model is registered with Subnet owner scope and an object schema.
- Fabric and Kubernetes test roles can publish and consume the registered model.

##### Steps

1. Register the model and declare its name in the Fabric Manager's `networkOutputs` and the Kubernetes Manager's `networkInputs`.
2. Select both managers through NetworkClass and create a Subnet.
3. Have Fabric publish an object that satisfies the registered schema under the Subnet UID.
4. Observe the Kubernetes AAP input and its backend mapping.

##### Expected Results

- OSAC accepts the manager pair because both declare the same registered model name.
- The Fabric JSON value passes generic schema validation and remains owned by the Subnet UID.
- OSAC passes that value to the Kubernetes role with the same model name; the role maps it to its implementation's backend fields.
- The new model and manager integration require provider catalog/manager configuration and Ansible content, with no manager-specific Go change or tenant API change.

### FR-6: Validate model definitions, scope, and produced values

#### TC-FR6-01: Validate profile-scoped output against its model

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-5, IC-7 | Integration | critical | automated |

##### Preconditions

- A valid model with `NetworkClass` owner scope and a JSON object schema is registered.
- The selected managers both declare the model name.
- The operator is authorized to create NetworkData from manager results; the Fabric task has no NetworkData write credentials.

##### Steps

1. During one provisioning attempt, return the model with a VirtualNetwork owner UID even though its registered owner scope is NetworkClass.
2. Observe owner-scope validation and verify that the Kubernetes job does not start.
3. Retry with the correct NetworkClass UID but an object that violates the registered schema.
4. Observe schema validation and verify that the Kubernetes job does not start.
5. Retry with a schema-valid object under the NetworkClass UID and inspect the Kubernetes job's `network_data`.

##### Expected Results

- The wrong owner scope and schema-invalid value each fail in separate attempts and block Kubernetes dispatch.
- The diagnostic identifies the model name and NetworkClass owner UID.
- The valid JSON value is stored as one NetworkData object under the NetworkClass UID and resolved with `resource_kind: NetworkClass`.
- The Kubernetes role receives only the declared input and does not receive the Fabric writer credential.

#### TC-FR6-02: Enforce immutable registry lifecycle and dependency-safe deletion

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-1, IC-3, IC-7 | Integration | high | automated |

##### Preconditions

- A valid NetworkDataModel is referenced by a NetworkManager, and an owner-scoped NetworkData object contains a valid value for that model.
- A valid NetworkClass selects that manager and has no VirtualNetworks.

##### Steps

1. Attempt Update and Patch on the NetworkDataModel, NetworkManager, and NetworkClass objects.
2. Attempt to delete the NetworkDataModel while its NetworkManager and NetworkData references remain, and attempt to delete the selected NetworkManager while NetworkClass still selects it.
3. Delete NetworkClass and verify that its owner-scoped NetworkData is removed through normal owner cleanup.
4. Delete NetworkManager, then delete NetworkDataModel.

##### Expected Results

- Provider Update and Patch requests are rejected for each object.
- Deletion is blocked while a dependent object references the target, and the response identifies the blocking object.
- NetworkDataModel deletion is blocked while its NetworkManager or NetworkData references remain; NetworkManager deletion is blocked while NetworkClass selects it.
- Owner cleanup removes NetworkClass-scoped NetworkData only after dependent manager work succeeds. Reverse-order deletion succeeds after owner cleanup and object references are gone.

### FR-7: Manage provider registrations through OSAC

#### TC-FR7-01: Manage registry objects through the fulfillment-service API

| Interface Change | Test Level | Priority | Automation |
|-----------------|------------|----------|------------|
| IC-1 | Integration | high | automated |

##### Preconditions

- The fulfillment-service APIs and backing NetworkDataModel, NetworkManager, and NetworkData CRDs are deployed.
- The caller is authorized as a Cloud Infrastructure Admin.

##### Steps

1. Create a NetworkDataModel and use Get and List to inspect it.
2. Create a NetworkManager that references the model and use Get and List to inspect it.
3. Attempt Update and Patch through the provider APIs.
4. Verify provider administrators may Get/List NetworkData but cannot Create, Update, Patch, or Delete it.
5. Delete a NetworkData value through OSAC owner cleanup, then delete the NetworkManager and NetworkDataModel through the provider APIs.
6. Repeat a registry read or delete with an unauthorized tenant identity.

##### Expected Results

- Create, Get, List, and Delete operate on canonical fulfillment-service records; the existing controller path projects the same objects as hub CRDs for the operator.
- NetworkData is created only by the OSAC service after schema and owner validation; owner cleanup deletes it after dependent manager work completes.
- Update and Patch are rejected; dependent-object deletion is blocked until dependencies are removed.
- Unauthorized callers cannot create, read, or delete provider registrations.

## Gaps

### Requirement Coverage Gaps

All PRD functional requirements have test cases.

### Interface Change Coverage Gaps

All design interface changes are exercised by test cases.

## Summary

| Metric | Count |
|--------|-------|
| Total test cases | 15 |
| Critical | 8 |
| High | 7 |
| Medium | 0 |
| Low | 0 |
| Automated | 15 |
| Manual | 0 |
| Requirements with test cases | 7 / 7 |
| Interface changes with test cases | 7 / 7 |

---

## Provenance

Authored: draft @ design 0.11.3 - 2bd6607, workspace main @ 1f3b63b82 (52 behind origin/main)
Final: revise @ design 0.11.3 - 2bd6607, workspace main @ 1f3b63b82 (99 behind origin/main, dirty)

> Context changed between draft and revise.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.3","ai_workflows":"2bd6607","source_repo":"1f3b63b82 (dirty)","source_repo_branch":"main","commits_behind_main":99,"commits_ahead_main":0,"main_ref":"main","phases":["draft","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
