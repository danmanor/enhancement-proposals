# Testplan — OSAC-4291

## Overview

- **Feature:** OSAC-4291 — CUDN EVPN K8s Manager Phase 1 Networking: Single-Cluster VM-to-Fabric Bridging
- **Total test cases:** 14
- **Requirements covered:** 9 of 9 (R1-R9)
- **Interface changes covered:** IC-1, IC-2, IC-3, IC-5, IC-6; no case has IC-4
- **Additional operational tests:** 2 deletion lifecycle tests
- **Lab suite:** `tests/e2e/networking/vmaas/test_vmaas_networking.py` is one ordered module; the job stops it at the first failed test (`pytest -x`), so later tests of the module do not run after a failure and fixture finalizers clean up. The job then runs the ExternalIP and NATGateway billing files as a second pytest run. It keeps `test_00` to `test_03` (connectivity and isolation), `test_04` to `test_08` (cluster DNS, SecurityGroup enforcement, cross-tenant isolation), `test_09` to `test_13` (NAT egress and ExternalIP ingress) and `test_14` (one teardown test). Every kept test decides the job; none is non-gating. The ExternalIP pool check is a fixture check, so it fails the first test that needs the pool. Removed E2E requirements, not covered by an equal E2E check: node service isolation from a tenant VM, the SecurityGroup ICMP capability gate (see TC-ICMP-01) and second-tenant automatic ExternalIP ingress and garbage collection. The `external_ip/` and `references/` directories were removed. The second run covers `test_external_ip_billing.py`, `test_external_ip_reattach_billing.py` and `test_nat_gateway_billing.py`.
- **Outside R1-R9:** TC-ICMP-01 (SecurityGroup ICMP capability gate) traces to contract FR-13 and IC-1, not to a requirement of this PRD; it is counted in the total

## Test Cases

### R1: K8s manager registration for EVPN fabric bridging (IPv4 only)

#### TC-R1-01: Register cudn_evpn k8s manager via ConfigMap

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-1 | high | automated |

##### Preconditions

- **Tier / owner:** Unit / [DEV].
- osac-installer deployed to cluster
- No existing ConfigMap `osac-network-k8s-manager-cudn-evpn` in osac namespace

##### Steps

1. Apply osac-installer Helm chart with cudn_evpn manager enabled
2. Verify ConfigMap `osac-network-k8s-manager-cudn-evpn` exists in osac namespace
3. Verify ConfigMap data.name = "cudn_evpn", data.ansible_role = "osac.templates.cudn_evpn", and data.network_type = "evpn"
4. Verify the ConfigMap has no `capabilities` key (cudn_evpn does not declare `security_group_icmp`)
5. Verify the rendered Netris registration has `capabilities: eastWestEthernet,security_group_icmp`

##### Expected Results

- ConfigMap created with label `osac.openshift.io/network-k8s-manager: "true"`
- `cudn_evpn` declares no capability, so a NetworkClass with it rejects ICMP rules (contract TC-CAP-03)
- NetworkClass controller loads cudn_evpn as available k8s manager

### R2: Fabric-to-k8s manager data dependency

#### TC-R2-01: Sequential provisioning fabric then k8s manager

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-3 | critical | automated |

##### Preconditions

- **Tier / owner:** Unit / [DEV].
- NetworkClass with fabric_manager="netris", k8s_manager="cudn_evpn"
- VirtualNetwork created with this NetworkClass
- Mocked Netris fabric returning `l3_vni` from `create_virtual_network` and `network_segment: {type: evpn, l2_vni}` from `create_subnet`

##### Steps

1. Observe the VirtualNetwork's Fabric `create_virtual_network` returns `l3_vni` and the VirtualNetwork CR records it in `status.l3Vni`
2. Create Subnet via fulfillment-service API
3. Observe Subnet controller creates fabric AAP Job first, with the `virtual_network` snapshot carrying `status.l3Vni`
4. Fabric job completes with `osac_result.network_segment`
5. Observe controller does not create k8s job until the Fabric `create_subnet` entry is `Succeeded` and `status.networkSegment` is recorded
6. Observe controller creates k8s AAP Job
7. Verify the k8s job's `osac_job_vars` contains `network_segment: {type: evpn, l2_vni}` and `virtual_network.status.l3Vni` (no route targets: both sides use `0:<VNI>`)

