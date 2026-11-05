# Kube-OVN BGP Integration – Setup and Validation Guide

## Overview

This document describes the steps to establish BGP peering between a Harvester cluster running Kube-OVN and an external FRR router.

The objective is to:
- Create an external VLAN network and an internal Kube-OVN overlay network.
- Connect the overlay network to the external network through a Kube-OVN VPC.
- Establish BGP peering between the Harvester host and the external FRR router.
- Advertise overlay subnet and VM routes to the external network.
- Validate end-to-end connectivity and route withdrawal.

## 1. Create the External Network

Create the external network infrastructure in Harvester/Kube-OVN:

1. Create the provider network.
2. Create the VLAN network using VLAN ID `2017`.
3. Create the Network Attachment Definition (NAD).
4. Create an external subnet and associate it with the VLAN and NAD created in step 3.

This provides the external network connectivity required for communication between Harvester nodes and the external network.

## 2. Create the Overlay Network

Create the internal Kube-OVN overlay network:

1. Create a custom VPC named `commonvpc`.
2. Create a NAD for the overlay network.
3. Create an internal overlay subnet and associate it with the overlay NAD.
4. Create a VM and attach it to the internal overlay subnet.

The VM will receive an IP address from the overlay subnet (for example, `172.20.10.2/24`).

## 3. Connect the Internal and External Networks

Connect the overlay network to the external network through the Kube-OVN VPC configuration.

1. Edit the existing `commonvpc` configuration.
2. Add the external subnet to the VPC spec.externalSubnets.
3. Enable `enableExternalSubnet`.

This establishes the external connectivity relationship between the VPC's internal overlay network and the external subnet.

## 4. Configure the Harvester Host Network

Create a HostNetworkConfig to configure the host-side external VLAN interface.

1. Create a HostNetworkConfig for VLAN 2017.
2. Configure the interface to create `mgmt-br.2017`.
3. Assign the static IP address `10.115.48.4/21` to the interface.

This interface provides the underlay connectivity used by the Harvester host's BGP Speaker to communicate with the external FRR router.

## 5. Configure the External FRR Router

Deploy an FRR router on a VM attached to the VLAN 2017 network.

Configure the FRR router with the following BGP peering information:

- FRR router IP: `10.115.53.2`
- FRR Autonomous System Number: `65030`
- Harvester BGP neighbor IP: `10.115.48.4`
- Harvester Autonomous System Number: `65000`

Configure the FRR BGP neighbor and other required BGP settings.

The FRR router will act as the external BGP peer for the Harvester cluster.

## 6. Select Harvester Nodes for BGP Speaker Deployment

Label the Harvester nodes that should run Kube-OVN BGP Speaker pods.

Apply the node label:

kubectl label nodes <speaker-node-1> ovn.kubernetes.io/bgp=true

The Kube-OVN BGP Speaker DaemonSet uses this label to determine which nodes should run BGP Speaker pods.

## 7. Install and Configure the Kube-OVN BGP Speaker

Install the speaker.yaml containing the following params.

Configure the BGP peering parameters:

- Local IP: `10.115.48.4`
- Local AS: `65000`
- Neighbor IP: `10.115.53.2`
- Neighbor AS: `65030`

The BGP Speaker DaemonSet should be deployed, and pods should run on the nodes labelled in the previous step.

## 8. Verify BGP Peering

Verify that the BGP Speaker is deployed and that the BGP session is successfully established.

**On the Harvester cluster:**

1. Verify that the Kube-OVN BGP Speaker DaemonSet is installed.
2. Verify that the BGP Speaker pods are running on the selected nodes.
3. Access a BGP Speaker pod.
4. Execute the GoBGP neighbor inspection command.
5. Verify that the BGP session with `10.115.53.2` is established.

**On the external FRR router:**

1. Log in to the FRR router VM.
2. Open the FRR CLI using `sudo vtysh`.
3. Inspect the BGP neighbor and session status.(show bgp neighbors)
4. Verify that the neighbor `10.115.48.4` is established.

Both sides should report an established BGP session.

## 9. Verify Route Advertisement

Enable BGP route advertisement for the Kube-OVN overlay subnet.

1. Annotate the internal overlay subnet with `ovn.kubernetes.io/bgp=true`.(kubectl annotate subnet sample ovn.kubernetes.io/bgp=true)
2. Verify that the Kube-OVN BGP Speaker advertises the overlay subnet routes.
3. Log in to the external FRR router.
4. Inspect the BGP routing table and IP routing table.
5. Verify that the expected overlay subnet and VM routes are received.(vtysh -> show ip route)

**Important validation:**

Verify that the advertised route uses the next-hop IP configured by the user (for example, the OVN EIP `10.115.48.3`) rather than the BGP neighbor IP (`10.115.53.2`) or the BGP peering source IP (`10.115.48.4`).

The advertised next hop must reflect the configured routing design.

## 10. Validate End-to-End Connectivity

Verify connectivity in both directions.

**External to internal:**

1. Log in to the FRR router VM.
2. Ping the overlay VM IP (`172.20.10.2`).
3. Verify that the VM is reachable through the advertised BGP route.

**Internal to external:**

1. Log in to the overlay VM.
2. Ping an IP address reachable through the external network.
3. Verify that outbound connectivity works as expected.

## 11. Verify Route Withdrawal

Validate that route advertisements are removed when BGP advertisement is disabled.

1. Remove the `ovn.kubernetes.io/bgp=true` annotation from the internal overlay subnet. (kubectl annotate subnet sample ovn.kubernetes.io/bgp- )
2. Allow the Kube-OVN BGP Speaker to reconcile the subnet configuration.
3. Log in to the external FRR router.
4. Inspect the BGP routing table and IP routing table.
5. Verify that the previously advertised overlay subnet and VM routes have been withdrawn.

This confirms that route advertisement and withdrawal are dynamically handled by the Kube-OVN BGP Speaker.

---

## Expected Outcome

At the end of this procedure:

- The external VLAN network and internal overlay network are connected through `commonvpc`.
- The Harvester host has underlay connectivity through `mgmt-br.2017`.
- The Kube-OVN BGP Speaker successfully peers with the external FRR router.
- Overlay subnet and VM routes are dynamically advertised to the external network.
- The advertised next hop matches the user-configured next-hop address.
- Traffic flows between the external network and overlay VMs.
- Removing the BGP annotation withdraws the advertised routes from the external router.
