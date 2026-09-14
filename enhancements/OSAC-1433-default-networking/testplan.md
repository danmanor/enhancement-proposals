# Testplan — OSAC-1433 Default Networking

## Overview

- **Feature:** OSAC-1433 — Default Networking and simplified resource creation
- **Source design:** [design.md](design.md)
- **Shared contract:** [Unified Networking test plan](../OSAC-1433-unified-networking/testplan.md)
- **Scope:** Tenant onboarding, default-resource lifecycle, readiness, default
  attachment resolution, automatic ExternalIP creation, cleanup, and supported
  combined-manager/K8s-only behavior.
- **Non-goals:** Per-tenant default configuration, additional automatic
  VirtualNetwork or Subnet creation, retroactive migration of existing
  resources, and UI support. API, REST, private API, and CLI are the tested
  surfaces.

## Test infrastructure and traceability

The test cases below are design-level requirements with implementation anchors
in the OSAC monorepo. New or extended tests should follow these existing
patterns rather than inventing a separate harness.

| Test level | Framework and environment | Existing implementation anchors |
|---|---|---|
| Unit | Ginkgo v2/Gomega; fulfillment-service in-memory DAO and fake manager state | `fulfillment-service/internal/servers/default_networking_provisioner_test.go`, `fulfillment-service/internal/servers/external_ip_pool_selector_test.go`, `fulfillment-service/internal/servers/cidr_validation_test.go`, `fulfillment-service/internal/servers/private_virtual_networks_server_test.go` |
| Integration | Ginkgo v2/Gomega; fulfillment-service integration harness with private gRPC clients, ephemeral database, and Kubernetes test clients | `fulfillment-service/it/it_default_networking_test.go`, `fulfillment-service/it/it_tenant_onboarding_test.go`, `fulfillment-service/it/it_validation_test.go`, `fulfillment-service/it/it_external_ip_test.go` |
| Operator integration | Ginkgo v2/Gomega with Kind and Kubernetes CR clients | `osac-operator/test/integration/networking_test.go` |
| E2E | pytest; connected single-hub deployment, `GRPCClient`, `K8sClient`, bounded polling helpers | `tests/e2e/vmaas/conftest.py`, `tests/e2e/core/grpc_client.py`, `tests/e2e/core/k8s_client.py`, `tests/e2e/core/helpers.py`, `tests/e2e/vmaas/sanity/test_virtual_network_lifecycle.py`, `tests/e2e/vmaas/sanity/test_subnet_lifecycle.py`, `tests/e2e/vmaas/sanity/test_security_group_lifecycle.py`, `tests/e2e/vmaas/regression/external_ip/test_external_ip_pool_capacity.py`, `tests/e2e/vmaas/regression/external_ip/test_external_ip_pool_lifecycle.py` |

### Shared test data and assertion contract

Unless a case overrides the value, use these concrete objects:

| Object | Concrete value |
|---|---|
| NetworkClass | `test-default-nc` |
| Tenant | `test-defnet-001` |
| VirtualNetwork CIDR | `10.200.0.0/16` |
| Default Subnet CIDR | `10.200.0.0/20` |
| Valid alternate Subnet CIDR | `10.200.1.0/24` |
| Invalid IPv6 CIDR | `2001:db8::/32` |
| Invalid host-bit CIDR | `10.200.0.7/20` |
| Invalid outside Subnet | `10.201.0.0/24` |
| MetalLB prefix | `32` |
| Default label | `osac.openshift.io/default: "true"` |
| Auto-created label | `osac.openshift.io/auto-created: "true"` |

Use private gRPC methods such as `NetworkClasses/Create`, `Tenants/Create`,
`Tenants/Get`, `VirtualNetworks/List`, `VirtualNetworks/Get`,
`Subnets/List`, `SecurityGroups/List`, and `ExternalIPs/Create`. For rejected
requests assert the gRPC status described by the shared design:

- `InvalidArgument` for malformed, missing, contradictory, or unsupported
  request values;
- `FailedPrecondition` for a valid request whose referenced resource is not in
  the required Ready/Allocated state, or for exhausted capacity/deletion
  dependencies; and
