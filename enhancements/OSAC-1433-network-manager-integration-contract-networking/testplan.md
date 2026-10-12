# Test Plan — OSAC-1433 Network Manager Integration Contract

## Scope

This plan covers the generic OSAC implementation and the provider conformance
checks required before selecting a manager in a production NetworkClass. The
provider runs the behavioral checks against its actual AAP execution
environment and backend. OSAC can run the same cases against its supported
implementations; this plan does not define a hosted provider certification
service.

## Test Cases

### TC-1: Validate NetworkDataModel and NetworkData

**Requirements:** FR-5, FR-6, FR-7; IC-1, IC-5, IC-7

**Steps**

1. Create a NetworkDataModel with a supported owner scope and valid inline
   Draft 2020-12 JSON Schema.
2. Try a missing or wrong `$schema`, malformed schema, remote reference,
   unsupported owner scope, duplicate name, and schema outside configured
   limits.
3. Create a NetworkData value that matches the model, then submit a value that
   violates its schema or owner scope.
4. Try Update/Patch on the NetworkDataModel and runtime NetworkData, then try
   deleting a model while it is referenced.

**Expected results**

- The valid model and value are accepted. The value is validated before the
  fulfillment-service database write.
- Invalid model definitions and values are rejected with a diagnostic naming
  the resource and invalid field or value.
- NetworkData uniqueness is enforced by model name, owner kind, and stable owner resource ID;
  an identical retry returns the existing record and a different value is
  rejected.
- Update/Patch are rejected for NetworkDataModel and NetworkData. Delete is
  blocked while dependencies remain; only the OSAC service identity may
  create or delete runtime NetworkData.

### TC-2: Validate NetworkManager and NetworkClass compatibility

**Requirements:** FR-1, FR-2, FR-4, FR-6, FR-7, FR-10; IC-1, IC-3, IC-8

**Steps**

1. Create Fabric and Kubernetes NetworkManager registrations with valid roles,
   globally unique role names, model declarations, and capabilities.
2. Try duplicate `managerName` values across roles, invalid role-name format,
   a missing model reference, a model declared in the wrong role direction,
   and unknown capability enum values.
3. Create NetworkClasses with unresolved or wrong-role manager names and with
   Kubernetes input model names absent from Fabric outputs.
4. Enable Ethernet east-west first with a Fabric Manager that lacks
   `EAST_WEST_ETHERNET`, then with one that declares it. Also leave the
   NetworkClass capability false while the manager declares it.
5. Try Update/Patch on NetworkManager and NetworkClass. Try deleting a manager
   while a NetworkClass selects it, and try deleting that NetworkClass while a
   VirtualNetwork, ExternalIPPool, or unresolved operation intent is bound to
   it.
6. Resolve the old operation intent and remove all profile dependents. Delete
   the NetworkClass and then the now-unreferenced NetworkManager.
7. Register a replacement manager with the same `spec.managerName` and create
   a replacement NetworkClass. Deliver a delayed result from the old operation
   after the replacement exists.

**Expected results**

- Invalid registrations are rejected before they can be selected.
- NetworkClass compatibility uses exact NetworkDataModel names and rejects an
  unsatisfied Kubernetes input before persisting the profile or dispatching a
  job. OSAC does not use a manager-name allowlist.
- The only accepted optional capability is `EAST_WEST_ETHERNET`, on a Fabric
  Manager. NetworkClass may enable Ethernet east-west only when its selected
  Fabric Manager declares that capability. A manager declaration alone does
  not enable tenant requests.
- NetworkManager and NetworkClass reject Update/Patch. Deletion is blocked
  while a dependent resource or active/uncertain operation intent retains the
  exact profile or registration identity. NetworkClass deletion submits no
  provider job and removes no NetworkData.
- Once the old profile, dependents, and operation intents are drained, deletion
  succeeds. Reusing `managerName` creates a new registration API identity;
  the replacement NetworkClass captures that identity rather than aliasing the
  old registration.
- The delayed result from the old attempt cannot be matched to the new
  registration or profile, cannot create NetworkData, and cannot change
  readiness for a replacement resource. Recovery and retries continue to use
  the identities captured in the old operation journal.

