# Testing and Validation — Layer 2/3 Network High Availability Lab

## Validation Scope

Six completed Packet Tracer failure tests examined connectivity during link and distribution-switch failures, focusing on the interaction between HSRP gateway redundancy and Rapid-PVST+ forwarding-path changes.

The [README](README.md) provides the project overview and topology. This report contains the detailed design reference, test observations, and verification methods.

## Baseline State

Configured gateway preferences and STP root placement provide the reference state for the tests.

| Area | Baseline configuration / verification |
| --- | --- |
| HSRP, VLANs 10 and 20 | MLS1 preferred Active; MLS2 standby |
| HSRP, VLANs 30 and 40 | MLS2 preferred Active; MLS1 standby |
| HSRP priorities | Preferred switch 110; peer 100; preemption enabled |
| Rapid-PVST+ | Primary roots aligned with HSRP preferences; peer secondary |
| Inter-VLAN routing | SVIs on both distribution switches; hosts use virtual gateways |
| Infrastructure trunks | VLANs 10, 20, 30, 40, 999; native VLAN 999 |
| Po1 | Fa0/23 and Fa0/24; LACP active/active |
| Management | Basic VLAN 40 reachability verified |

### Devices and connections

| Role | Devices | Platform |
| --- | --- | --- |
| Distribution | MLS1, MLS2 | Cisco WS-C3560-24PS-E |
| Access | ASW1, ASW2 | Cisco 2960 |
| Endpoints | Admin-PC, Staff-PC1, Staff-PC2, Server | Packet Tracer hosts |

Each access switch has a Gigabit uplink to each distribution switch. The inter-distribution Po1 bundle uses two FastEthernet links.

| Connection | Interfaces |
| --- | --- |
| MLS1 ↔ ASW1 | Gi0/1 ↔ Gi0/1 |
| MLS1 ↔ ASW2 | Gi0/2 ↔ Gi0/1 |
| MLS2 ↔ ASW1 | Gi0/1 ↔ Gi0/2 |
| MLS2 ↔ ASW2 | Gi0/2 ↔ Gi0/2 |
| MLS1 ↔ MLS2, Po1 | Fa0/23 ↔ Fa0/23 and Fa0/24 ↔ Fa0/24 |

### VLANs and addressing

The HSRP virtual address is the default gateway for each routed VLAN. MLS1 uses `.2` and MLS2 uses `.3` for their respective SVIs.

| VLAN | Purpose | Subnet | Virtual gateway | MLS1 SVI | MLS2 SVI |
| --- | --- | --- | --- | --- | --- |
| 10 | ADMIN | 10.10.10.0/24 | 10.10.10.1 | 10.10.10.2 | 10.10.10.3 |
| 20 | STAFF | 10.20.20.0/24 | 10.20.20.1 | 10.20.20.2 | 10.20.20.3 |
| 30 | SERVER | 10.30.30.0/24 | 10.30.30.1 | 10.30.30.2 | 10.30.30.3 |
| 40 | MANAGEMENT | 10.40.40.0/24 | 10.40.40.1 | 10.40.40.2 | 10.40.40.3 |

VLAN 999 (NATIVE) is the native VLAN on infrastructure trunks. The access-switch management addresses are `10.40.40.10/24` for ASW1 and `10.40.40.11/24` for ASW2, with gateway `10.40.40.1`.

| Access switch | Port | Endpoint | VLAN |
| --- | --- | --- | --- |
| ASW1 | Fa0/1 | Staff-PC1 | 20 |
| ASW1 | Fa0/2 | Admin-PC | 10 |
| ASW2 | Fa0/1 | Staff-PC2 | 20 |
| ASW2 | Fa0/2 | Server | 30 |

Host-facing ports use access mode, PortFast, and BPDU Guard. Preferred STP roots align with HSRP gateway ownership to keep Layer 2 forwarding aligned with the preferred Layer 3 gateway where possible.

## Failure Test Matrix

Connectivity remained available or recovered after convergence across all six scenarios. The matrix separates reachability from the protocol behavior established during testing.