- `PermissionDenied` when a tenant attempts a provider-only operation and the
  resource is visible to the caller.

Assert the exact condition reason and message where the Default Networking
design defines one, including `ResourcesPending`, `AllResourcesReady`,
`NoDefaultNetworking`, `VirtualNetworkProvisioningFailed`,
`SubnetProvisioningFailed`, `SecurityGroupProvisioningFailed`,
`NATGatewayProvisioningFailed`, and:

`ExternalIPPool exhaustion: no available capacity in any READY pool for IPv4`

## Coverage summary

| Requirement | Test cases | Unit | Integration | E2E |
|---|---:|---:|---:|---:|
| R1 NetworkClass defaults | 2 | Yes | Yes | Rejection path |
| R2 Tenant onboarding | 3 | Yes | Yes | Yes |
| R3 Readiness/recovery | 2 | Yes | Yes | Yes |
| R4 Workload default resolution and immutability | 2 | Yes | Yes | Yes |
| R5 Automatic ExternalIP lifecycle | 2 | Yes | Yes | Yes |
| R6 Unsupported behavior | 1 | Yes | Yes | Rejection paths |
| **Total** | **12** | **All applicable** | **All applicable** | **All user-visible flows** |

## Test cases

### R1: NetworkClass defaults are valid and mandatory

#### TC-R1-01: Valid defaults are accepted

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration | critical | automated |

**Implementation references:**
`default_networking_provisioner_test.go` default-class builders,
`network_classes_server_test.go`, and `it_default_networking_test.go`.

##### Preconditions

- The test database has no active `test-default-nc`.
- The caller uses the provider/private client authorized to create a
  deployment NetworkClass.

##### Steps

1. Call `NetworkClasses/Create` with `metadata.name: test-default-nc`,
   `is_default: true`, `fabric_manager: cudn_net`, and:

   ```yaml
   spec:
     defaults:
       virtual_network_ipv4_cidr: 10.200.0.0/16
       subnet_ipv4_cidr: 10.200.0.0/20
     metallb_vip_prefix_length: 32
   ```

2. Read the response and then call `NetworkClasses/Get` using the returned ID.
3. Run the same request through the integration client used by
   `it_default_networking_test.go`.

##### Expected results

- The create call succeeds with gRPC status `OK`.
- `spec.defaults.virtual_network_ipv4_cidr` is exactly `10.200.0.0/16`.
- `spec.defaults.subnet_ipv4_cidr` is exactly `10.200.0.0/20`.
- `spec.metallb_vip_prefix_length` is accepted only when the CaaS/MetalLB
  capability is advertised.
- No separate enable/disable flag is accepted or required.

#### TC-R1-02: Invalid defaults are rejected before persistence

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E rejection | critical | automated |

**Implementation references:** `cidr_validation_test.go`,
`network_classes_server_test.go`, `it_validation_test.go`, and
`tests/e2e/core/grpc_client.py` request/error helpers.

##### Preconditions

- Use a fresh NetworkClass name for every invalid request.
- The caller has provider authorization so failures test validation rather
  than authorization, except for the final tenant-authorization case.

##### Steps

1. Submit each input mutation to `NetworkClasses/Create` using a fresh
   NetworkClass name.
2. For each rejected request, call `NetworkClasses/Get` and verify the
   rejected object is absent or unchanged.

##### Expected results

| Input mutation | Expected status and assertion |
|---|---|
| Omit `spec.defaults` | `InvalidArgument`; field violation identifies `spec.defaults`; no NetworkClass or tenant resources are persisted. |
| Set VN CIDR to `2001:db8::/32` | `InvalidArgument`; IPv6 is rejected. |
| Set VN CIDR to `10.200.0.7/20` | `InvalidArgument`; host bits are rejected. |
| Set Subnet CIDR to `10.201.0.0/24` | `InvalidArgument`; Subnet is outside the VN. |
| Set Subnet CIDR to `10.200.0.0/16` | `InvalidArgument`; Subnet cannot equal the VN range. |
| Advertise MetalLB capability but omit prefix length | `InvalidArgument`; `spec.metallb_vip_prefix_length` is required for the advertised capability. |
| Submit provider-only defaults as a tenant | `PermissionDenied` or the platform visibility error; no provider configuration is changed. |

