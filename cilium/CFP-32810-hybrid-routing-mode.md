# CFP-32810: Add hybrid routing mode in Cilium

**SIG: SIG-NAME** SIG-Clustermesh

**Begin Design Discussion:** 2025-06-16

**Cilium Release:** N/A

**Authors:** Anubhab Majumdar <anmajumdar@microsoft.com>, Vamsi Kalapala <vakr@microsoft.com>, Krunal Jain <krunaljain@microsoft.com>

## Summary

Currently, Cilium supports two modes routing - [encapsulation](https://docs.cilium.io/en/stable/network/concepts/routing/#encapsulation) and [native-routing](https://docs.cilium.io/en/stable/network/concepts/routing/#native-routing). When workloads are on cross subnets with no direct connection, Cilium needs to run in encapsulation mode. This is not efficient, as every packet needs to be encapsulated, even when they are on same subnet, thus paying the MTU overhead and reducing network throughput. This CFP proposes adding a third option to allow Cilium dataplane to make the decision at runtime on whether to route the packet natively or encapsulate.

## Motivation

Consider any of the following scenarios:

1. Two Kubernetes cluster connected through cluster mesh. The nodes are backed by different subnets in two different VNETs, which are connected through VNET peering. The pods are assigned IP address from overlay network.
2. In a large Kubernetes cluster, pod addresses are assigned from two different subnets with no direct connection.

In either case, Cilium needs to run in encapsulation routing mode. However, when routing packets in above clusters, in many situations, we can get by native routing, and not pay the MTU overhead that comes with encapsulation. The reason for this inflexibility comes from the fact that Cilium datapath is not aware of the underlying network topology, and hence cannot make a decision between the two routing mode during runtime.

With higher adoption in cluster mesh and large scale clusters being common to support AI workloads, it would be beneficial to have a more performant dataplane that can optimize the throughput, while providing support for both encapsulation and native-routing.

## Goals

* Retain the behavior of current routing modes as-is
* Allow users to set a new routing mode called `hybrid`
* Allow users to pass in network topologies expressed through CIDR ranges during start-up

## Non-Goals

* Allow dynamic update to CIDR ranges (update without restart)

## Proposal

### Overview

Add a new option for routing-mode config as follows `routing-mode: hybrid`. Also, add a new field in config to pass information about the CIDRs and connectivity. The CIDR information will persist in pinned eBPF maps so that routing decision can be made in runtime by eBPF programs.

### Examples

```
subnet-topology: 10.0.0.1/24
```

If source and destination IP belongs to above CIDR, route natively. Else, encapsulate the packet.

```
subnet-topology: 10.0.0.1/24,10.10.0.1/24
```

The above two CIDR ranges have direct connectivity. If source and destination IP belongs to either CIDR, route natively. Else, encapsulate the packet.

```
subnet-topology: 10.0.0.1/24,10.10.0.1/24;10.20.0.1/24
```

* `10.0.0.1/24` and `10.10.0.1/24` are connected and traffic can be natively routed
* If source and destination IP belongs to `10.20.0.1/24`, route natively
* Else, encapsulate the packet


## Impacts / Key Questions

### Impact: Network throughput

Given that today Cilium is encapsulating packets that could be natively routed, removing those scenarios would yield higher throughput. This is especially true in clusters without jumbo frames.

### Key Question: Datapath latency

In a highly fragmented network topography, lookup of IP to make routing decision may not be negligible.

## Future Milestones

### Deferred Milestone 1

Allow dynamic configuration of CIDRs (add/remove) without agent restart.

### Deferred Milestone 2

Configure the subnets though a custom resource.