### TC-3: Dispatch AAP roles with shared group settings and secrets

**Requirements:** FR-1, FR-2, FR-8; IC-2, IC-3

**Steps**

1. Install a test `osac.networking` collection in the configured AAP execution
   environment. It contains roles named for each registration's `managerName`
   and the task files listed in the resource action table.
2. Put a non-secret backend endpoint in the networking fulfillment group's
   ConfigMap and a test token in its Secret. Run representative Fabric and
   Kubernetes operations in that fulfillment group.
3. Inspect the job environment, generic playbook invocation,
   `osac_job_vars`, `osac_result`, and logs. Confirm both selected roles can
   read the group values.
4. Configure a test public CA bundle through the group ConfigMap and verify a
   manager task can use it for its backend client while certificate
   verification remains enabled.

**Expected results**

- OSAC derives the role FQCN as `osac.networking.<managerName>` and passes the
  exact task filename stem selected from the resource kind and lifecycle
  action. AAP launches the one generic networking job template.
- AAP job variables contain `osac_network_task_from`, but neither
  `osac_job_vars` nor `osac_result` contains an operation or attempt
  identifier. A top-level `osac_dispatch_attempt_id` in AAP `extra_vars` is
  reserved for OSAC's lost-launch recovery; manager roles must not use it to
  select behavior or return it. The provider cannot select a different task,
  role, or manager stage.
- Every manager job receives the networking fulfillment group's ConfigMap and
  Secret keys as environment variables. The values are shared across all
  manager roles in that group; the design provides no per-manager secret
  isolation. The selected role reads only the environment-variable names it
  requires.
- Backend settings and secrets are not fields on NetworkManager or
  NetworkClass and do not appear in `osac_job_vars`, NetworkData, or
  `osac_result`. Secret values do not appear in task logs or AAP result
  artifacts.
- A role reports a clear failure when a required environment variable is
  missing or malformed, and does not report the resource Ready. For a custom
  backend CA, the role writes the configured bundle to a job-local file and
  passes that file to its backend client with certificate verification
  enabled; this does not change the pod-wide trust store. Any private key
  material comes from the Secret, is written with restrictive permissions,
  and is removed after use.
- OSAC derives expected output model/owner keys from the Fabric Manager's
  `networkOutputs`, the owner scopes in the operation context, and existing
  NetworkData. The provider-authored Fabric role knows its manager's declared
  model names; OSAC does not pass the declaration or derived key set to AAP.
  The task must return each expected key exactly once and no other key.
- `network_owner_context` contains each live owner's stable API resource ID
  and current hub projection UID. The manager result identifies the owner by
  stable ID and carries the projection UID as a fence; OSAC rejects an owner
  or projection that is not in the dispatched context.
- Missing role/task content or invalid manager-specific environment settings
  fail the operation and do not report the resource Ready.
- The manager cannot choose a different task, role, manager stage, or target.

### TC-4: Validate manager results, NetworkData exchange, and job identity

**Requirements:** FR-2, FR-5, FR-6, FR-10; IC-4, IC-5, IC-8

**Steps**

1. Create a VirtualNetwork and then a Subnet with a Fabric Manager that
   declares VirtualNetwork- and Subnet-scoped outputs. Verify a Subnet-scoped
   model is not expected during VirtualNetwork creation, and an already
   stored VirtualNetwork value is not expected again during Subnet creation.
2. Inspect the Subnet AAP input. Verify it matches the canonical fixture:
   a unique top-level OSAC launch-correlation ID, the complete Subnet spec,
   the resolved parent VirtualNetwork, stable owner IDs with current
   projection UIDs, and no expected-output list. Confirm that the manager
   payload has no operation/attempt field. Return exactly the missing
   Subnet-scoped values.
3. In separate attempts, omit an expected value, duplicate a key, return an
   unexpected key, use an owner outside the context, mismatch the projection
   UID, or violate a registered schema.
4. Verify the Kubernetes Manager starts only after all expected values have
   been persisted, and receives only its declared, applicable inputs. Delete
   the Subnet and inspect cleanup ordering and retained NetworkData.