### R2: Tenant onboarding creates exactly the supported graph

#### TC-R2-01: Combined-manager onboarding

| Test type | Priority | Automation |
|---|---|---|
| Integration, E2E | critical | automated |

**Implementation references:** `it_default_networking_test.go`,
`it_tenant_onboarding_test.go`, `default_networking_provisioner_test.go`,
`tests/e2e/vmaas/conftest.py`, and `tests/e2e/core/helpers.py`.

##### Preconditions

- `test-default-nc` exists with the valid values from the shared test-data
  table and both Fabric Manager and K8s Manager capabilities enabled.
- `test-defnet-001` does not exist.

##### Steps

1. Call `Tenants/Create` with `metadata.name: test-defnet-001`.
2. Poll `Tenants/Get` with `wait_for_tenant_condition` until the condition
   type is `DEFAULT_NETWORKING_READY`.
3. List `VirtualNetworks`, `Subnets`, and `SecurityGroups` with the tenant and
   default-label filter:

   ```text
   this.metadata.labels['osac.openshift.io/default'] == 'true'
   ```

4. If NAT capability is enabled, list `NATGateways` and inspect the
   referenced `ExternalIP` for the tenant.

##### Expected results

- `Tenants/Create` returns `OK` and one tenant ID.
- Exactly one default VirtualNetwork, one IPv4 Subnet, and one fallback
  SecurityGroup exist for `test-defnet-001`.
- Each default resource has tenant ownership and the default label.
- The deployment baseline is separate from the fallback SecurityGroup and its
  hard-coded `permit` action remains effective even if the fallback group has
  no rules.
- NATGateway exists only when NAT capability is enabled, carries the default
  label, and references a Ready/Allocated unconsumed ExternalIP.
- `DefaultNetworkingReady` transitions from `ResourcesPending` to
  `AllResourcesReady` only after every capability-required resource is Ready.
- The `DefaultNetworkingCreated` event is emitted when supported default
  resource creation starts.

#### TC-R2-02: K8s-only onboarding excludes NATGateway

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E | critical | automated |

**Implementation references:** `default_networking_provisioner_test.go`,
`it_default_networking_test.go`, `it_tenant_onboarding_test.go`, and
`tests/e2e/conftest.py` K8s-only NetworkClass fixture.

##### Preconditions

- Configure `test-k8s-only-nc` with `k8s_manager: cudn_evpn`, no Fabric
  Manager, and the same valid IPv4 defaults.
- No NATGateway capability is advertised.
- `test-k8s-only-001` does not exist.

##### Steps

1. Call `NetworkClasses/Create` for `test-k8s-only-nc`.
2. Call `Tenants/Create` for `test-k8s-only-001`.
3. Poll `Tenants/Get` and list the tenant's default resources.
4. Inspect the fake manager calls in the unit test and the Kubernetes CRs in
   the integration/E2E environment.
5. Call `NATGateways/Create` as the tenant.

##### Expected results

- VN, Subnet, and fallback SecurityGroup reach Ready.
- No `NATGateways/Create` call is dispatched to any manager.
- NATGateway is excluded from the readiness set; it is not left Pending or
  Failed.
- Tenant onboarding reaches `DefaultNetworkingReady=True` with reason
  `AllResourcesReady`.
- The tenant NATGateway request is rejected with `FailedPrecondition` because
  the resolved NetworkClass does not advertise NATGateway capability, and no
  NATGateway is persisted.

#### TC-R2-03: Onboarding is idempotent and does not create extra defaults

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration | high | automated |

**Implementation references:** `default_networking_provisioner_test.go`,
`it_default_networking_test.go`, and `it_tenant_lifecycle_test.go` concurrency
and reconciliation patterns.

##### Preconditions

- `test-idempotent-001` has no default resources.
- `test-tenant-delete-001` has no default resources.
- `test-default-nc` has the valid shared test data.

##### Steps

