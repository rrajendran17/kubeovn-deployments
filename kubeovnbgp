# Connecting Kube-OVN Overlay Networks to External Underlay Networks Using BGP

## Objective

This document describes how to establish connectivity between a Kube-OVN overlay network and an external underlay network and advertise overlay network routes to an external BGP peer running FRRouting (FRR).

The setup uses:

 - **Harvester Provider Network:** Provides connectivity to the physical network.
 - **VLAN Network:** Provides Layer 2 connectivity between Harvester nodes and the external router.
 - **Kube-OVN VPC:** Provides an isolated routing domain for overlay networks.
 - **External Subnet:** Connects the custom VPC to the physical underlay network.
 - **Kube-OVN BGP Speaker:** Uses GoBGP to advertise overlay subnet and VM routes.
 - **FRRouting (FRR):** Acts as the external BGP router and learns the advertised routes.

### Expected outcome

Once configured:

1. A VM connected to the Kube-OVN overlay subnet receives an IP address (for example, `172.20.10.2`).
2. The custom VPC has connectivity to the external underlay through its external logical router port.
3. The Kube-OVN BGP speaker establishes an eBGP session with the external FRR router.
4. The BGP speaker advertises the overlay subnet (`172.20.10.0/24`) and VM IP (`172.20.10.2/32`) to FRR.
5. FRR installs the received routes in its BGP routing table and, subject to route selection, its Linux routing table.

### Network Architecture

#### Logical network diagram

The following diagram illustrates the relationship between the overlay, underlay, BGP speaker, and external router.

```
                  HARVESTER CLUSTER
        +--------------------------------------+
        |                                      |
        |       Kube-OVN Custom VPC             |
        |             commonvpc                 |
        |                                      |
        |    +---------------------------+     |
        |    | Overlay Subnet            |     |
        |    | subnetinternal            |     |
        |    | 172.20.10.0/24             |     |
        |    |                           |     |
        |    |  +---------------------+  |     |
        |    |  | VM                  |  |     |
        |    |  | 172.20.10.2/24      |  |     |
        |    |  +---------------------+  |     |
        |    +-------------+-------------+     |
        |                  |                   |
        |           OVN Logical Router         |
        |                  |                   |
        |    +-------------+-------------+     |
        |    | External Logical Router   |     |
        |    | Port                      |     |
        |    | 10.115.48.3/21            |     |
        |    +-------------+-------------+     |
        |                  |                   |
        |          External Subnet             |
        |          10.115.48.0/21              |
        |                  |                   |
        |    +-------------+-------------+     |
        |    | Provider Network / VLAN   |     |
        |    | VLAN 2017                 |     |
        |    +-------------+-------------+     |
        |                  |                   |
        |    +-------------+-------------+     |
        |    | Harvester Node            |     |
        |    |                           |     |
        |    | BGP Speaker (GoBGP)        |     |
        |    | Host interface:           |     |
        |    | 10.115.48.4/21            |     |
        |    +-------------+-------------+     |
        +------------------|-------------------+
                           |
                     VLAN 2017
                           |
              +------------+-------------+
              |                          |
              |    External FRR Router   |
              |                          |
              |    10.115.53.2/21        |
              |    AS 65030              |
              |                          |
              +--------------------------+

       BGP Session:
       Harvester AS 65000 <----> FRR AS 65030

       Advertised routes:
         172.20.10.0/24
         172.20.10.2/32
```

**Note:** The diagram is logical. The external logical router port and the BGP speaker's host interface are separate networking components. The BGP speaker establishes its BGP session using the host's underlay IP address, not the VM's overlay IP address.

#### Example IP addressing

The following addresses are used throughout this document. Replace them with addresses from your environment.

| Component | IP address / CIDR | Description |
|---|---|---|
| Provider Network | Physical NIC `eno50` | Underlay uplink |
| VLAN | 2017 | External connectivity |
| External subnet | `10.115.48.0/21` | Underlay network |
| External gateway | `10.115.55.254` | Underlay gateway |
| VPC external LRP | `10.115.48.3` | External logical router port |
| BGP speaker node | `10.115.48.4` | Host interface IP |
| FRR router | `10.115.53.2` | External BGP peer |
| Harvester BGP AS | `65000` | Local AS |
| FRR BGP AS | `65030` | Neighbor AS |
| Overlay subnet | `172.20.10.0/24` | VM network |
| Overlay gateway | `172.20.10.1` | VM default gateway |
| VM IP | `172.20.10.2` | VM address |

