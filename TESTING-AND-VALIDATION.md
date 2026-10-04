# Testing and Validation — Layer 2/3 Network High Availability Lab

## Validation Scope

Six completed Packet Tracer failure tests examined connectivity during link and distribution-switch failures, focusing on the interaction between HSRP gateway redundancy and Rapid-PVST+ forwarding-path changes.

The [README](README.md) covers topology, addressing, and design decisions. This report describes observed reachability, protocol state, and forwarding behavior.

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

Packet Tracer validation is not physical-hardware benchmarking. No measured convergence time, packet-loss count, or lossless-failover claim is made. PortFast and BPDU Guard were configured but not deeply failure-tested; management checks covered basic reachability.

## Final Findings

The access-uplink test demonstrated Layer 2 recovery without an HSRP role change, while complete switch failures involved both gateway transition and STP reconvergence. Po1 member redundancy worked despite its lower normal forwarding preference.

These results show why connectivity, protocol state, and the observed traffic path must be evaluated together to explain redundancy behavior.