1. Submit two concurrent `Tenants/Create`/onboarding requests for
   `test-idempotent-001`.
2. Interrupt reconciliation after VN creation, after Subnet creation, and
   after SecurityGroup creation, then invoke the tenant signal/reconciliation
   path.
3. Repeat onboarding after the graph is complete.
4. Onboard `test-tenant-delete-001`, wait for its default graph to be Ready,
   delete the Tenant, and list its former default resources.
5. Create a deliberately mismatched default resource with the same tenant and
   default label, then rerun onboarding.

##### Expected results

- The matching graph is adopted idempotently.
- Exactly one default VN, Subnet, and SecurityGroup exist after every retry.
- No duplicate jobs, ExternalIP capacity reservations, or default resources
  are created.
- Deleting `test-tenant-delete-001` removes its default resources through
  tenant owner-reference cleanup.
- The mismatched graph returns a provider configuration error and is not
  silently adopted or overwritten.

### R3: Default readiness and failure recovery

#### TC-R3-01: Readiness waits for every supported default

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E | critical | automated |

**Implementation references:** `it_default_networking_test.go`,
`default_networking_provisioner_test.go`, `tests/e2e/core/helpers.py`
(`wait_for_tenant_condition`), and `tests/e2e/core/k8s_client.py`.

##### Preconditions

- Use a controllable fake manager for unit/integration tests.
- Create `test-readiness-001` under a NetworkClass with the valid defaults.

##### Steps

1. Set each manager/resource state in the table below.
2. Call `Tenants/Get` and inspect the Tenant condition and Kubernetes events.
3. Attempt workload creation without an explicit attachment while the default
   graph is not Ready.

##### Expected results

| Manager/resource state | Required assertion |
|---|---|
| VN Pending | `DefaultNetworkingReady=False`, reason `ResourcesPending`; workload create is rejected with `FailedPrecondition`. |
| VN Failed | `DefaultNetworkingReady=False`, reason `VirtualNetworkProvisioningFailed`; the event includes `DefaultNetworkingFailed`. |
| Subnet Failed | `DefaultNetworkingReady=False`, reason `SubnetProvisioningFailed`; no workload receives a default Subnet. |
| SecurityGroup Failed | `DefaultNetworkingReady=False`, reason `SecurityGroupProvisioningFailed`; no workload receives a default SecurityGroup. |
| Supported NATGateway Failed | `DefaultNetworkingReady=False`, reason `NATGatewayProvisioningFailed`. |
| Feedback for another tenant/VN | Ignore the feedback; the target tenant condition and resource state do not change. |
| All required resources Ready | `DefaultNetworkingReady=True`, reason `AllResourcesReady`; all returned references are Ready, and the `DefaultNetworkingReady` event is present. |

#### TC-R3-02: Failure and documented recovery path

| Test type | Priority | Automation |
|---|---|---|
| Integration, E2E | high | automated |

**Implementation references:** `it_default_networking_test.go`,
`it_tenant_lifecycle_test.go`, and `tests/e2e/core/helpers.py` bounded polling
helpers.

##### Preconditions

- `test-recovery-001` has a valid NetworkClass and a controllable manager.
- Configure the manager to fail one operation at a time.

##### Steps

1. Fail VN provisioning and read `Tenants/Get` plus the Kubernetes Tenant CR.
2. Restore the manager and signal reconciliation.
3. Repeat steps 1–2 for Subnet, SecurityGroup, and supported NATGateway.
4. For a terminal graph error, delete the tenant as specified by the design,
   recreate it, and poll until recovery completes.

##### Expected results

- Each failure emits `DefaultNetworkingFailed` and the exact reason listed in
  TC-R3-01; the message includes the failed resource name.
- Transient failure retries without setting `DefaultNetworkingReady=True`.
- After recovery, exactly one clean default graph exists and the condition is
  `DefaultNetworkingReady=True/AllResourcesReady`.
- Existing immutable workload attachments are not rewritten while readiness
  is degraded.

### R4: Workload default resolution

#### TC-R4-01: Omitted, empty, partial, and complete inputs

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E | critical | automated |

