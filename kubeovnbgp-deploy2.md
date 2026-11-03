# Connecting Kube-OVN Overlay Networks to External Underlay Networks Using Provider Network and Establishing external connectivity using BGP

## 1. Objective

This document describes how to connect overlay to underlay using provider network and advertise routes for a custom Kube-OVN overlay subnet to an external FRR BGP peer.

The workload traffic path is:

**VM → Kube-OVN overlay subnet → provider network bridge (`br-pn1`) → Harvester host → external network**

The BGP peering session uses the Harvester host's management VLAN interface, `mgmt-br.2012`. The external FRR peer learns the overlay subnet and VM routes through this session.

## 2. Lab topology and BGP parameters

| Component | Value |
|---|---|
| Harvester nodes | `hp-46`, `hp-65` |
| Provider network | `pn1` |
| Physical interface | `eno50` |
| Provider bridge | `br-pn1` |
| VLAN resource | `untagged-vlan` |
| Overlay NAD | `vswitchinternal` |
| Custom VPC | `commonvpc` |
| Overlay subnet | `subnetinternal` |
| Overlay CIDR | `172.20.10.0/24` |
| Overlay gateway | `172.20.10.1` |
| BGP transport interface | `mgmt-br.2012` |
| Harvester BGP next hop | `10.115.14.43` |
| FRR peer IP | `10.115.13.86` |
| Harvester local AS | `65000` |
| FRR peer AS | `65030` |

## 3. Establish BGP peering with FRR

Create a VM connected to the Harvester VLAN 2012 L2 network. This VM will run FRRouting and act as the external BGP peer.

Install FRR:

```bash
sudo apt update
sudo apt install -y frr frr-pythontools
```

Enable the BGP daemon by setting the following in `/etc/frr/daemons`:

```ini
bgpd=yes
```

Configure `/etc/frr/frr.conf`:

```text
frr defaults traditional
hostname bgppeer-vm
service integrated-vtysh-config
!
router bgp 65030
 bgp router-id 10.115.13.86
 neighbor 10.115.14.43 remote-as 65000
 !
 address-family ipv4 unicast
  neighbor 10.115.14.43 route-map ACCEPT-ALL in
  neighbor 10.115.14.43 route-map ACCEPT-ALL out
 exit-address-family
exit
!
route-map ACCEPT-ALL permit 10
!
```

Restart FRR:

```bash
sudo systemctl restart frr
```

Verify the BGP configuration:

```bash
sudo vtysh
show running-config
show ip bgp summary
show ip bgp neighbors
show ip fib
```

The BGP session should become `Established` after installing the bgp speaker on harvester host.

## 4. Deploy the Kube-OVN BGP speaker

Label the Harvester nodes where the BGP speaker should run:

```bash
kubectl label nodes hp-65 ovn.kubernetes.io/bgp=true
kubectl label nodes hp-46 ovn.kubernetes.io/bgp=true
```

Deploy the speaker DaemonSet. The following example uses Kube-OVN v1.16.1 and the FRR peer details configured above.

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
          image: docker.io/kubeovn/kube-ovn:v1.16.1
          imagePullPolicy: IfNotPresent
          command:
            - /kube-ovn/kube-ovn-speaker
          args:
            - --logtostderr=false
            - --alsologtostderr=true
            - --log_file=/var/log/kube-ovn/kube-ovn-speaker.log
            - --log_file_max_size=200
            - --neighbor-address=10.115.13.86
            - --neighbor-as=65030
            - --cluster-as=65000
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

Apply the DaemonSet and verify the speaker pods:

```bash
kubectl apply -f speaker.yaml
kubectl get pods -n kube-system -o wide | grep kube-ovn-speaker
```

## 5. Create the provider network

Create a provider network using physical interface `eno50`.

```yaml
apiVersion: kubeovn.io/v1
kind: ProviderNetwork
metadata:
  name: pn1
spec:
  defaultInterface: eno50
  customInterfaces: []
  excludeNodes: []
```

Verify the provider network:

```bash
kubectl get provider-network pn1 -o yaml
```

The expected result is that the provider network becomes ready on the selected Harvester nodes and creates the provider bridge `br-pn1`.

## 6. Create an untagged VLAN

Create a VLAN resource associated with provider network `pn1`.

```yaml
apiVersion: kubeovn.io/v1
kind: Vlan
metadata:
  name: untagged-vlan
spec:
  id: 0
  provider: pn1
```

Apply and verify:

```bash
kubectl apply -f vlan.yaml
kubectl get vlan untagged-vlan -o yaml
```

## 7. Create the custom VPC

Create the custom VPC used by the overlay subnet.

```yaml
apiVersion: kubeovn.io/v1
kind: Vpc
metadata:
  name: commonvpc
```

Apply and verify:

```bash
kubectl apply -f vpc.yaml
kubectl get vpc commonvpc
```