##### Expected Results

- Fabric job completes before k8s job starts (not concurrent)
- K8s job receives the L2 VNI from `Subnet.status.networkSegment` and the L3 VNI from `VirtualNetwork.status.l3Vni`
- `status.taskInput` stores the `virtual_network` snapshot with `status.l3Vni` and no `network_segment`

#### TC-R2-02: Invalid VNI result from the fabric job

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-3 | high | automated |

##### Preconditions

- **Tier / owner:** Unit / [DEV].
- Subnet provisioning in progress, fabric job completed
- Fabric job's `osac_result` is invalid: `network_segment` missing, `l2_vni` a string or out of 1..16777215, an extra `l3_vni`, or an `l2_vni` that a live VirtualNetwork or Subnet holds

##### Steps

1. Controller validates the fabric job's `osac_result`
2. Validation fails
3. Observe Subnet `Ready=False`, reason `Failed`, with a message naming the broken rule ("VNI already in use" for a held VNI)
4. Repeat with an `l2_vni` held only by a Subnet that has a deletionTimestamp: observe the Fabric `create_subnet` entry goes to `Retrying`, and the rerun is accepted once that Subnet is gone

##### Expected Results

- Subnet provisioning stops for an invalid result (k8s job never created); `status.networkSegment` stays unset
- The message names no VNI and no other resource's ID
- A VNI held only by deleting resources is retried, not rejected

### R3: Automatic overlay network provisioning on hosting clusters

#### TC-R3-01: CUDN provisioned with EVPN transport

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-5 | critical | automated |

##### Preconditions

- **Tier / owner:** Unit / [DEV].
- Subnet created with NetworkClass k8s_manager="cudn_evpn"
- K8s manager job running

##### Steps

1. Observe the VirtualNetwork's namespace `osac-vn-<VirtualNetwork API ID>` (created by Kubernetes `create_virtual_network`) has the label `k8s.ovn.org/primary-user-defined-network`
2. Observe playbook creates ClusterUserDefinedNetwork CR with:
   - metadata.name = `osac-vn-<VirtualNetwork API ID>`
   - metadata.labels `evpn: "true"`
   - spec.network.topology = "Layer2"
   - spec.network.transport = "EVPN"
   - spec.network.layer2.mtu = 1500, defaultGatewayIPs = [ipam.gateway_ipv4]
   - spec.network.evpn.vtep = "tenant-vtep"
   - spec.network.evpn.macVRF = {vni: network_segment.l2_vni, routeTarget: "0:<l2_vni>"}
   - spec.network.evpn.ipVRF = {vni: virtual_network.status.l3Vni, routeTarget: "0:<l3_vni>"}
3. Wait until the CUDN reports `TransportAccepted`, `NetworkCreated` and `NetworkAllocationSucceeded` True (OVN-Kubernetes 4.22 sets no `Ready` condition)

##### Expected Results

- CUDN CR exists with correct VNI values
- CUDN reports `TransportAccepted`, `NetworkCreated` and `NetworkAllocationSucceeded` True within 60 seconds
- OVN-Kubernetes provisions VXLAN interfaces on worker nodes

### R4: VM-to-fabric connectivity (same Subnet; between Subnets in a later phase)

#### TC-R4-01: L2 same-subnet VM to bare-metal connectivity

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-5, IC-6 | critical | manual |

##### Preconditions

- **Tier / owner:** E2E / [QE].
- CUDN provisioned with subnet 200.200.1.0/24
- VirtualMachine deployed in CUDN namespace, IP 200.200.1.3
- Bare-metal node provisioned on Netris fabric in same subnet, IP 200.200.1.10
- FRR advertising EVPN routes to fabric

##### Steps