#### Configure the External Underlay Network

The first step is to provide physical network connectivity between the Harvester nodes and the external FRR router.

##### Create a Provider Network

Create a Provider Network that uses the physical interface connected to the external network.

Example:

```yaml
apiVersion: kubeovn.io/v1
kind: ProviderNetwork
metadata:
  name: pn1
spec:
  defaultInterface: eno50
```

Here:

- `pn1` is the Provider Network name.
- `eno50` is the physical NIC used for external connectivity.

Make sure the selected physical NIC is connected to the appropriate physical network.

##### Create a VLAN

Create a VLAN resource associated with the Provider Network.

```yaml
apiVersion: kubeovn.io/v1
kind: Vlan
metadata:
  name: vlan2017
spec:
  id: 2017
  provider: pn1
```

This associates VLAN 2017 with the Provider Network `pn1`.

##### Create the external Network Attachment Definition

Create a Network Attachment Definition (NAD) that allows Kube-OVN to use the external provider network.

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: external
  namespace: default
  labels:
    network.harvesterhci.io/ready: "true"
    network.harvesterhci.io/type: OverlayNetwork
spec:
  config: '{"cniVersion":"0.3.1","name":"external","type":"kube-ovn","server_socket":"/run/openvswitch/kube-ovn-daemon.sock","provider":"external.default.ovn"}'
```

The provider name follows this convention:

`<nad-name>.<nad-namespace>.ovn`

Therefore, the provider name is `external.default.ovn`.

##### Create the external Subnet

Create an external subnet in the default Kube-OVN VPC (`ovn-cluster`).

```yaml
apiVersion: kubeovn.io/v1
kind: Subnet
metadata:
  name: external
spec:
  cidrBlock: 10.115.48.0/21
  gateway: 10.115.55.254
  protocol: IPv4
  provider: external.default.ovn
  vlan: vlan2017
  vpc: ovn-cluster
  default: false
  natOutgoing: false
```

Important configuration:

- `cidrBlock`: Underlay subnet CIDR.
- `gateway`: Gateway address of the external network.
- `provider`: Provider associated with the external NAD.
- `vlan`: VLAN resource created earlier.
- `vpc`: The default Kube-OVN VPC.
- `natOutgoing`: Disabled because this is an external network connection, not a NAT-based VM subnet.

##### Verify the external network

Check the resources:

```bash
kubectl get providernetwork
kubectl get vlan
kubectl get network-attachment-definitions -n default
kubectl get subnet external
```

The external subnet should be ready before proceeding.

#### Configure the Overlay VPC

The next step is to create a custom VPC and connect it to the external subnet.

##### Create a custom VPC

Create a VPC named `commonvpc`.

```yaml
apiVersion: kubeovn.io/v1
kind: Vpc
metadata:
  name: commonvpc
spec:
  enableExternal: true
  extraExternalSubnets:
    - external
  staticRoutes:
    - cidr: 0.0.0.0/0
      nextHopIP: 10.115.55.254
      policy: policyDst
```

Explanation:

- `enableExternal: true` enables external connectivity for the custom VPC.
- `extraExternalSubnets` associates the external subnet with the custom VPC.
- `staticRoutes` configures the default route through the external gateway.
- `policyDst` applies the route based on the destination address.

The external subnet provides the connectivity between the custom VPC and the physical underlay.

##### Create the overlay Network Attachment Definition

Create an overlay NAD for the VM network.

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vswitchinternal
  namespace: default
  labels:
    network.harvesterhci.io/ready: "true"
    network.harvesterhci.io/type: OverlayNetwork
spec:
  config: '{"cniVersion":"0.3.1","name":"vswitchinternal","type":"kube-ovn","server_socket":"/run/openvswitch/kube-ovn-daemon.sock","provider":"vswitchinternal.default.ovn"}'
```

This creates the overlay provider used by the VM subnet.

##### Create the overlay Subnet

Create a subnet named `subnetinternal` in `commonvpc`.