5. Retry a running and a failed task; inspect the AAP job IDs, operation
   journal, and returned artifacts.

**Expected results**

- Every successful manager task returns `artifacts.osac_result` with the
  stable `resourceID`, current projection `resourceUID`, `observedGeneration`,
  and `data.network_data`. The data list is empty when no new values are
  expected. The artifact does not contain operation or attempt IDs;
  extension-specific outputs use the same envelope.
- A NetworkData key is `(model_name, owner.kind, owner.resourceId)`, where
  `owner.resourceId` is the stable fulfillment-service API ID. The current hub
  `metadata.uid` is a separate projection fence and is not persisted as the
  owner key.
- OSAC derives expected Fabric model/owner keys internally. A successful
  Fabric task returns each expected key exactly once and no other key; each
  value matches its owner scope and JSON Schema. Existing values and values
  whose owner scope is not in the operation context are not expected again.
- OSAC validates the complete result before creating NetworkData through the
  fulfillment-service API. Managers do not write Kubernetes objects or call
  that API. A bad owner, projection, schema, duplicate key, or unexpected key
  prevents all writes and dependent Kubernetes dispatch.
- Subnet create runs Fabric then Kubernetes; delete runs Kubernetes then
  Fabric, retaining values until consumer cleanup succeeds. Parent-owned data
  remains until its own owner is deleted.
- OSAC polls the exact job ID and validates the result's resource ID,
  projection UID, and generation against that job's original `osac_job_vars`.
  The tracked AAP job ID and OSAC-reserved launch correlation bind the result
  to the journaled operation, manager stage, and task. Attachment retries are
  idempotent by stable target resource ID and logical interface; the current
  target projection UID fences the request context.

### TC-5: Verify the shared workload attachment contract

**Requirements:** FR-1, FR-3; IC-3, IC-6

**Steps**

1. Exercise VirtualNetwork and Subnet behavior with a Fabric Manager, and
   exercise an overlay attachment when a Kubernetes Manager is selected.
2. Apply and delete `SubnetAttachment` requests whose extension-defined
   targets normalize to `overlay` and `physical` interfaces. Do not make the
   manager contract test depend on a specific workload API or service.
3. Inspect manager job input and request status. Verify the request is the
   tracked `resource`, the referenced workload is the separate `target`, and
   `network_attachment` contains the resolved Subnet, one interface, and
   complete SecurityGroup rules.
4. Exercise matching, non-matching, and established-reply SecurityGroup
   traffic on each supported interface type.

**Expected results**

- Subnets in one VirtualNetwork are separate L2 broadcast domains and have L3
  routing through their VirtualNetwork. Separate VirtualNetworks remain
  isolated.
- Workloads on the same Subnet share its broadcast domain.
- SecurityGroup rules apply to each network attachment, combine as a union,
  allow established replies, and deny unmatched new traffic.
- The target-only request owns networking job history, status, and its cleanup
  finalizer. The workload extension owns its readiness mirror and waits for
  cleanup before continuing teardown.
- Every result identifies the SubnetAttachment request and its current
  projection; the separate target identity and normalized interface scope the
  backend attachment state.
- Policy is installed before traffic is enabled. A failed apply leaves the
  request non-ready and traffic unavailable; deletion removes policy and
  detaches the interface before workload teardown.

### TC-6: Verify FabricDomain and ExternalIP results

**Requirements:** FR-2, FR-4; IC-2, IC-4

**Steps**

1. Try creating a FabricDomain with Ethernet east-west disabled. Repeat with
   it enabled and the selected Fabric Manager declaring
   `EAST_WEST_ETHERNET`.
2. Allocate and release an ExternalIP, retrying allocation for the same
   stable resource ID. Submit a malformed address and an address outside the
   selected pool.

**Expected results**

- A disabled FabricDomain request fails before AAP dispatch. When enabled,
  OSAC invokes `create_fabric_domain` or `delete_fabric_domain` through the
  selected generic Fabric Manager; its implementation preserves
  VirtualNetwork routing and Subnet broadcast domains.
