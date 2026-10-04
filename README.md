# Layer 2/3 Network High Availability Lab

## Overview

This Cisco Packet Tracer lab explores how a redundant Layer 2/3 network maintains or restores user connectivity during uplink and distribution-switch failures. HSRP provides gateway redundancy, Rapid-PVST+ provides alternate Layer 2 paths, and SVIs provide inter-VLAN routing.

The project follows **Design → Configure → Verify → Break → Diagnose → Recover → Document**. Its main finding is that continued connectivity can result from a Layer 2 path change without an HSRP role change. Protocol state and traffic paths were examined alongside ping results to explain the behavior.

## Requirements and Scope

The design uses two distribution switches and two access switches, with each access switch connected to both distribution switches. It aims to:

- Provide redundant gateways for user, server, and management VLANs.
- Align preferred STP roots with preferred HSRP gateways.
- Maintain or recover connectivity during the tested link and switch failures.
- Verify protocol behavior before drawing conclusions from reachability alone.

HSRP, Rapid-PVST+, redundant uplinks, inter-VLAN routing, and failure recovery are the main focus. LACP EtherChannel, the management VLAN, native VLAN 999, PortFast, and BPDU Guard support the design.

## Topology

![Redundant Layer 2/3 lab topology](screenshots/topology.png)

The diagram shows physical topology, not per-VLAN STP forwarding state.

| Role | Devices | Platform |
| --- | --- | --- |
| Distribution | MLS1, MLS2 | Cisco WS-C3560-24PS-E |
| Access | ASW1, ASW2 | Cisco 2960 |
| Endpoints | Admin-PC, Staff-PC1, Staff-PC2, Server | Packet Tracer hosts |

Each access switch has a Gigabit uplink to each distribution switch. MLS1 and MLS2 also share a two-member FastEthernet LACP bundle, Port-Channel1 (Po1).

| Connection | Interfaces |
| --- | --- |
| MLS1 ↔ ASW1 | Gi0/1 ↔ Gi0/1 |
| MLS1 ↔ ASW2 | Gi0/2 ↔ Gi0/1 |
| MLS2 ↔ ASW1 | Gi0/1 ↔ Gi0/2 |
| MLS2 ↔ ASW2 | Gi0/2 ↔ Gi0/2 |
| MLS1 ↔ MLS2, Po1 | Fa0/23 ↔ Fa0/23 and Fa0/24 ↔ Fa0/24 |

## VLAN and Addressing Design

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

## Technical Design

### Gateway redundancy and STP alignment

| VLANs | Preferred HSRP Active / STP primary root | HSRP standby / STP secondary root |
| --- | --- | --- |
| 10, 20 | MLS1 | MLS2 |
| 30, 40 | MLS2 | MLS1 |

The preferred HSRP switch has priority 110; its peer has priority 100. Preemption is configured so the preferred switch can reclaim the Active role after recovery. Rapid-PVST+ root placement follows the same ownership split to align Layer 2 forwarding with the preferred Layer 3 gateway where possible.

### Routing, trunks, and supporting controls

The distribution switches route between VLANs using SVIs. Infrastructure trunks carry VLANs 10, 20, 30, 40, and 999, with VLAN 999 configured as native.

Po1 uses LACP in active/active mode. Host-facing ports use access mode, PortFast, and BPDU Guard. VLAN 40 provides switch management connectivity; basic management reachability was verified.

## Verification Approach

Verification combined ping, IOS state checks, and Packet Tracer Simulation Mode. HSRP roles, STP paths, trunk membership, SVI state, and EtherChannel state were examined to distinguish continued reachability from the protocol mechanism that enabled it.

See [Testing and Validation](TESTING-AND-VALIDATION.md) for the test matrix, detailed observations, verification commands, and validation boundaries.

## Testing Summary

Six failure scenarios were completed:

| Failure type | Observed behavior |
| --- | --- |
| Single Po1 member failure | Po1 remained operational through its remaining member. |
| Complete Po1 failure | HSRP failover was not necessarily triggered while alternate Layer 2 paths remained available. |
| Single access-uplink failure | MLS1 remained HSRP Active for VLAN 20 while STP redirected traffic. |
| Partial MLS1 isolation | Connectivity remained available or recovered after convergence. |
| Complete MLS1 failure | HSRP role transition and STP reconvergence occurred for affected VLANs. |
| Complete MLS2 failure | HSRP role transition and STP reconvergence occurred for affected VLANs. |

Connectivity remained available or recovered after convergence across the tested scenarios.

## Key Findings

- A failed direct uplink does not necessarily require a gateway role change: the current Active gateway may remain reachable through another Layer 2 path.
- Complete distribution-switch failure involves both gateway redundancy and Layer 2 reconvergence for affected VLANs.
- A working EtherChannel is not necessarily the preferred STP forwarding path.
- Ping establishes reachability; protocol commands and traffic tracing explain how that reachability is achieved.

## Limitations and Future Improvements

### EtherChannel design tradeoff

Po1 uses FastEthernet members, while the access-to-distribution links use Gigabit Ethernet. In the tested topology, STP normally preferred lower-cost Gigabit-based paths through the access layer over Po1.

Po1 provides an additional logical path and successfully tolerated a member failure. Its lower forwarding preference is an interface-allocation tradeoff, not an EtherChannel failure.

A future version could use distribution switches with enough Gigabit interfaces for both redundant access uplinks and a Gigabit inter-distribution EtherChannel.

### Validation boundaries

This is a Packet Tracer lab, not a production availability benchmark. No claim is made about lossless failover, precise recovery time, or physical-hardware performance. SSH, Telnet, SNMP, AAA, and centralized management were outside the main objectives. PortFast and BPDU Guard are configuration controls, not demonstrated fault-test outcomes.

Further validation could measure recovery time and packet loss, and test the configured edge-port protections.

## Repository Structure

The repository separates the design overview, detailed validation, device configurations, and Packet Tracer lab file.

<details>
<summary>View repository structure</summary>

<pre>
Redundant-Layer2-Layer3-HA-Lab/
├── <a href="README.md">README.md</a>
├── <a href="TESTING-AND-VALIDATION.md">TESTING-AND-VALIDATION.md</a>
├── <a href="packet-tracer/">packet-tracer/</a>
│   └── <a href="packet-tracer/Layer 2-3 Network High Availability Lab.pkt">Layer 2-3 Network High Availability Lab.pkt</a>
├── <a href="configs/">configs/</a>
│   ├── <a href="configs/MLS1-config.txt">MLS1.txt</a>
│   ├── <a href="configs/MLS2-config.txt">MLS2.txt</a>
│   ├── <a href="configs/ASW1-config.txt">ASW1.txt</a>
│   └── <a href="configs/ASW2-config.txt">ASW2.txt</a>
└── <a href="screenshots/">screenshots/</a>
    └── <a href="screenshots/topology.png">topology.png</a>
</pre>

</details>