```yaml
apiVersion: kubeovn.io/v1
kind: Subnet
metadata:
  name: subnetinternal
spec:
  cidrBlock: 172.20.10.0/24
  gateway: 172.20.10.1
  protocol: IPv4
  provider: vswitchinternal.default.ovn
  vpc: commonvpc
  natOutgoing: false
  enableDHCP: true
  default: false
```

Important:

`natOutgoing` is set to `false` because the intention is to advertise the original overlay IP addresses to the external network instead of translating them to an external IP.

##### Create a VM

Create a VM and attach it to `vswitchinternal`.

The VM should receive an address similar to:

```text
IP address: 172.20.10.2/24
Gateway:    172.20.10.1
```

Verify the VM connectivity to its gateway:

```bash
ping 172.20.10.1
```

##### Verify the VPC routing configuration

Check the VPC logical router:

```bash
kubectl exec -it -n kube-system <ovn-central-pod> -- \
  ovn-nbctl show commonvpc
```

Expected logical router structure:

```text
router commonvpc
    port commonvpc-external
        networks: ["10.115.48.3/21"]
    port commonvpc-subnetinternal
        networks: ["172.20.10.1/24"]
```

Check the routing table:

```bash
ovn-nbctl lr-route-list commonvpc
```

Expected output:

```text
IPv4 Routes
Route Table <main>:
    172.20.10.0/24    172.20.10.1    src-ip
    0.0.0.0/0         10.115.55.254  dst-ip
```

The logical router now has an external connection and a default route pointing to the external gateway.

#### Deploy the Kube-OVN BGP Speaker

The BGP speaker is responsible for establishing a BGP session with the external FRR router and advertising overlay routes.

Kube-OVN uses GoBGP internally to implement this functionality.

##### Configure the BGP speaker node

The BGP speaker must run on a node that has connectivity to the external underlay network.

For this example, the node has the following interface:

```text
mgmt-br.2017@mgmt-br:
    inet 10.115.48.4/21
```

Label the node to allow the BGP speaker DaemonSet to schedule on it:

```bash
kubectl label nodes speaker-node-1 ovn.kubernetes.io/bgp=true
```

Replace `speaker-node-1` with the actual Harvester node name.

##### Install the BGP speaker

Deploy the following DaemonSet.

Use a Kube-OVN image matching the version deployed in the cluster. The image below is the example image used in this lab.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-ovn-speaker
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: kube-ovn-speaker
  template:
    metadata:
      labels:
        app: kube-ovn-speaker
        component: network
        type: infra
    spec:
      tolerations:
        - operator: Exists
          effect: NoSchedule
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: kube-ovn-speaker
              topologyKey: kubernetes.io/hostname
      priorityClassName: system-node-critical
      serviceAccountName: ovn
      automountServiceAccountToken: true
      hostNetwork: true
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: kube-ovn-speaker
          image: renukaraaj17/kube-ovn:v1.16.1
          imagePullPolicy: IfNotPresent
          command:
            - /kube-ovn/kube-ovn-speaker
          args:
            - --logtostderr=false
            - --alsologtostderr=true
            - --log_file=/var/log/kube-ovn/kube-ovn-speaker.log
            - --log_file_max_size=200
            - --neighbor-address=10.115.53.2
            - --neighbor-as=65030
            - --cluster-as=65000
            - --node-ips=10.115.48.4
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: POD_IPS
              valueFrom:
                fieldRef:
                  fieldPath: status.podIPs
          resources:
            requests:
              cpu: 500m
              memory: 300Mi
          volumeMounts:
            - name: kube-ovn-log
              mountPath: /var/log/kube-ovn
      nodeSelector:
        kubernetes.io/os: linux
        ovn.kubernetes.io/bgp: "true"
      volumes:
        - name: kube-ovn-log
          hostPath:
            path: /var/log/kube-ovn
            type: DirectoryOrCreate