- ExternalIP allocation returns a canonical IPv4 address in
  `osac_result.data.external_ip.address`. OSAC validates it against the
  selected pool, then writes `ExternalIP.status.address`. Retrying the same
  stable resource ID returns the same reservation; invalid results are
  rejected, and capacity is released only after confirmed provider cleanup.
  Managers have no Kubernetes write credential.

### TC-7: Verify recovery, no-op results, and provider qualification

**Requirements:** FR-9, FR-10, FR-11; IC-4, IC-8

**Steps**

1. Simulate a lost AAP launch response after OSAC persists a `LaunchUnknown`
   attempt. Search the complete AAP scope by the exact persisted correlation
   in original `extra_vars`; restart reconciliation with one matching job,
   zero matches, and multiple matching jobs.
2. Return results with a wrong stable resource ID, projection UID, or
   generation. Separately associate a result with the wrong AAP attempt or
   manager stage. Test terminal and non-terminal prior attempts, plus a late
   result after a requested delete.
3. Configure an assigned task as a deliberate no-op. Return a valid result
   without claiming that allocation, connectivity, policy enforcement, or
   cleanup happened; then test a result that fabricates one of those outputs.
4. Run the published provider qualification selection with a conforming
   backend, with a failing assertion, and with a required case skipped because
   backend access is unavailable.
5. Stop processing after a full result is accepted but before all NetworkData
   writes finish. Retry result consumption, then request owner deletion while
   a separate create attempt is terminal but has no accepted output.

**Expected results**

- OSAC adopts and polls exactly one job matching the OSAC-reserved attempt
  correlation. Zero or
  incomplete searches remain unresolved; multiple matching jobs are all
  tracked and drained. OSAC does not launch a duplicate or opposite lifecycle
  action until every possible execution is resolved.
- Results whose resource, projection, or generation differs from the original
  job input are rejected. A result from a job not tracked for the journaled
  attempt, operation, manager stage, or task is also rejected. None can advance
  readiness, persist NetworkData, or release an address.
- A well-formed no-op may complete only when the task requires no backend
  change. It does not prove any advertised network behavior and cannot invent
  address allocation or successful cleanup.
- A result accepted before a Fulfillment write failure is replayed from the
  durable journal without another AAP launch or recomputing the expected
  outputs. A create superseded by deletion publishes no new NetworkData or
  consumer input; cleanup uses its captured API owner and manager binding.
- Qualification reports `pass`, `fail`, or `incomplete`; a skipped required
  behavior is incomplete and is not counted as a pass. The report retains the
  tested OSAC/AAP versions, manager registrations, selection, job references,
  and redacted diagnostics. OSAC contract tests and provider qualification use
  the same complete serialized fixtures for every shared task and role
  variant, including negative cases for missing fields, invalid owners,
  unexpected context, and wrong types.

## Provider Evidence

Before production selection, the Cloud Infrastructure Admin verifies every
base task set and normalized interface type assigned to each selected role,
plus each enabled optional capability. Retain the qualification report,
selected manager and model registrations, tested OSAC/AAP versions, AAP job
details with secret values redacted, and observations of resulting backend
behavior. The report distinguishes pass, fail, and incomplete cases. The
registration APIs validate declarations and data shapes; passing registration
validation alone is not evidence of behavioral conformance.

---

## Provenance

Authored: revise @ design 0.11.3 - 2bd6607, workspace main @ d165396
Final: revise @ design 0.11.5 - 1c6e8de, workspace docs/OSAC-1433-network-manager-provider-guide @ eabff3b59 (92 behind origin/main, dirty)

> Context changed between revise and revise.

> This document's phase history does not include an initial /draft — structure was not verified against the template from origin.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.5","ai_workflows":"1c6e8de","source_repo":"eabff3b59 (dirty)","source_repo_branch":"docs/OSAC-1433-network-manager-provider-guide","commits_behind_main":92,"commits_ahead_main":2,"main_ref":"main","phases":["revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","manual-edit","revise","revise","revise","revise"],"authoring_modes":["manual","skill"],"context_changed":true,"origin_untracked":true} -->
