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
4. Try Update/Patch and try deleting a model while it is referenced.

**Expected results**

- The valid model and value are accepted. The value is validated before the
  fulfillment-service database write.
- Invalid model definitions and values are rejected with a diagnostic naming
  the resource and invalid field or value.
- NetworkData uniqueness is enforced by model name, owner kind, and owner UID;
  an identical retry returns the existing record and a different value is
  rejected.
- Update/Patch are rejected. Delete is blocked while dependencies remain.

### TC-2: Validate NetworkManager and NetworkClass compatibility

**Requirements:** FR-1, FR-2, FR-4, FR-6, FR-7; IC-1, IC-3

**Steps**

1. Create Fabric and Kubernetes NetworkManager registrations with valid roles,
   globally unique role names, model declarations, configuration, and
   credential IDs.
2. Try duplicate `managerName` values across roles, invalid role-name format,
   a missing model reference, a model
   declared in the wrong role direction, invalid credential IDs, and unknown
   capability enum values.
3. Create NetworkClasses with unresolved or wrong-role manager names and with
   Kubernetes input model names absent from Fabric outputs.
4. Enable Ethernet east-west first with a Fabric Manager that lacks
   `EAST_WEST_ETHERNET`, then with one that declares it. Also leave the
   NetworkClass capability false while the manager declares it.
5. Try Update/Patch and delete registrations while referenced.

**Expected results**

- Invalid registrations are rejected before they can be selected.
- NetworkClass compatibility uses exact NetworkDataModel names and rejects an
  unsatisfied Kubernetes input before persisting the profile or dispatching a
  job. OSAC does not use a manager-name allowlist.
- The only accepted optional capability is `EAST_WEST_ETHERNET`, on a Fabric
  Manager. NetworkClass may enable Ethernet east-west only when its selected
  Fabric Manager declares that capability. A manager declaration alone does
  not enable tenant requests.
- Immutable objects reject Update/Patch and block Delete while referenced.

### TC-3: Dispatch the generic AAP role with isolated settings and credentials

**Requirements:** FR-1, FR-2, FR-8; IC-2, IC-3

**Steps**

1. Install a test `osac.networking` collection in the configured AAP execution
   environment. It contains roles named for each registration's `managerName`
   and the task files listed in the resource action table.
2. Run representative Fabric and Kubernetes operations using different
   `configuration` objects and different AAP Credential IDs.
3. Inspect the generic playbook invocation, AAP launch request, and each
   manager's `osac_job_vars`.

**Expected results**

- OSAC derives the role FQCN as `osac.networking.<managerName>` and passes the
  exact task filename stem selected from the resource kind and lifecycle
  action. AAP launches the one generic networking job template.
- AAP job variables contain `osac_network_task_from`, but neither
  `osac_job_vars` nor `osac_result` contains an operation identifier. The
  provider cannot select a different task, role, or manager stage.
- `manager.configuration` contains the selected manager's non-secret values.
  AAP receives only that registration's credential IDs; credential values do
  not appear in job variables, NetworkData, or result artifacts.
- OSAC derives expected output model/owner keys from the Fabric Manager's
  `networkOutputs`, the owner scopes in the operation context, and existing
  NetworkData. The provider-authored Fabric role knows its manager's declared
  model names; OSAC does not pass the declaration or derived key set to AAP.
  The task must return each expected key exactly once and no other key.
- `network_owner_context` contains the live owner kinds and hub UIDs for the
  operation. The manager result uses those identities, and OSAC rejects an
  owner that is not in that context.
- Missing role/task content, denied credentials, or invalid manager-specific
  settings fail the operation and do not report the resource Ready.
- The manager cannot choose a different operation or target.

### TC-4: Validate manager results, NetworkData exchange, and job identity

**Requirements:** FR-2, FR-5, FR-6; IC-4, IC-5

**Steps**

1. Create a VirtualNetwork and then a Subnet with a Fabric Manager that
   declares VirtualNetwork- and Subnet-scoped outputs. Verify that a
   Subnet-scoped model is not expected during VirtualNetwork creation, and
   that the already stored VirtualNetwork value is not expected again during
   Subnet creation.
2. Inspect the Subnet AAP input and confirm it contains existing NetworkData
   and the live NetworkClass, VirtualNetwork, and Subnet identities in
   `network_owner_context`, but no expected-output list. Return exactly the
   missing Subnet-scoped values. In separate attempts, omit an expected
   value, duplicate a key, return an unexpected key, use a wrong owner, or
   violate a schema.