## 8. Create the overlay network attachment definition

Create the `vswitchinternal` network attachment definition in the `default` namespace.

The provider name used by the Kube-OVN subnet is `vswitchinternal.default.ovn`.

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vswitchinternal
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "vswitchinternal",
      "type": "kube-ovn",
      "server_socket": "/run/openvswitch/kube-ovn-daemon.sock",
      "provider": "vswitchinternal.default.ovn"
    }
```

Apply:

```bash
kubectl apply -f nad.yaml
kubectl get network-attachment-definitions -n default
```

## 9. Create the custom overlay subnet attaching to the vlan of provider network

Create the `subnetinternal` subnet with the custom VPC, provider, and VLAN.

Enable BGP advertisement using the `ovn.kubernetes.io/bgp` annotation.

```yaml
apiVersion: kubeovn.io/v1
kind: Subnet
metadata:
  name: subnetinternal
  annotations:
    ovn.kubernetes.io/bgp: "true"
spec:
  cidrBlock: 172.20.10.0/24
  default: false
  enableDHCP: true
  enableLb: true
  excludeIps:
    - 172.20.10.1
  gateway: 172.20.10.1
  natOutgoing: false
  private: false
  protocol: IPv4
  provider: vswitchinternal.default.ovn
  vlan: untagged-vlan
  vpc: commonvpc
```

Apply and verify:

```bash
kubectl apply -f subnet.yaml
kubectl get subnet subnetinternal -o yaml
```

If the subnet already exists, enable BGP advertisement with:

```bash
kubectl annotate subnet subnetinternal ovn.kubernetes.io/bgp=true --overwrite
```

## 10. Create a VM on the overlay network

Create a VM and attach it to the `vswitchinternal` network. The VM should receive an IP address from `172.20.10.0/24`.

For example, a VM might receive:

```text
172.20.10.3
```

The subnet annotation enables Kube-OVN to advertise the subnet and relevant VM routes to the configured BGP peer.

## 11. Configure the provider bridge for return traffic

The intended traffic path is for VM traffic to reach the Harvester host through the provider bridge `br-pn1`, and for the host's routing configuration to forward traffic externally.

In the lab,configure the br-pn1 interface on each node with the following:

```bash
ip addr add 172.20.10.1/24 dev br-pn1
```

The corresponding connected routes are:

```text
172.20.10.0/24 dev br-pn1 proto kernel scope link src 172.20.10.1
```

Inspect the bridge and routes:

```bash
ip addr show br-pn1
ip route show | grep 172.20
```

## 12. Verify routes on the FRR peer

On the FRR VM:

```bash
sudo vtysh
show ip bgp summary
show ip bgp
show ip fib
```

The expected BGP routes include the overlay subnet and VM address. The observed lab output included:

```text
Network          Next Hop
172.20.10.0/24   10.115.14.43
172.20.10.3/32   10.115.14.43
```

Both should be reachable through the Harvester BGP next hop `10.115.14.43`.

## 13. Verify routes advertised by Kube-OVN

Find the speaker pod running on the relevant node:

```bash
kubectl get pods -n kube-system -o wide | grep kube-ovn-speaker
```

Open a shell in the speaker pod:

```bash
kubectl exec -it <speaker-pod> -n kube-system -- /bin/bash
```

Check the BGP neighbor:

```bash
gobgp neighbor
```

Inspect routes advertised to the FRR peer:

```bash
gobgp neighbor 10.115.13.86 adj-out
```

The output should include the custom overlay subnet and VM route:

```text
Network              Next Hop
172.20.10.0/24       10.115.14.43
172.20.10.3/32       10.115.14.43
```

## 14. Troubleshooting

**BGP session is not established**

- Check that the FRR VM can reach the Harvester host address `10.115.14.43`.
- Verify the FRR peer IP and AS configuration.
- Verify that `bgpd=yes` is enabled and FRR is running.
- Check the speaker pod logs and its BGP neighbor state.

**Subnet route is not advertised**

- Confirm that the subnet has `ovn.kubernetes.io/bgp: "true"`.
- Check the speaker's `gobgp neighbor <peer-ip> adj-out` output.
- Confirm that the speaker is running on the intended node and has the expected neighbor configuration.

**FRR learns the route but VM traffic fails**

- Check the FRR routing table and forwarding table.
- Check the Harvester host route to the overlay subnet.
- Verify the provider bridge and the intended gateway ownership.
- Check the VM's default gateway and the return path.
- Verify host forwarding and firewall rules.

## 15. Expected outcome

The Kube-OVN BGP speaker establishes an eBGP session with the external FRR peer over the management VLAN network. Once BGP advertisement is enabled on `subnetinternal`, the FRR peer learns the custom overlay subnet route and VM-specific route through the Harvester host's BGP next hop.

This provides the external routing information required to reach the overlay VM, provided that the provider bridge, host forwarding, and return path are also configured correctly.
