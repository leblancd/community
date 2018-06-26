---
kep-number: 0
title: Kubernetes Dual Stack Support
authors:
  - "leblancd@"
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

<<<<<< NOTE: THIS SPECIFICATION IS A WORK-IN-PROGRESS >>>>>>

Table of Contents
=================

   * [IPv4/IPv6 Dual Stack](#ipv4ipv6-dual-stack)
   * [Table of Contents](#table-of-contents)
      * [Summary](#summary)
      * [Motivation](#motivation)
         * [Goals](#goals)
         * [Non-Goals](#non-goals)
      * [Proposal](#proposal)
         * [Awareness of Multiple IPs per Pod](#awareness-of-multiple-ips-per-pod)
            * [Versioned API Change: PodStatus v1 core](#versioned-api-change-podstatus-v1-core)
               * [Default Pod IP Selection](#default-pod-ip-selection)
            * [PodStatus Internal Representation:](#podstatus-internal-representation)
            * [kubelet Startup Configuration for Multiple Pod CIDRs](#kubelet-startup-configuration-for-multiple-pod-cidrs)
            * [kube-proxy Startup Configuration for Multiple Pod CIDRs](#kube-proxy-startup-configuration-for-multiple-pod-cidrs)
         * [Container Networking Interface (CNI) Plugin Considerations](#container-networking-interface-cni-plugin-considerations)
         * [Support for Multiple Service IPs per Service](#support-for-multiple-service-ips-per-service)
            * [First Release of Dual Stack: No Service Address Configuration "Knobs"](#first-release-of-dual-stack-no-service-address-configuration-knobs)
            * [Versioned API Change: Service v1 core](#versioned-api-change-service-v1-core)
            * [Internal Representation:](#internal-representation)
            * [Changes to the Service IP Allocator](#changes-to-the-service-ip-allocator)
            * [kube-apiserver Startup Configuration for Multiple Service CIDRs](#kube-apiserver-startup-configuration-for-multiple-service-cidrs)
            * [controller-manager Startup Configuration for Multiple Service CIDRs](#controller-manager-startup-configuration-for-multiple-service-cidrs)
            * ['kubectl get service' Command Display for Multiple Service IPs](#kubectl-get-service-command-display-for-multiple-service-ips)
         * [Endpoints](#endpoints)
            * ['kubectl get endpoints' Command Display for Dual-Stack Backend Pods](#kubectl-get-endpoints-command-display-for-dual-stack-backend-pods)
         * [kube-proxy Operation](#kube-proxy-operation)
            * [Kube-Proxy Startup Configuration Changes](#kube-proxy-startup-configuration-changes)
               * [Multiple bind addresses configuration](#multiple-bind-addresses-configuration)
               * [Multiple cluster CIDRs configuration](#multiple-cluster-cidrs-configuration)
         * [IPVS Support and Operation](#ipvs-support-and-operation)
         * [Support of Health/Liveness/Readiness Probes](#support-of-healthlivenessreadiness-probes)
         * [Kube-DNS/CoreDNS Operation](#kube-dnscoredns-operation)
         * [Ingress Controller Operation](#ingress-controller-operation)
            * [GCE Ingress Controller: Does Not Support External, Dual-Stack Operation](#gce-ingress-controller-does-not-support-external-dual-stack-operation)
            * [NGINX Ingress Controller - Dual-Stack Support for Bare Metal Clusters](#nginx-ingress-controller---dual-stack-support-for-bare-metal-clusters)
         * [Load Balancer Operation ???](#load-balancer-operation-)
         * [Network Policy Considerations ???](#network-policy-considerations-)
         * [Cloud Provider Plugins Considerations ???](#cloud-provider-plugins-considerations-)
         * [Container Environment Variables](#container-environment-variables)
         * [Kubeadm Support](#kubeadm-support)
         * [Spf13/pflag](#vendorgithubcomspf13pflag)
         * [End-to-End Test Support](#end-to-end-test-support)
         * [User Stories [optional]](#user-stories-optional)
            * [Story 1](#story-1)
            * [Story 2](#story-2)
         * [Implementation Details/Notes/Constraints [optional]](#implementation-detailsnotesconstraints-optional)
         * [Risks and Mitigations](#risks-and-mitigations)
      * [Graduation Criteria](#graduation-criteria)
      * [Implementation History](#implementation-history)
      * [Drawbacks [optional]](#drawbacks-optional)
      * [Alternatives [optional]](#alternatives-optional)
         * [Dual Stack at the Edge &lt;more info TBD&gt;](#dual-stack-at-the-edge-more-info-tbd)

## Summary

This proposal adds full IPv4/IPv6 dual stack functionality to Kubernetes clusters. This includes the following concepts:
- Awareness of multiple IPv4/IPv6 address assignments per pod
- Support for multiple service IPv4/IPv6 assignments per service
- Native IPv4-to-IPv4 in parallel with IPv6-to-IPv6 communications to, from, and within a cluster

## Motivation

The adoption of IPv6 has increased in recent years, and customers are requesting IPv6 support in Kubernetes clusters. To this end, the support of IPv6-only clusters was added as an alpha feature in Kubernetes Version 1.9. Clusters can now be run in either IPv4-only, IPv6-only, or in a very limited dual-stack configuration. This limited dual-stack support might be called "headless", in the sense that a CNI network plugin can configure dual-stack addresses on a pod, but Kubernetes remains aware of only a single IP address per pod.  This "headless" dual-stack support is limited by the following restrictions:
- Some CNI plugins are capable of assigning dual-stack addresses on a pod, but Kubernetes is aware of only one address per pod.
- Service IPs are either all IPv4 or all IPv6 in a cluster.
- Kubernetes system pods (api server, controller manager, etc.) can have only one IP address per pod, and addresses are either all IPv4 or all IPv6.
- Endpoints for services are either all IPv4 or all IPv6.
- Kube-dns is capable of running dual-stack, but it is currently only made aware of either all IPv4 or all IPv6 service addresses.

For scenarios that require legacy IPv4-only clients or services, the above restrictions mean that complex and expensive IPv4/IPv6 transition mechanisms (e.g. NAT64/DNS64, stateless NAT46, or SIIT/MAP) will need to be implemented in the data center networking.

An alternative is to allow for IPv4 and IPv6 pods and services to be supported in parallel throughout the cluster (a.k.a. full dual stack). This proposal aims to add full dual-stack access to Kubernetes clusters, providing native IPv4-to-IPv4 communication and native IPv6-to-IPv6 communication to, from and within a Kubernetes cluster.

### Goals

- Pod Connectivity: IPv4-to-IPv4 and IPv6-to-IPv6 access between pods
- Service Access: IPv4-to-IPv4 and IPv6-to-IPv6 access from pods to Kubernetes services
- External Server Access: IPv4-to-IPv4 and IPv6-to-IPv6 access from pods to external servers
- Ingress Access: Access from IPv4 and/or IPv6 clients to Kubernetes services. Depending on the ingress controller used, connections may cross IP families (IPv4 clients to IPv6 services, or vice versa).
- Service Discovery: Support for both A and AAAA records for kube-DNS and coreDNS
- Functionality tested with the Bridge CNI plugin and Host-Local IPAM plugin as references
- Maintain backwards-compatible support for IPv4-only and IPv6-only clusters

### Non-Goals

- Cross-family connectivity: IPv4-to-IPv6 and IPv6-to-IPv4 connectivity is considered outside of the scope of this proposal, except in the case of ingress access (depending on ingress controller used).
- CNI network plugins other than the Bridge and Host-Local IPAM plugins should support Kubernetes dual stack, but the development and testing of dual stack support for these other plugins is considered outside of the scope of this proposal.
- Code changes should support any number IPv4 addresses and any number of IPv6 addresses per pod (up to some reasonable limit). However, testing will be done only up to the following limits:
  - Pod addresses: 1 IPv4 address and 2 IPv6 addresses per pod maximum
  - Service addresses: 1 IPv4 and 1 IPv6 service addresses per service maximum
- For simplicity, for the first release of dual-stack support, the [Kubernetes Endpoints API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#endpoints-v1-core) will not be modified. Each IPv4/IPv6 address that is assigned to a backend pod will be treated and displayed as a separate endpoint for the corresponding service. Service load balancing (e.g. using kube-proxy) will be performed on a per-address basis, rather than being done on a per-dual-stack-endpoint basis.

## Proposal

In order to support dual-stack in Kubernetes clusters, Kubernetes needs to have awareness of and support dual-stack addresses for pods, nodes and services. To make this happen, the following changes will be required:

- Kubernetes needs to be made aware of multiple IPs per pod (tested with up to one IPv4 and up to two IPv6 addresses).
- Link Local Addresses (LLAs) on a pod will remain implicit (Kubernetes will not display nor track these addresses).
- Kubernetes needs to be configurable for up to two service CIDRs.
- Backend pods for a service can be dual stack. For the first release of dual-stack support, each IPv4/IPv6 address of a backend pod will be treated as a separate Kubernetes endpoint.
- Kube-proxy needs to support IPv4 and IPv6 services in parallel (e.g. drive iptables and ip6tables in parallel).
- Health/liveness/readiness probes should work for dual-stack pods. For the first release of dual-stack support, no additional configuration "knobs" will be added for probe definitions, and a probe is deemed successful if either an IPv4 or IPv6 response is received
- Kubectl commands and output displays will need to be modified for dual-stack.
- Kubeadm support will need to be added to enable spin-up of dual-stack clusters.
- New e2e test cases will need to be added to test parallel IPv4/IPv6 connectivity between pods, nodes, and services.

### Awareness of Multiple IPs per Pod

Since Kubernetes Version 1.9, Kubernetes users have had the capability to use dual-stack-capable CNI network plugins (e.g. Bridge + Host Local, Calico, etc.), using the 
[0.3.1 version of the CNI Networking Plugin API](https://github.com/containernetworking/cni/blob/spec-v0.3.1/SPEC.md), to configure multiple IPv4/IPv6 addresses on pods. However, Kubernetes currently captures and uses only the first IP address in the list of assigned pod IPs that a CNI plugin returns to Kubelet in the [CNI Results structure](https://github.com/containernetworking/cni/blob/spec-v0.3.1/SPEC.md#result).

This proposal aims to extend the Kubernetes Pod Status API so that Kubernetes can track and make use of multiple IPv4/IPv6 address assignments per pod.

#### Versioned API Change: PodStatus v1 core
In order to maintain backwards compatibility for the core V1 API, this proposal adds a new "ExtraPodIPs" field to the core V1 version of the [PodStatus V1 core API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#podstatus-v1-core), while maintaining the existing "PodIP" field. This change is in accordance with the [Kubernetes API change
quidelines](https://github.com/kubernetes/community/blob/master/contributors/devel/api_changes.md):
```
    // Default IP address allocated to the pod. Routable at least within the
    // cluster. Empty if not yet allocated.
    // +optional
    PodIP string `json:"podIP,omitempty" protobuf:"bytes,6,opt,name=podIP"`

    // Additional IP addresses allocated to the pod. Routable at least within
    // the cluster. Empty if not yet allocated.
    // +optional
    ExtraPodIPs []string `json:"podIP,omitempty" protobuf:"bytes,6,opt,name=ExtrapodIPs"`
```

Older servers and clients that were built before the introduction of full dual stack will only be aware of and make use of the original, singular "PodIP" field. It is therefore considered to be the default IP address for the pod. This choice of which pod IP to choose as the default IP address is discussed in the following subsection.

##### Default Pod IP Selection
When the internal representation of the PodStatus (see the next section) is converted to the "v1 core" versioned API object (see the [Operational Overview section](https://github.com/kubernetes/community/blob/master/contributors/devel/api_changes.md#operational-overview) of the API changes guidelines), a choice needs to be made as to which IP address from the "PodIPs" slice needs to be populated into the singular "PodIP" field. As described earlier, older servers and clients will only be aware of this singular field, so it is considered to be the default IP address for the pod. The following algorithm will be used to select a default pod IP address:
- Use the first IPv4 address found in the "ips" list returned from the CNI plugin, if any.
- If there are no IPv4 addresses, then use the first IPv6 address found in the "ips" list.

The IPv4 addresses take precedence here because:
- Less ambiguity: There should typically be only one IPv4 address assigned to a pod.
- Consistency with current node IP selection, where IPv4 addresses on the host take precedence.

It's not expected that we'll need any configuration "knobs" to allow users to tweak this algorithm: Any users who care about the choice of default IP address (i.e. anyone who is using older servers and clients in their cluster) should simply configure their CNI plugins to only use a single IP family (IPv4-only or IPv6-only). With this configuration, there should only be one pod IP address to choose as the default.

#### PodStatus Internal Representation:
The PodStatus internal representation will be modified to use a slice of IPs ("PodIPs") rather than a singular IP ("PodIP"):
```
    // IP addresses allocated to the pod. Routable at least within the cluster.
    // Empty if not yet allocated.
    // +optional
    PodIPs []string `json:"podIP,omitempty" protobuf:"bytes,6,opt,name=podIP"`
```
This "PodIPs" slice representation should eventually become part of a versioned API (after a period of deprecation for the current versioned APIs).

#### kubelet Startup Configuration for Multiple Pod CIDRs
A new, plural "pod-cidrs" option for the [kubelet startup configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) is proposed, in addition to retaining the existing, singular "pod-cidr" option (for backwards compatibility):
```
  --pod-cidr   string        [Singular IP CIDR,  Default: ""]
  --pod-cidrs  stringSlice   [Multiple IP CIDRs, comma separated list of CIDRs, Default: []]
```
The singular --pod-cidr argument will become deprecated.

#### kube-proxy Startup Configuration for Multiple Pod CIDRs
A new, plural "cluster-cidrs" option for the [kube-proxy startup configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) is proposed, in addition to retaining the existing, singular "cluster-cidr" option (for backwards compatibility):
```
  --cluster-cidr   string        [Singular IP CIDR,  Default: ""]
  --cluster-cidrs  stringSlice   [Multiple IP CIDRs, comma separated list of CIDRs, Default: []]
```
The singular --cluster-cidr argument will become deprecated.

### Container Networking Interface (CNI) Plugin Considerations

- This feature requires the use of CNI API 0.3.1 or later.
- No changes to the CNI API specification (version 0.3.1 or later) are required for Kubernetes dual stack support.
- A dual-stack capable CNI plugin should return pod IP assignments to kubelet via the "ips" array defined in the [CNI plugin result specification](https://github.com/containernetworking/cni/blob/master/SPEC.md#result).
- It's expected that plugins will typically not assign more than one IPv4 address per pod.
- The order of assigned IP addresses that are returned in the CNI plugin result for pod creation will be used to determine the "Default Pod IP" as described in the previous section.

### Support for Multiple Service IPs per Service

This proposal adds the capability to define multiple service CIDRs for a cluster.

#### First Release of Dual Stack: No Service Address Configuration "Knobs"
For the first release of Kubernetes dual-stack support, no new configuration "knobs" will be added for service definitions. This greatly simplifies the design and implementation, but requires imposing the following behavior:
- Service IP allocation: Kubernetes will always allocate a service IP from each service CIDR for each service that is created. (In the future, we might want to consider adding configuration options to allow a user to select e.g. whether a given service should be assigned only IPv4, only IPv6, or both IPv4 and IPv6 service IPs.)
- Service mappings: Kubernetes will make the following service mappings (e.g. via Kube-Proxy) between service IPs and backend pod IPs:
  - Each IPv4 service IP will be mapped to all IPv4 addresses on backend pods
  - Each IPv6 service IP will be mapped to all IPv6 addresses on backend pods

(Note that this assumed behavior works very well for the typical dual-stack case where there are one IPv4 and one IPv6 service CIDR, and one IPv4 and one IPv6 address per pod.)

#### Versioned API Change: Service v1 core
In order to maintain backwards compatibility for the core V1 API, this proposal adds a new "ExtraClusterIPs" field to the core V1 version of the [Service V1 core API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#service-v1-core), while maintaining the existing "ClusterIP" field. This change is in accordance with the [Kubernetes API change
quidelines](https://github.com/kubernetes/community/blob/master/contributors/devel/api_changes.md):

```
    // clusterIP is the IP address of the service and is usually assigned
    // randomly by the master.
    ClusterIP string `json:"clusterIP,omitempty" protobuf:"bytes,3,opt,name=clusterIP"`

    // Additional cluster IPs
    ExtraClusterIPs []string `json:"extraclusterIP,omitempty" protobuf:"bytes,3,opt,name=ExtraclusterIPs"`
```

#### Internal Representation:
The internal representation for the service IP (a.k.a. cluster IP) will be changed from a singular string to a slice of strings, and renamed "ClusterIPs":
```
    ClusterIPs []string `json:"clusterIPs,omitempty" protobuf:"bytes,3,opt,name=clusterIPs"`
```

#### Changes to the Service IP Allocator
- The service allocator will need to have one or more prefixes specified for it to allocate service IPs
- The service allocator will allocate and track service IPs for each prefix

#### kube-apiserver Startup Configuration for Multiple Service CIDRs
A new, plural "service-cluster-ip-ranges" option for the [kube-apiserver startup configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) is proposed, in addition to retaining the existing, singular "service-cluster-ip-range" option (for backwards compaibility):
```
  --service-cluster-ip-range  ipNet         [Singular IP CIDR,  Default: 10.0.0.0/24]
  --service-cluster-ip-ranges ipNetSlice    [Multiple IP CIDRs, comma separated list of CIDRs, Default: []]
```
The singular --service-cluster-ip-range argument will become deprecated.

#### controller-manager Startup Configuration for Multiple Service CIDRs
A new, plural "service-cluster-ip-ranges" option for the [controller-manager startup configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/) is proposed, in addition to retaining the existing, singular "service-cluster-ip-range" option (for backwards compaibility):
```
  --service-cluster-ip-range  ipNet         [Singular IP CIDR,  Default: 10.0.0.0/24]
  --service-cluster-ip-ranges ipNetSlice    [Multiple IP CIDRs, comma separated list of CIDRs, Default: []]
```
The singular --service-cluster-ip-range argument will become deprecated.

#### 'kubectl get service' Command Display for Multiple Service IPs
The 'kubectl get service ...' command display will need to be modified to return multiple service IPs, e.g.:
```
       kube-master# kubectl get service
       NAME         TYPE        CLUSTER-IP               EXTERNAL-IP   PORT(S)   AGE
       kubernetes   ClusterIP   fd00:1234::1,10.96.0.1   <none>        443/TCP   20m
       kube-master#
```

### Endpoints

For the first release of dual-stack support, the [Kubernetes Endpoints API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#endpoints-v1-core) will not be modified. The addresses that are included in the "Subsets" field of an "Endpoints" object will continue to be a list of separate addresses, rather than changing this to a list of dual-stack N-tuples of addresses.

Behaviorly, this means that each IPv4/IPv6 address that is assigned to a backend pod will be treated as a separate endpoint for the corresponding service. Service load balancing (e.g. using kube-proxy) will be performed on a per-address basis, rather than being done on a per-dual-stack-endpoint basis.

#### 'kubectl get endpoints' Command Display for Dual-Stack Backend Pods
There should be no changes required for the 'kubectl get endpoints ...' command display. The command output should include both IPv4 and IPv6 addresses as individual endpoints, e.g.:
```
       kube-master# kubectl get endpoints
       NAME         ENDPOINTS                         AGE
       kubernetes   [fd00:90::2]:6443,10.0.0.2:6643   20m
       kube-master#
```

### kube-proxy Operation

In order to support dual-stack Kubernetes services, kube-proxy will be modified to support parallel support for iptables and ip6tables mappings. This will require the implementation of a second "proxier" interface in the Kube-Proxy server in order to modify and track changes to iptables and ip6tables in parallel.

#### Kube-Proxy Startup Configuration Changes

##### Multiple bind addresses configuration
New [kube-proxy configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) arguments will be added to allow a user to specify separate IPv4 and IPv6 bind addresses for the kube-proxy server:
```
- --bind-address string     [Existing argument, currently used for IPv4 OR IPv6, will now be IPv4-only. Default: 0.0.0.0]
- --bind-address-v6 string  [New argument. Default: '::']
- --healthz-bind-address string     [Existing argument, currently used for IPv4 OR IPv6, will now be IPv4-only. Default: 0.0.0.0]
- --healthz-bind-address-v6 string  [New argument. Default: '::']
- --metrics-bind-address string     [Existing argument, currently used for IPv4 OR IPv6, will now be IPv4-only. Default: 0.0.0.0]
- --metrics-bind-address-v6 string  [New argument. Default: '::']
```

##### Multiple cluster CIDRs configuration
A new [kube-proxy configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) argument will be added to allow a user to specify multiple cluster CIDRs.
- --cluster-cidr  string       [Existing argument. Retained for backwords compatibility. Can be an IPv4 or IPv6 CIDR]
- --cluster-cidrs stringSlice  [New argument. Comma-separated list of CIDRs]
If both arguments are specified, the CIDRs from both will be combined.

### IPVS Support and Operation

\<TBD\>

### Support of Health/Liveness/Readiness Probes

Currently, health, liveness, and readiness probes are defined without any concern for IP addresses or families. For the first release of dual-stack support, no configuration "knobs" will be added for probe definitions. A probe for a dual-stack pod will be deemed successful if either an IPv4 or IPv6 response is received. (QUESTION: Does the current probe implementation include DNS lookups, or are IP addresses hard coded?)

In the future, we may want to consider adding configuration support to allow a user e.g. to specify whether health probes require:
- IPv4 connectivity
- IPv6 connectivity
- IPv4 or IPv6 connectivity
- IPv4 and IPv6 connectivity

Or we could offer even more control, e.g. allowing the user to select which particular prefixes should be considered when determining which IPs need to be included for health probes. If we were to offer this capability, we would want to avoid configuring the prefixes directly in the probe definition. Having explicit prefixes in the probe definition would limit its portability from cluster to cluster (there would be a dependence on prefixes that might not exist in other clusters). Instead, the probe definition would need to refer to a prefix indirectly through an abstract tag or label.

### Kube-DNS/CoreDNS Operation

kube-DNS and coreDNS services should be able to support dual-stack services and endpoints with very little change, if any. What is required is that the DNS servers are capable of presenting all IPv4 and IPv6 backend pod addresses as A and AAAA records, respectively, when resolving Kubernetes services.

### Ingress Controller Operation

The [Kubernetes ingress feature](https://kubernetes.io/docs/concepts/services-networking/ingress/) relies on the use of an ingress controller. The two "reference" ingress controllers that are considered here are the [GCE ingress controller](https://github.com/kubernetes/ingress-gce/blob/master/README.md#glbc) and the [NGINX ingress controller](https://github.com/kubernetes/ingress-nginx/blob/master/README.md#nginx-ingress-controller).

#### GCE Ingress Controller: Does Not Support External, Dual-Stack Operation
It appears that [GCE ingress controller](https://github.com/kubernetes/ingress-gce/blob/master/README.md#glbc) support is limited to IPv4-only access (QUESTION: Is this true???) from outside of the cluster. Therefore, GCE ingress controllers will not be capable of supporting external, dual-stack access to services running in a dual-stack Kubernetes cluster.

#### NGINX Ingress Controller - Dual-Stack Support for Bare Metal Clusters
The [NGINX ingress controller](https://github.com/kubernetes/ingress-nginx/blob/master/README.md#nginx-ingress-controller) should provide dual-stack external access to Kubernetes services that are hosted on baremetal clusters, with little or no changes.

- Dual-stack external access to NGINX ingress controllers is not supported with GCE/GKE or AWS cloud platforms.
- NGINX ingress controller needs to be run on a pod with dual-stack external access (needs to be on host network, or can be run directly on a pod network???)
- On the load balancer (internal) side of the NGINX ingress controller, dual-stack access to Kubernetes service backends requires the resolver to be configured with 'ipv6=on' (which is default, see the ["Configuring HTTP Load Balancing Using DNS" section of the NGINX ingress controller admin guide](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/#configuring-http-load-balancing-using-dns).
- On the load balancer (internal) side of the NGINX ingress controller, the controller will load balance to backend service pods on a per-address basis, rather than load balancing on a per-dual-stack-endpoint basis. For example, if a given backend pod has both an IPv4 and an IPv6 address, the ingress controller will treat the IPv4 and IPv6 address endpoints as separate load-balance targets. (Changing this behavior would require upstream changes to the NGINX ingress controller.)
- Ingress access can cross IP families. For example, an incoming L7 request that is received via IPv4 can be load balanced to an IPv6 endpoint address in the cluster, and vice versa. 

### Load Balancer Operation ???

### Network Policy Considerations ???

### Cloud Provider Plugins Considerations ???

### Container Environment Variables
The [container environmental variables](https://kubernetes.io/docs/concepts/containers/container-environment-variables/#container-environment) should support dual stack.


 The Downward API [status.podIP](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/#capabilities-of-the-downward-api)
 will contain a single IP address, and should be the same family as the default family.
 A new variable named status.podIPs will contain a comma separated list of IP addresses.
 The new pod API will have a slice of structures for the additional IP addresses,
 Kubelet will translate the pod structs and return podIPs as a comma delimited string.

In the pod definition yaml file

```
   - name: MY_POD_IPS
      valueFrom:
        fieldRef:
          fieldPath: status.podIPs
```
Creates the environmental variable in the pod

```
 MY_POD_IPS=fd00:10:20:0:3::3,10.20.3.3
```


### Kubeadm Support
Kubeadm should support dual stack.

- The Kubeadm config options and config file will support dual stack options
for apiserver-advertise-address, and podSubnet.

The following kubeadm config options will be updated for dual stack.

```
api:
  advertiseAddress: "fd00:90::2,10.90.0.2" [Multiple IP CIDRs, comma separated list of CIDRs]
networking:
  podSubnet: "fd00:10:20::/72,10.20.0.0/16" [Multiple IP CIDRs, comma separated list of CIDRs]
  serviceSubnet: "fd00:1234::/110,10.1.0.0/16" [Multiple IP CIDRs, comma separated list of CIDRs]
```


The following manifests will be updated by Kubeadm
kube-apiserver.yaml

```
spec:
  containers:
  - command:
    - kube-apiserver
    - --service-cluster-ip-range=fd00:1234::/110,10.96.0.0/12

```

kube-controller-manager.yaml

```
spec:
  containers:
  - command:
    - kube-controller-manager
    - --cluster-cidr=fd00:10:20::/72,10.20.0.0/16
```

### vendor/github.com/spf13/pflag
IPNetSlice will be added to spf13.pflag to allow parsing of comma separated CIDRs.

[https://github.com/spf13/pflag/pull/170](https://github.com/spf13/pflag/pull/170)

### End-to-End Test Support
\<TBD\>

### Risks and Mitigations
\<TBD\>

## Graduation Criteria
\<TBD\>

## Implementation History
\<TBD\>

## Drawbacks [optional]
\<TBD\>

## Alternatives [optional]

### Dual Stack at the Edge \<more info TBD\>