```

BGP arguments:

| Argument | Description |
|---|---|
| `neighbor-address` | IP address of the external FRR peer |
| `neighbor-as` | AS number of the FRR router |
| `cluster-as` | Local AS number used by the Kube-OVN speaker |
| `node-ips` | Underlay IP address used by the speaker node |

In this example:

- Local AS: `65000`
- Neighbor AS: `65030`
- Neighbor IP: `10.115.53.2`
- Speaker node IP: `10.115.48.4`

The speaker uses host networking so that it can establish the BGP session directly over the node's underlay network.

Apply the DaemonSet:

```bash
kubectl apply -f kube-ovn-speaker.yaml
```

Verify:

```bash
kubectl get pods -n kube-system -o wide | grep kube-ovn-speaker
```

##### Enable BGP advertisement for the overlay subnet

Annotate the overlay subnet:

```bash
kubectl annotate subnet subnetinternal ovn.kubernetes.io/bgp=true
```

This tells Kube-OVN to advertise the subnet through the BGP speaker.

The default subnet announcement policy is `Cluster`, meaning that routes can be advertised from each running speaker, regardless of whether the VM is hosted on that particular node.

For this reason, if multiple speaker instances are deployed, the external router should support ECMP if it is expected to use multiple advertised next hops.

#### Configure the External FRR Router

The external router runs FRRouting (FRR) on Linux and acts as the BGP neighbor of the Kube-OVN speaker.

##### Install FRR

On the Linux machine that will act as the BGP peer:

```bash
sudo apt update
sudo apt install -y frr frr-pythontools
```

##### Enable the BGP daemon

Edit:

```bash
sudo vi /etc/frr/daemons
```

Enable the BGP daemon:

```
bgpd=yes
```

Restart FRR:

```
sudo systemctl restart frr
```

### 6.3 Configure the FRR BGP neighbor

Edit:

```
sudo vi /etc/frr/frr.conf
```

Example configuration:

```text
frr version 8.1
frr defaults traditional
hostname bgppeer-vlan2017
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
router bgp 65030
 bgp router-id 10.115.53.2
 neighbor 10.115.48.4 remote-as 65000
 !
 address-family ipv4 unicast
  neighbor 10.115.48.4 activate
  neighbor 10.115.48.4 soft-reconfiguration inbound
 exit-address-family
exit
!
end
```

Configuration explanation:

| Parameter | Value | Description |
|---|---|---|
| Local AS | `65030` | FRR's AS number |
| Router ID | `10.115.53.2` | FRR BGP router ID |
| Neighbor | `10.115.48.4` | Harvester BGP speaker IP |
| Remote AS | `65000` | Kube-OVN BGP AS |

The `soft-reconfiguration inbound` setting allows FRR to retain received routes for inspection using the `received-routes` command.

The earlier `ACCEPT-ALL` route-map configuration can also be used if required. This example omits it because no additional route filtering is required in this basic lab.

Restart FRR:

```bash
sudo systemctl restart frr
```

### 6.4 Verify the FRR configuration

Enter the FRR shell:

```bash
sudo vtysh
```

Check the running configuration:

```text
show running-config
```

Check the BGP session:

```text
show ip bgp summary
```

Check neighbor details:

```text
show ip bgp neighbors
```

Check routes received from Harvester:

```text
show bgp ipv4 unicast neighbors 10.115.48.4 received-routes
```

Check the BGP routing table:

```text
show ip bgp
```

Check the Linux routing table:

```text
show ip route
```

Check the forwarding information base:

```text
show ip fib
```

## 7. Verify BGP Route Advertisement

After deploying the speaker and configuring FRR, verify that the BGP session is established.

### 7.1 Verify the session from Harvester

On the BGP speaker node, inspect the GoBGP neighbor state:

```bash
kubectl exec -it -n kube-system <speaker-pod> -- gobgp neighbor
```

Expected output:

```text
Peer           AS     Up/Down      State
10.115.53.2    65030  02:04:29     Establ
```

`Establ` indicates that the BGP session has been successfully established.

#### Verify advertised routes on FRR

On the FRR router:

```text
show ip bgp
```

Expected output:

```text
BGP table version is 22, local router ID is 10.115.53.2
local AS number 65030

   Network          Next Hop        Metric LocPrf Weight Path
