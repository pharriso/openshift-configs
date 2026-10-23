# MetalLB with BGP

In this example, MetalLB is configured in BGP mode to peer with an upstream router.

![image](../images/metallb-bgp.png)

This assumes you have deployed the MetalLB operator and an instance of MetalLB.

## MetalLB configuration

Configure BGP peering. Update the example config **bgp-peer.yaml** and update the ASN for MetalLB, the upstream router and the upstream router IP.

```
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: metallb-bgp-peer
  namespace: metallb-system
spec:
  myASN: 64500
  peerASN: 65000
  peerAddress: 192.168.50.1
```

Apply the config:

```
oc apply -f bgp-peer.yaml
```

Create **L2Advertisement** CRD for each VLAN. Examples in l2-advertisement-vlan10.yaml and l2-advertisement-vlan20.yaml

Create an address pool. Update **bgp-address-pool.yaml** as appropriate:

```
oc apply -f bgp-address-pool.yaml
```

Create the BGP advertisement:

```
oc apply -f bgp-advertise.yaml
```

## Test the deployment

When creating a LoadBalancer service with MetalLB, you can add annotations to request specific IP addresses or address from a specific IP Pool. In this example we will request a specific pool with this annotation on the Service config:

```
  annotations:
    metallb.io/address-pool: bgp-address-pool
```

Deploy a sample app and create a Loadbalancer service using the BGP advertisement.


```
oc new-project app1
oc create deployment app1 --image registry.redhat.io/rhel8/httpd-24:latest
oc apply -f expose-app-bgp.yaml
```

Check the services. You should see addresses from the relevant pools as the EXTERNAL-IP:

Here is app1 with an IP in the IP pool specified earlier.

```
oc get svc -n app1
NAME           TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)        AGE
app1-service   LoadBalancer   172.32.234.111   192.168.100.240   80:32185/TCP   32m
```

Test connectivity to the apps on the LB address:

```
curl -s http://192.168.100.240 | grep title
		<title>Test Page for the HTTP Server on Red Hat Enterprise Linux</title>
```

## Troubleshooting on the router

On the upstream router, validate peering:

```
# show bgp summary

IPv4 Unicast Summary (VRF default):
BGP router identifier 192.168.50.1, local AS number 65000 vrf-id 0
BGP table version 28
RIB entries 3, using 576 bytes of memory
Peers 3, using 2151 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
192.168.50.10   4      65000      8514      8515        0    0    0 00:19:11            1        1 N/A
192.168.50.11   4      65000      8514      8515        0    0    0 00:19:11            1        1 N/A
192.168.50.12   4      65000      5401      5401        0    0    0 00:19:11            1        1 N/A

Total number of neighbors 3
```

Check routes, in this example you should see 192.168.100.240 being advertised and a route for it:

```
# show ip route bgp
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

B>* 192.168.100.240/32 [200/0] via 192.168.50.10, virbr1, weight 1, 00:19:40
  *                            via 192.168.50.11, virbr1, weight 1, 00:19:40
  *                            via 192.168.50.12, virbr1, weight 1, 00:19:40
```