**Implementation references:** `default_networking_provisioner_test.go`,
`fulfillment-service/internal/servers/private_virtual_networks_server_test.go`,
`tests/e2e/vmaas/conftest.py`, `tests/e2e/core/grpc_client.py`, and the
service-specific VM/CaaS/BMaaS networking test plans.

##### Preconditions

- Tenant `test-defaulting-001` has Ready default Subnet
  `default-ipv4` (`10.200.0.0/20`) and default SecurityGroup `default-sg`.
- Create an explicit Ready alternate Subnet `explicit-subnet`
  (`10.200.1.0/24`) in the same VN and an explicit Ready SecurityGroup
  `explicit-sg`.
- Use typed local references: `{name: "explicit-subnet"}` and
  `{name: "explicit-sg"}`.

##### Steps

1. Submit each request row below as a separate VM, Cluster, or BM create
   request.
2. Read the created parent with the corresponding `Get` method.
3. Inspect the resolved network fields and assert the
   `NetworkAttachmentsPopulated` event when defaulting occurred.

##### Expected results

| Request | Expected result and assertion |
|---|---|
| VM `compute_network_attachments` omitted | One resolved attachment: Subnet `default-ipv4`, SecurityGroup `default-sg`, `primary=true`. |
| VM attachment list empty | Same result as omitted; no second attachment is created. |
| Cluster `network_attachment` omitted | One cluster attachment containing both defaults. |
| Cluster attachment message empty | Same result as omitted; no arbitrary Subnet is selected. |
| BM `network_attachments` omitted or empty | Exactly one resolved attachment with default Subnet, default SecurityGroup, and the first eligible fabric interface. |
| Only Subnet supplied as `{name: "explicit-subnet"}` | Preserve `explicit-subnet`; fill only `default-sg`. |
| Only SecurityGroup supplied as `[{name: "explicit-sg"}]` | Preserve `explicit-sg`; fill only `default-ipv4`. |
| Complete Subnet and SecurityGroup input supplied | Preserve both references and do not replace them with defaults. |
| Non-Ready explicit Subnet or SecurityGroup | `FailedPrecondition`; no fallback substitution occurs. |
| VM/BM list has two attachments | `InvalidArgument`; no workload is persisted or dispatched. |
| VM attachment has `primary=false` | `InvalidArgument`; the supported single attachment is always primary. |

#### TC-R4-02: Default resources and network-owned fields are immutable

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E rejection | critical | automated |

**Implementation references:** `private_virtual_networks_server_test.go`,
`private_subnets_server_test.go`, `security_groups_server_test.go`,
`it_validation_test.go`, and `tests/e2e/vmaas/regression/test_name_immutability.py`.

##### Preconditions

- Create Ready default VN `default`, Subnet `default-ipv4`, and fallback
  SecurityGroup `default-sg` for `test-defaulting-001`.
- Create a VM referencing `default-ipv4` so deletion has a dependency.

##### Steps

1. Call `VirtualNetworks/Update`, `Subnets/Update`, and
   `SecurityGroups/Update` with a network-owned field mask.
2. Repeat with `PATCH` and full replacement payloads.
3. Call `Subnets/Delete` while the VM exists.
4. Delete the VM, then call `Subnets/Delete` and recreate the desired Subnet.

##### Expected results

- Every network-owned update, patch, and replacement returns
  `InvalidArgument` or `FailedPrecondition` according to the shared API
  operation guard; the stored spec is unchanged.
- Subnet deletion while referenced returns `FailedPrecondition` and leaves the
  Subnet present.
- After dependencies are removed, delete succeeds and a replacement can be
  created with a new immutable specification.

### R5: Automatic ExternalIP lifecycle

#### TC-R5-01: Successful automatic external access

| Test type | Priority | Automation |
|---|---|---|
| Integration, E2E | critical | automated |

**Implementation references:** `it_default_networking_test.go`,
`it_external_ip_test.go`, `external_ip_pool_selector_test.go`,
`tests/e2e/vmaas/regression/external_ip/test_external_ip_pool_lifecycle.py`,
`tests/e2e/core/grpc_client.py`, and `tests/e2e/core/helpers.py`.

