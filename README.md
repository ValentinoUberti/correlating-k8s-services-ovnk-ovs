# Correlating OVN Load Balancer ID to OVS OpenFlow Rules in OVN-Kubernetes

This guide explains how to trace an OVN load balancer ID to OpenFlow rules in an OVN-Kubernetes environment like OpenShift, where load balancers are auto-generated for Kubernetes Services.

## Prerequisites

Access to the OpenShift \> 4.16 cluster with OVN-Kubernetes deployed (default CNI).

Command-line tools: kubectl (or oc), ovn-nbctl, ovn-sbctl, ovs-ofctl.

* ovn-nbctl and ovn-sbctl are found in one of the “ovnkube-node-xxxxx” pods in the openshift-ovn-kubernetes project  
* ovs-ofctl is present on every OpenShift nodes

Create the “bookinfo” project

```c
oc new-project bookinfo
```

   
Deploy the “bookinfo” application:

### 

```c
oc apply -f https://raw.githubusercontent.com/openshift-service-mesh/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
```

### Step 1: Identify the Kubernetes services cluster-ip

```c
oc get svc -n bookinfo
```

   
Example output:

```c
NAME                TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)         AGE
productpage         ClusterIP      172.30.80.46     <none>           9080/TCP        12d
...
```

Get the “productpage” service endpoints:

```c
oc get endpoints productpage -n bookinfo
```

Example output:

```c
NAME          ENDPOINTS          AGE
productpage   10.129.0.59:9080   12d
```

### Step 2: Identify the Load Balancer in OVN

List All the OVN Load Balancers

Run the following command to view load balancers in the OVN Northbound database within an ovnkube-node-xxxxx pod:

```c
ovn-nbctl lb-list | grep "172.30.80.46"
```

Example output:

```c
UUID             LB                   PROTO      VIP			      IPs
d461..d803       Service_bookinfo     tcp        172.30.80.46:9080       10.129.0.59:9080
```

Note the UUID, NAME, and VIP (ClusterIP).

The LB Name is truncated. The full name could be retrieved using the following command.

### Step 3: Verify OVN Load Balancer Details

Grep the k8s service ip from the OVN Load Balancer list.

```c
ovn-nbctl list Load_Balancer | grep "172.30.80.46" -B 9
```

Example:

```c
_uuid               : d461c9f3-1894-48e4-bb9c-1be45b31d803
external_ids        : {"k8s.ovn.org/kind"=Service, "k8s.ovn.org/owner"="bookinfo/productpage"}
health_check        : []
ip_port_mappings    : {}
name                : "Service_bookinfo/productpage_TCP_cluster"
options             : {event="false", hairpin_snat_ip="169.254.0.5 fd69::5", neighbor_responder=none, reject="true", skip_snat="false"}
protocol            : tcp
selection_fields    : []
vips                : {"172.30.80.46:9080"="10.129.0.59:9080,10.130.0.172:9080"}
```

### Step 4: Trace to the Southbound Database

Filter logical flows for the VIP:

```c
ovn-sbctl lflow-list | grep "172.30.80.46"
```

Example (hard to read) output:

```c
 table=6 (ls_in_pre_stateful ), priority=120  , match=(reg0[2] == 1 && ip4.dst == 172.30.80.46 && tcp.dst == 9080), action=(reg1 = 172.30.80.46; reg2[0..15] = 9080; ct_lb_mark;)
  table=13(ls_in_lb           ), priority=120  , match=(ct.new && ip4.dst == 172.30.80.46 && tcp.dst == 9080), action=(reg1 = 172.30.80.46; reg2[0..15] = 9080; ct_lb_mark(backends=10.129.0.59:9080);)
  table=6 (lr_in_defrag       ), priority=100  , match=(ip && ip4.dst == 172.30.80.46), action=(ct_dnat;)
  table=8 (lr_in_dnat         ), priority=120  , match=(ct.new && !ct.rel && ip4 && ip4.dst == 172.30.80.46 && tcp && tcp.dst == 9080), action=(flags.force_snat_for_lb = 1; ct_lb_mark(backends=10.129.0.59:9080; force_snat);)
```

### Step 5: Correlate to OVS OpenFlow Rules  
Dump OpenFlow Rules

On one OpenShift node running OVS (e.g., br-int), filter for the VIP:

```c
ovs-ofctl dump-flows br-int | grep "172.30.80.46"
```

Example output:

```c
cookie=0xcf3f348a, duration=22470.995s, table=14, n_packets=0, n_bytes=0, idle_age=22470, priority=120,tcp,reg0=0x4/0x4,metadata=0x3,nw_dst=172.30.80.46,tp_dst=9080 actions=load:0xac1e502e->NXM_NX_XXREG0[64..95],load:0x2378->NXM_NX_XXREG0[32..47],ct(table=15,zone=NXM_NX_REG13[0..15],nat)
 cookie=0x4f37dc6d, duration=22470.998s, table=14, n_packets=0, n_bytes=0, idle_age=22470, priority=100,ip,metadata=0x5,nw_dst=172.30.80.46 actions=ct(table=15,zone=NXM_NX_REG11[0..15],nat)
 cookie=0xaa45d1bf, duration=1991.111s, table=16, n_packets=0, n_bytes=0, idle_age=1991, priority=120,ct_state=+new-rel+trk,tcp,metadata=0x5,nw_dst=172.30.80.46,tp_dst=9080 actions=load:0x1->NXM_NX_REG10[3],group:222
 cookie=0xb823fb0a, duration=1991.111s, table=21, n_packets=0, n_bytes=0, idle_age=1991, priority=120,ct_state=+new+trk,tcp,metadata=0x3,nw_dst=172.30.80.46,tp_dst=9080 actions=load:0xac1e502e->NXM_NX_XXREG0[64..95],load:0x2378->NXM_NX_XXREG0[32..47],group:221
```

# Summary of Correlation

* Kubernetes Service: productpage, ClusterIP 172.30.80.46  
* OVN Load Balancer: UUID d461..d803, name Service\_bookinfo….  
* Southbound Flows: Match 172.30.80.46, DNAT to pod IPs.  
* OpenFlow Rules: Match nw\_dst=172.30.80.46, apply NAT in br-int.