1. Verify VM running: `oc get vmi -n <namespace>`
2. Open the VM serial console (`osac` console command or `virtctl console <vm-name>`)
3. Ping bare-metal node: `ping 200.200.1.10`
4. Verify FRR shows Type-2 route for VM MAC: `vtysh -c "show bgp l2vpn evpn" | grep <vm-mac>`
5. Verify Netris fabric learned VM MAC via EVPN

##### Expected Results

- Ping succeeds (RTT <10ms)
- FRR advertises Type-2 EVPN route with VM MAC and IP
- Netris leaf switch has VM MAC in EVPN table pointing to OCP VTEP

#### TC-R4-02: L3 cross-subnet VM to bare-metal connectivity

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-5, IC-6 | critical | manual, Blocked (later phase) |

##### Preconditions

- **Tier / owner:** E2E / [QE].
- **Status:** Blocked. VM routing between Subnets waits for a later phase. In phase 1, VMs run only on the Subnet that owns the hub network, and the case also depends on the Netris leaves importing `0:<VNI>`. Not run until then.
- CUDN provisioned with subnet 200.200.1.0/24
- VirtualMachine deployed in CUDN namespace, IP 200.200.1.3
- Bare-metal node provisioned on Netris fabric in different subnet 200.200.2.0/24, IP 200.200.2.10
- Both subnets under same Netris VPC (shared ipVRF)

##### Steps

1. Console into VM
2. Ping bare-metal node in different subnet: `ping 200.200.2.10`
3. Verify FRR shows Type-5 route for CUDN prefix: `vtysh -c "show bgp l2vpn evpn" | grep Type-5 | grep 200.200.1.0`
4. Verify Netris VPC routing table includes both subnets

##### Expected Results

- Ping succeeds (routed via ipVRF)
- FRR advertises Type-5 EVPN route for 200.200.1.0/24 prefix
- Traffic encapsulated with L3 VNI (ipVRF), not L2 VNI (macVRF)

### R5: One VM Subnet per VirtualNetwork for this k8s manager

#### TC-R5-01: Second Subnet is accepted, a VM on it fails with the limit error

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-2 | critical | automated |

##### Preconditions

- **Tier / owner:** Component integration / [DEV]. `osac-aap/tests/integration/targets/cudn_evpn`.
- NetworkClass with k8s_manager="cudn_evpn"
- VirtualNetwork created with this NetworkClass
- One Subnet already exists under this VirtualNetwork and owns the hub network

##### Steps

1. Run `create_subnet` for a second Subnet under the same VirtualNetwork
2. Run `apply_network_attachment` for a VM on the second Subnet

##### Expected Results

- `create_subnet` succeeds and creates no second hub network; OSAC does not reject the Subnet (contract §4.7)
- The attachment fails as final, with a message that starts "cudn_evpn implementation limit" and names the Subnet that does not own the hub network
- The first Subnet's hub network is unchanged

### R6: Non-conflicting IP address assignment

#### TC-R6-01: OVN answers the VM; Netris DHCP does not

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-5 | high | manual |

##### Preconditions

- **Tier / owner:** E2E / [QE].
- CUDN provisioned with subnet 200.200.1.0/24
- Netris V-Net created by the Netris manager; Netris DHCP is on and covers `physical_ipv4_cidrs` only

##### Steps

1. Verify CUDN `spec.network.layer2.reservedSubnets` covers exactly the Subnet outside `ipam.overlay_ipv4_cidrs` and the two infrastructure addresses
2. Verify CUDN `spec.network.layer2.infrastructureSubnets` is `[<gateway_ipv4>/32, <network + 2>/32]`
3. Deploy VirtualMachine in CUDN namespace
4. Verify VM receives IP address via DHCP
5. Check VM received IP from OVN DHCP (inside VM: check DHCP server IP in lease file)
6. Verify the Netris DHCP scope is only `physical_ipv4_cidrs`, not the overlay range
7. Verify VM IP is not in fabric reserved range (reservedSubnets)
8. Check Netris DHCP does not answer the VM MAC address
9. Verify VM IP is in subnet CIDR (200.200.1.0/24)

##### Expected Results