##### Preconditions

- A Ready IPv4 ExternalIPPool exists with CIDR `198.51.100.0/29`, at least
  four available addresses, and no overlapping pool.
- The VM, Cluster, and BM target resources each have one Ready network
  attachment and a discoverable workload IP/VIP.
- A separate explicitly managed ExternalIP and ExternalIPAttachment exist for
  one Ready target and do not carry the auto-created label.

##### Steps

1. Set `auto_external_ip_attachment=true` in the VM and BM create requests.
2. Set it for the Cluster API and Ingress endpoints in the Cluster create
   request.
3. Observe `ExternalIPs/Create` and `ExternalIPAttachments/Create` records
   through the private API.
4. Complete target IP/VIP discovery and wait for the attachment status.
5. Delete each parent resource and observe the cleanup order.
6. Delete the target that owns the explicitly managed ExternalIPAttachment and
   inspect that attachment and ExternalIP.

##### Expected results

- VM and BM receive one ExternalIP; Cluster receives two, one for API and one
  for Ingress.
- Create persists Pending records only after synchronous capacity validation;
  allocation, discovery, DNAT, and Ready transitions are asynchronous.
- `ExternalIP` transitions `Pending -> Allocated` and the attachment
  transitions `Pending -> Ready`.
- DNAT targets the discovered workload IP/VIP.
- Auto-created attachments are deleted before their ExternalIPs and carry
  `osac.openshift.io/auto-created: "true"`; auto-created ExternalIPs also
  carry `osac.openshift.io/auto-created-for: <resource-id>`.
- The `AutoExternalIPCreated` event is present on each workload that received
  automatic external access.
- The explicitly managed ExternalIP and ExternalIPAttachment remain after
  their target parent is deleted because they do not carry the auto-created
  label.

#### TC-R5-02: Capacity and cleanup-failure behavior

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E rejection | critical | automated |

**Implementation references:** `external_ip_pool_selector_test.go`,
`external_ip_pools_server_test.go`, `it_external_ip_test.go`,
`tests/e2e/vmaas/regression/external_ip/test_external_ip_pool_capacity.py`,
and `tests/e2e/core/helpers.py` `assert_grpc_rejected`/polling helpers.

##### Preconditions

- Create a Ready IPv4 pool `small-pool` with CIDR `198.51.100.0/30`; the
  usable capacity is two addresses.
- Configure every pool considered by automatic selection with `available: 0`
  for the exhaustion subcase.
- Create two additional Ready IPv4 pools with non-overlapping CIDRs and
  controlled capacities so that one has the greatest capacity and two can be
  configured with equal capacity for the deterministic tie-break check.
- Provide a controllable manager/finalizer fixture that can inject transient
  and permanent cleanup failures.

##### Steps

1. Call `ExternalIPs/Create` for `small-pool` when `available: 0`, then call
   `ComputeInstances/Create` with `auto_external_ip_attachment=true` for an
   otherwise valid VM while every candidate pool is exhausted.
2. Restore capacity in the additional pools and submit equivalent automatic
   ExternalIP requests while the pools have different available capacities,
   then reset capacity and repeat while two candidate pools have equal
   capacity.
3. Complete one automatic allocation successfully, delete its parent, and
   observe attachment deletion, ExternalIP release, and parent finalizer
   completion.
4. Create another automatic allocation, inject a transient cleanup failure,
   delete its parent, and observe the finalizer retry before allowing parent
   deletion.
5. Create another automatic allocation, inject a permanent cleanup failure,
   delete its parent, and wait until the controller exhausts its retry policy.
6. Inspect the orphaned resources, then manually delete the orphaned
   ExternalIPAttachment before the orphaned ExternalIP.

##### Expected results

- Pool selection chooses the Ready pool with greatest available capacity.
  Equal-capacity selection is deterministic, but the tie-break remains
  implementation-defined as specified by the design; the test asserts that
  repeated equivalent requests select the same pool without imposing a
  pool-ID ordering that the contract does not define.
- Exhaustion returns `FailedPrecondition` with the exact message:
  `ExternalIPPool exhaustion: no available capacity in any READY pool for IPv4`.