3. Verify the Kubernetes Manager starts only after all expected values have
   been persisted, and receives only its declared, applicable inputs.
4. Retry a running task and a task whose prior attempt failed;
   inspect the AAP job IDs and returned artifacts.
5. Delete the Subnet and inspect cleanup ordering and retained NetworkData.

**Expected results**

- Every successful manager task returns `artifacts.osac_result` with
  `resourceUID`, `observedGeneration`, and `data.network_data`.
  `data.network_data` is an empty list when no new outputs are expected. DHCP
  lease lookup and ExternalIP allocation use this same envelope.
- OSAC derives the expected Fabric model/owner keys internally. A successful
  Fabric task returns every expected key exactly once and no other key; each
  value matches its owner scope and JSON Schema. Existing values and values
  whose owner scope is not in the operation context are not expected again.
- OSAC validates the entire result before creating NetworkData. It creates
  runtime records through the fulfillment-service API; managers do not write
  Kubernetes objects or call that API.
- A missing or invalid expected value, duplicate key, or unexpected key
  prevents all NetworkData writes and dependent Kubernetes dispatch. Subnet
  create runs Fabric then Kubernetes; delete runs Kubernetes then Fabric,
  retaining values until cleanup succeeds.
- OSAC also checks that every applicable Kubernetes input exists after
  persistence. If one is absent, it reports the missing model and does not
  launch the Kubernetes job.
- OSAC polls the exact job ID it recorded. It checks the result against that
  job's resource context, manager stage, and task name. A result from another
  job, resource, generation, or manager target cannot complete the attempt.
  Retried manager operations reuse the same owner-scoped values and are
  idempotent by resource UID.

### TC-5: Verify the shared resource behavior across workload targets

**Requirements:** FR-1, FR-3; IC-3, IC-6

**Steps**

1. Exercise VirtualNetwork and Subnet behavior with the selected Fabric
   Manager, and Subnet overlay behavior when a Kubernetes Manager is selected.
2. Apply and delete workload attachments for ComputeInstance, Cluster, and
   BaremetalInstance targets supported by the profile.
3. Exercise SecurityGroup traffic matches, non-matches, and established reply
   traffic on each supported attachment target.

**Expected results**

- Subnets in one VirtualNetwork are separate L2 broadcast domains and have L3
  routing through their VirtualNetwork. Separate VirtualNetworks remain
  isolated.
- Workloads on the same Subnet share its broadcast domain.
- SecurityGroup rules apply to each network attachment, combine as a union,
  allow established replies, and deny unmatched new traffic.
- Policy is installed before traffic is enabled. A failed apply leaves the
  workload non-ready and traffic unavailable; deletion removes policy and
  detaches the interface before workload teardown.

### TC-6: Verify FabricDomain, ExternalIP, and DHCP result behavior

**Requirements:** FR-2, FR-4; IC-2, IC-4

**Steps**

1. Try creating a FabricDomain with Ethernet east-west disabled. Repeat with
   it enabled and the selected Fabric Manager declaring `EAST_WEST_ETHERNET`.
2. Allocate and release an ExternalIP, retrying allocation for the same
   ExternalIP UID. Submit a malformed address and an address outside the pool.
3. Query DHCP leases for Cluster and BaremetalInstance attachments, including
   missing and ambiguous matches.

**Expected results**

- A disabled FabricDomain request fails before AAP dispatch. When enabled,
  OSAC invokes `create_fabric_domain` or `delete_fabric_domain` through the
  selected generic Fabric Manager; its implementation preserves
  VirtualNetwork routing and Subnet broadcast domains.
- ExternalIP allocation returns the canonical IPv4 address in
  `osac_result.data.external_ip.address`. OSAC validates the result against
  its tracked job context and the address against the selected pool, then writes
  `ExternalIP.status.address`. Managers have no Kubernetes write credential.
  Retrying the same UID returns the same reservation; invalid results are
  rejected and capacity is released only after confirmed provider release.
- DHCP leases are returned in `osac_result.data.dhcp_leases`, with one
  unambiguous entry per requested attachment. Missing or ambiguous results
  fail the task.

## Provider Evidence

Before production selection, the Cloud Infrastructure Admin verifies every
base task set and target assigned to each selected role, plus each enabled
optional capability. Retain the test output, AAP job details with secret
values redacted, and observations of the resulting backend behavior. The
registration APIs validate declarations and data shapes; passing registration
validation alone is not evidence of behavioral conformance.