- CUDN `reservedSubnets` and `infrastructureSubnets` follow the `ipam` layout (contract §4.8)
- VM IP assigned by OVN-Kubernetes DHCP (not Netris DHCP), in `ipam.overlay_ipv4_cidrs`
- VM DHCP lease gives `gateway_ipv4` as router, served by OVN-Kubernetes
- Only OVN-Kubernetes answers the VM; the VM's DHCP request never reaches Netris DHCP
- **VM IP does not collide with the gateway, the management port (network + 2), or the fabric DHCP and LB VIP ranges**
- OVN IPAM respects the reserved range (correctness requirement, not optional)
- No second DHCP server exists on the Subnet for VMs

### R7: Installation prerequisites documentation

#### TC-R7-01: Installation guide covers all manual prerequisites

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| — | high | manual |

##### Preconditions

- Access to osac-installer documentation

##### Steps

1. Read installation guide for cudn_evpn k8s manager
2. Verify guide documents:
   - VTEP CR creation
   - FRRConfiguration underlay BGP peering
   - RouteAdvertisements CR
   - BGP underlay connectivity (worker ↔ fabric switch)
   - Gateway MAC addresses: Netris serves `gateway_ipv4` with its own MAC and OVN-Kubernetes serves it to VMs; two MACs are accepted (contract §4.9)
3. Verify guide includes validation commands to check prerequisites complete

##### Expected Results

- All manual prerequisites documented with examples
- Validation commands provided (check VTEP exists, FRR BGP session up, etc.)
- Guide states that two gateway MACs are accepted and a shared MAC is optional

### R8: Diagnostic tooling documentation

#### TC-R8-01: Diagnostic commands documented for troubleshooting

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| — | medium | manual |

##### Preconditions

- Access to osac documentation

##### Steps

1. Read troubleshooting guide for cudn_evpn
2. Verify guide documents:
   - FRR VNI status check: `vtysh -c "show evpn vni"`
   - BGP EVPN routes check: `vtysh -c "show bgp l2vpn evpn"`
   - CUDN status check: `oc get clusteruserdefinednetwork`
   - Netris VNet query via API
   - Packet capture for VXLAN traffic

##### Expected Results

- Diagnostic commands cover VNI mismatch detection
- Commands cover gateway MAC comparison
- Commands cover BGP session verification
- Known failure modes documented with symptoms and fixes

### R9: Gateway MAC addresses

#### TC-R9-01: Two gateway MACs documented

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| — | high | manual |

##### Preconditions

- Access to installation guide

##### Steps

1. Read installation prerequisites
2. Verify guide documents the gateway: Netris serves `gateway_ipv4` with its own MAC, OVN-Kubernetes serves it to VMs
3. Verify guide explains the effect of two gateway MACs on routed traffic

##### Expected Results

- Two gateway MACs documented as accepted (contract §4.9); a shared MAC is optional, not a prerequisite
- Guide explains the effect of two gateway MACs on routed traffic

### SecurityGroup ICMP: capability gate with cudn_evpn

#### TC-ICMP-01: ICMP rule rejected at the API, protocol all passes ICMP