- Both explicit and automatic exhaustion failures happen before persistence;
  no parent, ExternalIP, attachment, or capacity reservation remains.
- A successful automatic request persists the ExternalIP and attachment as
  `Pending`; manager allocation and attachment/DNAT activation then proceed
  asynchronously.
- After successful parent deletion, cleanup order is
  `ExternalIPAttachment -> ExternalIP -> parent`, and the released capacity
  is available again.
- A transient cleanup failure retries through the parent finalizer.
- After permanent cleanup failure, the finalizer is removed, the parent is
  deleted, and orphaned ExternalIP/ExternalIPAttachment resources remain with
  `osac.openshift.io/auto-created: "true"` and no parent reference; manual
  cleanup is then required. The orphaned ExternalIP retains its
  `osac.openshift.io/auto-created-for: <resource-id>` label.
- Manual cleanup must delete the attachment before the ExternalIP; after both
  are removed, the previously reserved capacity is released.

### R6: Unsupported Default Networking behavior

#### TC-R6-01: Unsupported scope is not silently enabled

| Test type | Priority | Automation |
|---|---|---|
| Unit, integration, E2E rejection | high | automated |

**Implementation references:** `it_default_networking_test.go`,
`it_validation_test.go`, `private_virtual_networks_server_test.go`,
`tests/e2e/vmaas/sanity/test_virtual_network_lifecycle.py`, and
`tests/e2e/vmaas/regression/test_name_immutability.py`.

##### Preconditions

- Use `test-default-nc` and tenant `test-unsupported-001` with the valid
  shared test data.
- The default graph is Ready before exercising workload and deletion guards.

##### Steps

1. Submit each unsupported request in the table below through its corresponding
   API or CLI operation.
2. For every rejection, call the relevant `Get`/`List` method and inspect
   Kubernetes events for the affected Tenant or workload.
3. Verify that no manager job, capacity reservation, or partial resource graph
   was created.

##### Expected results

| Unsupported request | Expected result |
|---|---|
| Tenant supplies custom default CIDRs or provider defaults | `PermissionDenied`/`InvalidArgument`; provider defaults remain unchanged. |
| Tenant requests an automatic second VN or Subnet | `InvalidArgument`; no second resource or manager job is created. |
| Existing tenant is retroactively assigned defaults | No mutation; request is rejected or excluded by the API contract. |
| UI-only simplified creation through the API/CLI | No hidden UI behavior is exposed; normal API validation applies. |
| Tenant-created empty SecurityGroup used as fallback | `InvalidArgument`; only the system-created fallback may be empty because the deployment baseline is hard-coded `permit`. |
| Workload omits networking while defaults are missing | `FailedPrecondition` with `No default networking resources available. Please contact your administrator.`; no workload or attachment is persisted. |
| Workload references Pending or Failed defaults | `FailedPrecondition`; no workload or attachment is persisted and no fallback substitution occurs. |
| Delete a default resource with active dependents | `FailedPrecondition`; parent and dependent resources remain. |
| Network-owned update, patch, or replacement | Rejected with the shared CRUD guard; stored network fields are unchanged. |
| Tenant supplies an arbitrary ExternalIP address instead of a Ready pool | `InvalidArgument`; only pool allocation is accepted. |

##### Final assertions

1. For every rejected request, call the corresponding `Get`/`List` method.
2. Assert no hidden fallback, partial resource graph, manager job, capacity
   reservation, or workload dispatch was created.
3. Assert the rejection includes the expected gRPC status and field path when
   the API contract defines one.

## Graduation gate

- All 12 test cases have explicit implementation references.
- Every test case has concrete preconditions, numbered steps or a complete
  input/case table, and observable expected results.
- Every onboarding and defaulting rule maps to a unit or integration test.
- Combined-manager and K8s-only supported workflows have E2E coverage.
- All three workload services have omitted/empty/partial/complete coverage.
- Failure, retry, idempotency, capacity exhaustion, cleanup, and immutability
  tests assert exact condition reasons, gRPC statuses, or resource fields.