| Scenario | Observed behavior | Significance |
| --- | --- | --- |
| Single Po1 member failure | Po1 remained operational through its remaining member. | Bundle member redundancy worked. |
| Complete Po1 failure | HSRP failover was not necessarily triggered while alternate Layer 2 paths remained available. | Losing Po1 did not necessarily isolate the HSRP peers. |
| Single access-uplink failure | ASW1 lost its direct MLS1 link; MLS1 remained Active for VLAN 20 while STP redirected traffic. | The existing Active gateway remained reachable. |
| Partial MLS1 isolation | Connectivity remained available or recovered after convergence. | Connectivity behaved as intended under partial isolation. |
| Complete MLS1 failure | HSRP role transition and STP reconvergence occurred for affected VLANs. | Gateway and forwarding-path redundancy responded to switch loss. |
| Complete MLS2 failure | HSRP role transition and STP reconvergence occurred for affected VLANs. | Testing covered failure of either distribution switch. |

## Important Test Scenarios

### Access-uplink failure without gateway failover

When ASW1 lost its direct MLS1 link, STP redirected Layer 2 traffic while MLS1 remained HSRP Active for VLAN 20. The alternate path preserved access to the existing gateway without requiring an HSRP role change.

Packet Tracer Simulation Mode showed traffic passing through MLS2 and ASW2 toward MLS1. This was an observed failure-state path, not the intended normal path, and explained why losing the direct uplink did not make the Active gateway unreachable.

### Complete distribution-switch failure

MLS1 and MLS2 failures were tested separately. Both HSRP role transition and STP reconvergence occurred for affected VLANs, with connectivity remaining available or recovering after convergence.

MLS1 was preferred for VLANs 10 and 20; MLS2 for VLANs 30 and 40. Testing both switches demonstrated how gateway redundancy and Layer 2 forwarding responded together to distribution-switch loss.

## EtherChannel Validation

### Member redundancy

During the single-member failure test, Po1 remained operational through its remaining FastEthernet member, demonstrating LACP bundle resilience. Po1 being operational does not prove that user traffic actually traversed it.

### Complete bundle failure and path selection

With Po1 unavailable, alternate Layer 2 paths could still connect the distribution switches, so bundle loss did not necessarily trigger HSRP failover.

STP normally preferred lower-cost Gigabit-based paths through the access layer over the FastEthernet bundle. Po1 provided an additional logical path; its lower forwarding preference reflects the topology and interface-allocation tradeoff, not an EtherChannel fault.

## Verification Commands and Evidence

The following tools distinguished connectivity, HSRP state, STP forwarding state, and actual traffic paths.

| Command / tool | Verification purpose |
| --- | --- |
| `show standby brief` | HSRP roles and virtual gateways |
| `show spanning-tree vlan <id>` | Per-VLAN root, port roles, states, and path costs |
| `show spanning-tree interface <interface>` | STP information for the affected interface |
| `show etherchannel summary` | Bundle state and member participation |
| `show interfaces trunk` | Trunk operation and VLAN information |
| `show vlan brief` | VLAN existence and access-port assignment |
| `show ip interface brief` | Interface addressing and operational state |
| `ping` | Endpoint reachability |
| Packet Tracer Simulation Mode | Observed packet forwarding |

Ping alone did not establish protocol behavior. HSRP and STP checks identified role and forwarding-state changes; Simulation Mode explained the traffic path. The topology diagram shows physical connections, not per-VLAN STP forwarding state.

## Recovery and Validation Boundaries

After restoring the failed components, connectivity and the original preferred roles returned: MLS1 resumed HSRP Active and STP root roles for VLANs 10 and 20, and MLS2 resumed those roles for VLANs 30 and 40. HSRP preemption was configured to allow the preferred gateways to reclaim the Active role.

Packet Tracer validation is not physical-hardware benchmarking. No measured convergence time, packet-loss count, or lossless-failover claim is made. PortFast and BPDU Guard were configured but not deeply failure-tested; management checks covered basic reachability. SSH, Telnet, SNMP, AAA, and centralized management were outside the main objectives.

## Final Findings

The access-uplink test demonstrated Layer 2 recovery without an HSRP role change, while complete switch failures involved both gateway transition and STP reconvergence. Po1 member redundancy worked despite its lower normal forwarding preference.

These results show why connectivity, protocol state, and the observed traffic path must be evaluated together to explain redundancy behavior.