Traces to contract FR-13 and [contract TC-CAP-03](/enhancements/OSAC-1433-network-manager-integration-contract-networking/testplan.md#tc-cap-03-reject-icmp-rules-at-the-securitygroup-api); it adds no PRD requirement.

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-1 | medium | removed from E2E (lower tiers only) |

##### Preconditions

- **Tier / owner:** no E2E case remains, so no [QE] owner. The evidence is [DEV]-owned lower-tier cases: the API gate (contract TC-CAP-03) in `fulfillment-service/it/it_networking_design_test.go` (TC-SG-ICMP-01, deployed Fulfillment with PostgreSQL and Keycloak, runs in CI) and `fulfillment-service/internal/servers/security_groups_server_test.go` (real PostgreSQL, mocked tenancy); the derivation (contract TC-CAP-01, TC-CAP-02) in `osac-operator/test/integration/networking_networkclass_test.go` (netris plus `cudn_evpn` yields false). Not covered at any tier: a capable NetworkClass reaching READY through AAP, which `cudn_evpn` cannot exercise.
- **Suite and command:** none in CI. The E2E check `test_05_icmp_security_group_rule_follows_capability` was removed from `tests/e2e/networking/vmaas/test_vmaas_networking.py`, so this E2E requirement is removed, not covered. The ICMP rejection stays covered by [DEV] cases at lower tiers (contract TC-CAP-01 to TC-CAP-03). The deployed lab (netris + `cudn_evpn`, `make e2e INFRA=netris DEPLOY_SUITE=vmaas`) does not exercise it.
- **Status:** removed from the E2E job; the steps below are not run in CI.
- Deployed lab with Netris and `cudn_evpn`, so the NetworkClass has `status.security_group_icmp` false
- Two VMs on one Subnet

##### Steps

1. Create a SecurityGroup with an ICMP ingress rule through the public API
2. Create a SecurityGroup with an `all` ingress rule from the Subnet CIDR and attach it to both VMs
3. Ping one VM from the other

##### Expected Results

- Step 1 fails with `INVALID_ARGUMENT`, the message says to use protocol `all`, and no SecurityGroup is stored
- Step 3 succeeds, because `all` includes ICMP
- No `cudn_evpn implementation limit` error appears in any job

### Subnet Deletion: Ordered cleanup prevents stale VRFs

#### TC-DELETE-01: Ordered delete: VM, Subnet, VirtualNetwork

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| IC-5 | high | automated |

##### Preconditions

- **Tier / owner:** E2E / [QE].
- One VirtualNetwork with one Subnet that owns the CUDN
- VirtualMachine running in the namespace `osac-vn-<VirtualNetwork API ID>`

##### Steps

1. Delete the Subnet while the VM exists
2. Delete the VM and wait until its record is gone
3. Delete the Subnet
4. Verify the CUDN is gone before the Fabric `delete_subnet` runs, and the Netris V-Net is deleted
5. Verify the namespace `osac-vn-<VirtualNetwork API ID>` still exists
6. Delete the VirtualNetwork
7. Verify the namespace and its baseline deny are deleted
8. Check for stale VRF on worker nodes (should not exist)

##### Expected Results

- Step 1 fails with `FAILED_PRECONDITION` ([Unified — Deletion dependency guards](/enhancements/OSAC-1433-unified-networking/design.md#deletion-dependency-guards))
- Subnet delete: the owner Subnet's Kubernetes `delete_subnet` removes the CUDN and waits until it is gone, then the Fabric delete runs
- The namespace stays until Kubernetes `delete_virtual_network` removes it
- **Normal case:** No stale VRF devices persist after CUDN deleted
- **Rare failure case:** Stale VRF requires manual recovery (see TC-DELETE-02 and Support Procedures)
- Subnet and VirtualNetwork CRs are deleted, finalizers removed

#### TC-DELETE-02: Stale VRF manual recovery (troubleshooting)

| Interface Change | Priority | Automation |
|-----------------|----------|------------|
| — | low | manual |

##### Preconditions

- **Tier / owner:** E2E / [QE].
- CUDN deleted successfully (confirmed via `oc get clusteruserdefinednetwork`)
- VRF device persists on worker node (observed rare race condition)
- Cloud Infrastructure Admin troubleshooting connectivity issue

##### Steps

1. Detect stale VRF:
   ```bash
   oc debug node/<node-name> -- chroot /host ip link show type vrf
   ```
2. Verify CUDN is deleted: `oc get clusteruserdefinednetwork <vnet-name>` returns NotFound
3. Follow manual recovery procedure from Support Procedures section
4. Option 1: Restart ovnkube-node pod (impacts all VMs on node)
5. Option 2: Direct VRF deletion via node debug (less disruptive)
6. Verify VRF cleaned up after recovery

##### Expected Results

- VRF cleanup procedure documented in Support Procedures
- **Automatic restart NOT performed by delete_subnet.yaml** (too disruptive for routine delete)
- Manual recovery successful (VRF removed after procedure)
- Documented impact: ovnkube-node restart affects all VMs on node (not just deleted namespace)

## Gaps

All requirements map to test cases. No case exercises IC-4. TC-R4-02 is Blocked until VM routing between Subnets lands in a later phase.