*> 172.20.10.0/24   10.115.48.4                         0 65000 i
*> 172.20.10.2/32   10.115.48.4                         0 65000 i
```

The output demonstrates that FRR has learned two routes:

| Advertised route | Meaning |
|---|---|
| `172.20.10.0/24` | Entire overlay subnet |
| `172.20.10.2/32` | Individual VM IP |

The next hop for both routes is `10.115.48.4`, which is the underlay IP address of the Harvester node running the BGP speaker.

The AS path contains `65000`, indicating that the routes originated from the Kube-OVN AS.

#### Verify routes installed in the Linux routing table

Run:

```text
show ip route
```

Expected output:

```text
B>* 172.20.10.0/24 [20/0] via 10.115.48.4, enp1s0
B>* 172.20.10.2/32 [20/0] via 10.115.48.4, enp1s0
```

This indicates that FRR has selected the BGP routes for installation into the Linux routing table.

The external router now knows that traffic destined for `172.20.10.0/24` must be forwarded to the Harvester node at `10.115.48.4`.

#### End-to-End Traffic Flow

The route advertisement establishes the control-plane connectivity. For actual VM communication, the underlay and overlay forwarding paths must also work.

Consider an external machine trying to reach VM `172.20.10.2`.

##### Forward traffic

1. The external machine sends traffic to `172.20.10.2`.
2. Its gateway (FRR) looks up the destination in its routing table.
3. FRR matches `172.20.10.2/32` or `172.20.10.0/24`.
4. FRR forwards the packet to the next hop `10.115.48.4`.
5. The packet reaches the Harvester node running the BGP speaker.
6. Kube-OVN/OVN forwarding delivers the packet to the VM's overlay network.
7. The VM receives the traffic.

##### Return traffic

1. The VM sends its response through its default gateway `172.20.10.1`.
2. The Kube-OVN logical router processes the packet.
3. The VPC uses its routing configuration to forward traffic towards the external network.
4. The external network receives the response.

**Important:** BGP route advertisement alone does not guarantee end-to-end connectivity. The forwarding path, return route, firewall rules, and absence of unintended NAT must also be verified. In particular, the VPC routing configuration must provide a valid return path to the external destination.

### Troubleshooting

| Problem | Checks |
|---|---|
| BGP session not established | Verify peer IP, AS numbers, host interface IP, routing and TCP port 179 |
| Speaker pod not running | Check node label, DaemonSet, image and pod logs |
| BGP established but no routes | Check the subnet BGP annotation and speaker logs |
| FRR learns only subnet route | Check the speaker's announcement behavior and VM/subnet configuration |
| FRR has routes but VM is unreachable | Verify next-hop reachability, OVN logical routing, firewall and return path |
| Routes disappear after node failure | Check speaker placement, ECMP and BGP convergence |
| Wrong next-hop address | Verify the node interface selected for the BGP session and advertised next hop |

### Useful commands:

**Harvester:**

```bash
kubectl get pods -n kube-system -o wide | grep speaker

kubectl logs -n kube-system <speaker-pod>

kubectl get subnet subnetinternal -o yaml

kubectl exec -it -n kube-system <ovn-central-pod> -- \
  ovn-nbctl lr-route-list commonvpc
```

**FRR:**

```text
show ip bgp summary
show ip bgp
show ip bgp neighbors 10.115.48.4
show ip route bgp
show ip fib
```

### Cleanup

To stop advertising the overlay subnet:

```bash
kubectl annotate subnet subnetinternal ovn.kubernetes.io/bgp-
```

To remove the speaker:

```bash
kubectl delete daemonset kube-ovn-speaker -n kube-system
```

Remove the node label if it is no longer needed:

```bash
kubectl label nodes speaker-node-1 ovn.kubernetes.io/bgp-
```

Remove the VPC, subnet, VLAN, Provider Network and NAD resources only after ensuring that no VMs or other workloads depend on them.

### Summary

This implementation connects a Kube-OVN overlay network to an external underlay network using a custom VPC and an external subnet.

The Provider Network and VLAN establish physical underlay connectivity. The external subnet connects the custom VPC to that underlay. The BGP speaker runs on a Harvester node with underlay connectivity and establishes an eBGP session with an external FRR router.

Once the overlay subnet is annotated with `ovn.kubernetes.io/bgp=true`, Kube-OVN advertises the overlay routes to the external BGP peer.

The external router learns the overlay subnet and VM IP addresses and installs the corresponding routes, enabling external networks to route traffic towards the overlay VM IPs.

The key separation of responsibilities is:

- **Provider Network / VLAN:** Physical underlay connectivity.
- **External Subnet / VPC:** Overlay-to-underlay logical routing.
- **Kube-OVN BGP Speaker:** Advertisement of overlay routes.
- **FRR:** External BGP route learning and forwarding.
