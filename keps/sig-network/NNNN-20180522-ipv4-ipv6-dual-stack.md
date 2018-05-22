---
kep-number: 0
title: Kubernetes Dual Stack Support
authors:
  - "leblancd@"
  - "pcm@"
  - "rpothier@"
owning-sig: sig-network
participating-sigs:
  - sig-clusterlifecycle
reviewers:
  - TBD
approvers:
  - "thockin@"
editor: TBD
creation-date: 2018-05-21
last-updated: 2018-05-21
status: provisional

---

# IPv4/IPv6 Dual Stack

## Table of Contents

* [Table of Contents](#table-of-contents)
* [Summary](#summary)
* [Motivation](#motivation)
    * [Goals](#goals)
    * [Non-Goals](#non-goals)
* [Proposal](#proposal)
    * [User Stories [optional]](#user-stories-optional)
      * [Story 1](#story-1)
      * [Story 2](#story-2)
    * [Implementation Details/Notes/Constraints [optional]](#implementation-detailsnotesconstraints-optional)
    * [Risks and Mitigations](#risks-and-mitigations)
* [Graduation Criteria](#graduation-criteria)
* [Implementation History](#implementation-history)
* [Alternatives](#alternatives-optional)

[Tools for generating]: https://github.com/ekalinin/github-markdown-toc

## Summary

(Dane)
<needs work>
This proposal adds IPv4/IPv6 dual stack functionality to Kubernetes clusters. This includes the following concepts:
- Multiple IP address assignments per pod (Note 1)
- Multiple IP address assignments per node (Note 1)
- Multiple service IP assignments per service (Note 1)
- Dual-stack endpoints for Kubernetes services (Note 1)
- Native IPv4-to-IPv4 and IPv6-to-IPv6 communications within a cluster
- Parallel IPv4 and IPv6 ingress access to Kubernetes services

Note 1: Up to one IPv4 address, and up to one or more IPv6 addresses

## Motivation

The adoption of IPv6 has increased in recent years, and customers are requesting IPv6 support in Kubernetes clusters. To this end, the support of IPv6-only clusters was added as an alpha feature in Kubernetes Version 1.9. This work brought IPv6-only support to parity with IPv4-only support for Kubernetes clusters.

Using a single-IP-family cluster (IPv4-only or IPv6-only) works well when all pods, services, and external clients/services use the corresponding IP family. However, with scenarios that require legacy clients or services that are not compatible with the chosen cluster's IP family, then complex and expensive IPv4/IPv6 transition mechanisms (e.g. NAT64/DNS64, stateless NAT46, or SIIT/MAP) will need to be implemented in the data center networking.

An alternative is to allow for IPv4 and IPv6 pods and services to be supported in parallel (a.k.a. dual stack). This proposal aims to add dual-stack access to Kubernetes clusters, providing native IPv4-to-IPv4 communication and native IPv6-to-IPv6 communication to, from and within a Kubernetes cluster.


### Goals

- Pod Connectivity: IPv4-to-IPv4 and IPv6-to-IPv6 access between pods
- Service Access: IPv4-to-IPv4 and IPv6-to-IPv6 access from pods to Kubernetes services
- External Server Access: IPv4-to-IPv4 and IPv6-to-IPv6 access from pods to external servers
- Ingress Access: Access from IPv4 and/or IPv6 clients to Kubernetes services. Depending on the ingress controller used, connections may cross IP families (IPv4 clients to IPv6 services, or vice versa).
- Service Discovery: Support for both A and AAAA records for kube-DNS and coreDNS
- Functionality tested with the Bridge CNI plugin and Host-Local IPAM plugin as references
- Maintain support for IPv4-only and IPv6-only clusters

### Non-Goals

- Cross-family connectivity: IPv4-to-IPv6 and IPv6-to-IPv4 connectivity is considered outside of the scope of this proposal, except in the case of ingress access (depending on ingress controller used).
- CNI network plugins other than the Bridge and Host-Local IPAM plugins may support Kubernetes dual stack, but the development and testing of dual stack support for these other plugins is considered outside of the scope of this proposal.

## Proposal


### Multiple IPs per Pod

This feature requires a paradigm shift for pod IP assignments from support of a single IP address to support for multiple IP addresses per pod.

#### Primary Pod IP Selection Algorithm
- The first IPv4 address found in the "ips" list returned from the CNI plugin, if any.
- If there are no IPv4 addresses, then first IPv6 address found in the "ips" list.

The reasons that an IPv4 address is preferred as the primary pod IP are:
- Less ambiguity: There should be only one IPv4 address assigned to a pod.
- Consistency with node IP selection: 

### Container Networking Interface (CNI) Plugin Considerations

This feature requires the use of CNI API 0.3.1 or later. No changes to the CNI API specification are required for Kubernetes dual stack support. If a CNI plugin supports dual-stack assignments of IP addresses, it should return pod IP assignments to kubelet via the "ips" array defined in the [CNI plugin result specification](https://github.com/containernetworking/cni/blob/master/SPEC.md#result).

CNI plugins may return multiple IPv6 address per pod, but should return a maximum of one IPv4 address per pod.

The order of assigned IP addresses that are returned in the CNI plugin result for pod creation will be used to determine the "Primary Pod IP" as described in the previous section.

### Multiple Service VIPs per Service
### Kube-Proxy/IPVS Operation (Dane - kube-proxy)
### Support of Health Probes (Dane)
### Kube-DNS/Core-DNS Operation
### Ingress Controller Operation (Dane)
### Load Balancer Operation
### Network Policy Considerations
### Cloud Provider Plugins Considerations
### 

### API Changes
#### PodStatus

This feature requires a paradigm shift for pod IP assignments from support of a single IP address to support for multiple IP addresses.

##### Versioned API Changes: core V1 API
In order to maintain backwards compatibility for the core V1 API, this proposal adds a new "ExtraPodIPs" field to the core V1 version of the core API, while maintaining the existing "PodIP" field:
```
    // Primary IP address allocated to the pod. Routable at least within the
    // cluster. Empty if not yet allocated.
    // +optional
    PodIP string `json:"podIP,omitempty" protobuf:"bytes,6,opt,name=podIP"`

    // Additional IP addresses allocated to the pod. Routable at least within
    // the cluster. Empty if not yet allocated.
    // +optional
    ExtraPodIPs []string `json:"podIP,omitempty" protobuf:"bytes,6,opt,name=ExtrapodIPs"`
```

##### Internal Representation:
```
    // IP addresses allocated to the pod. Routable at least within the cluster.
    // Empty if not yet allocated.
    // +optional
    PodIPs []string `json:"podIP,omitempty" protobuf:"bytes,6,opt,name=podIP"`
```

#### Service Status

#### Node Resource

### kubectl CLI changes

### Kubernetes Controller Configuration Changes
#### kubelet Configuration Changes
#### kube-proxy Configuration Changes
#### kube-dns Configuration Changes
#### coreDns Configuration Changes

### kubeadm Support

### End-to-End Test Support


### User Stories [optional]

Detail the things that people will be able to do if this KEP is implemented.
Include as much detail as possible so that people can understand the "how" of the system.
The goal here is to make this feel real for users without getting bogged down.

#### Story 1

#### Story 2

### Implementation Details/Notes/Constraints [optional]

What are the caveats to the implementation?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they releate.

### Risks and Mitigations

What are the risks of this proposal and how do we mitigate.
Think broadly.
For example, consider both security and how this will impact the larger kubernetes ecosystem.

## Graduation Criteria

How will we know that this has succeeded?
Gathering user feedback is crucial for building high quality experiences and SIGs have the important responsibility of setting milestones for stability and completeness.
Hopefully the content previously contained in [umbrella issues][] will be tracked in the `Graduation Criteria` section.

[umbrella issues]: https://github.com/kubernetes/kubernetes/issues/42752

## Implementation History

Major milestones in the life cycle of a KEP should be tracked in `Implementation History`.
Major milestones might include

- the `Summary` and `Motivation` sections being merged signaling SIG acceptance
- the `Proposal` section being merged signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded

## Drawbacks [optional]

Why should this KEP _not_ be implemented.

## Alternatives [optional]

Similar to the `Drawbacks` section the `Alternatives` section is used to highlight and record other possible approaches to delivering the value proposed by a KEP.
