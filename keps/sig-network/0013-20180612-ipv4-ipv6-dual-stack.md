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
            * [PodStatus Internal Representation](#podstatus-internal-representation)
            * [kubelet Startup Configuration for Dual-Stack Pod CIDRs](#kubelet-startup-configuration-for-dual-stack-pod-cidrs)
            * [kube-proxy Startup Configuration for Dual-Stack Pod CIDRs](#kube-proxy-startup-configuration-for-dual-stack-pod-cidrs)
            * ['kubectl get pods -o wide' Command Display for Dual-Stack Pod Addresses](#kubectl-get-pods--o-wide-command-display-for-dual-stack-pod-addresses)
            * ['kubectl describe pod ...' Command Display for Dual-Stack Pod Addresses](#kubectl-describe-pod--command-display-for-dual-stack-pod-addresses)
         * [Container Networking Interface (CNI) Plugin Considerations](#container-networking-interface-cni-plugin-considerations)
         * [Configuration of IP Family in Service Definitions](#configuration-of-ip-family-in-service-definitions)
         * [Endpoints](#endpoints)
            * ['kubectl get endpoints' Command Display for Dual-Stack Backend Pods](#kubectl-get-endpoints-command-display-for-dual-stack-backend-pods)
            * ['kubectl describe service' Command Display for Dual-Stack Backend Pods](#kubectl-describe-service-command-display-for-dual-stack-backend-pods)
         * [kube-proxy Operation](#kube-proxy-operation)
            * [Kube-Proxy Startup Configuration Changes](#kube-proxy-startup-configuration-changes)
               * [Multiple bind addresses configuration](#multiple-bind-addresses-configuration)
               * [Multiple cluster CIDRs configuration](#multiple-cluster-cidrs-configuration)
         * [IPVS Support and Operation](#ipvs-support-and-operation)
         * [Health/Liveness/Readiness Probes for Dual-Stack Pods](#healthlivenessreadiness-probes-for-dual-stack-pods)
         * [CoreDNS Operation](#coredns-operation)
         * [Ingress Controller Operation](#ingress-controller-operation)
            * [GCE Ingress Controller: Out-of-Scope, Testing Deferred For Now](#gce-ingress-controller-out-of-scope-testing-deferred-for-now)
            * [NGINX Ingress Controller - Dual-Stack Support for Bare Metal Clusters](#nginx-ingress-controller---dual-stack-support-for-bare-metal-clusters)
         * [Load Balancer Operation](#load-balancer-operation)
         * [Cloud Provider Plugins Considerations](#cloud-provider-plugins-considerations)
         * [Container Environment Variables](#container-environment-variables)
         * [Kubeadm Support](#kubeadm-support)
            * [Kubeadm Configuration Options](#kubeadm-configuration-options)
            * [Kubeadm-Generated Manifests](#kubeadm-generated-manifests)
         * [Spf13/pflag](#vendorgithubcomspf13pflag)
         * [End-to-End Test Support](#end-to-end-test-support)
         * [User Stories](#user-stories)
         * [Risks and Mitigations](#risks-and-mitigations)
      * [Graduation Criteria](#graduation-criteria)
      * [Implementation History](#implementation-history)
      * [Alternatives](#alternatives)
         * [Dual Stack at the Edge](#dual-stack-at-the-edge)
         * [Variation: Dual-Stack Service CIDRs (a.k.a. Full Dual Stack)](#variation-dual-stack-service-cidrs-aka-full-dual-stack)
            * [Benefits](#benefits)
            * [Changes Required](#changes-required)

## Summary

This proposal adds IPv4/IPv6 dual stack functionality to Kubernetes clusters. This includes the following concepts:
- Awareness of multiple IPv4/IPv6 address assignments per pod
- Native IPv4-to-IPv4 in parallel with IPv6-to-IPv6 communications to, from, and within a cluster

## Motivation

The adoption of IPv6 has increased in recent years, and customers are requesting IPv6 support in Kubernetes clusters. To this end, the support of IPv6-only clusters was added as an alpha feature in Kubernetes Version 1.9. Clusters can now be run in either IPv4-only, IPv6-only, or in a very limited dual-stack configuration. This limited dual-stack support might be called "headless", in the sense that a CNI network plugin can configure dual-stack addresses on a pod, but Kubernetes remains aware of only a single IP address per pod.  This "headless" dual-stack support is limited by the following restrictions:
- Some CNI plugins are capable of assigning dual-stack addresses on a pod, but Kubernetes is aware of only one address per pod.
- Kubernetes system pods (api server, controller manager, etc.) can have only one IP address per pod, and addresses are either all IPv4 or all IPv6.
- Endpoints for services are either all IPv4 or all IPv6 within a cluster.
- Service IPs are either all IPv4 or all IPv6 within a cluster.

For scenarios that require legacy IPv4-only clients or services (either internal or external to the cluster), the above restrictions mean that complex and expensive IPv4/IPv6 transition mechanisms (e.g. NAT64/DNS64, stateless NAT46, or SIIT/MAP) will need to be implemented in the data center networking.

One alternative to adding transition mechanisms would be to modify Kubernetes to provide support for IPv4 and IPv6 communications in parallel, for both pods and services, throughout the cluster (a.k.a. "full" dual stack).

A second, simpler alternative, which is a variation to the "full" dual stack model, would be to provide dual stack addresses for pods and nodes, but restrict service IPs to be single-family (i.e. allocated from a single service CIDR). In this case, service IPs in a cluster would be either all IPv4 or all IPv6, as they are now. Compared to a full dual-stack approach, this "dual-stack pods / single-family services" approach saves on implementation complexity, but would introduce some minor feature restrictions. (For more details on these tradeoffs, please refer to the "Variation: Dual-Stack Service CIDRs" section under "Alternatives" below).

This proposal aims to add "dual-stack pods / single-family services" support to Kubernetes clusters, providing native IPv4-to-IPv4 communication and native IPv6-to-IPv6 communication to, from and within a Kubernetes cluster.

### Goals

- Pod Connectivity: IPv4-to-IPv4 and IPv6-to-IPv6 access between pods
- External Server Access: IPv4-to-IPv4 and IPv6-to-IPv6 access from pods to external servers
- Ingress Access: Access from IPv4 and/or IPv6 clients to Kubernetes services. Depending on the ingress controller used, connections may cross IP families (IPv4 clients to IPv6 services, or vice versa).
- Functionality tested with the Bridge CNI plugin, PTP CNI plugin, and Host-Local IPAM plugins as references
- Maintain backwards-compatible support for IPv4-only and IPv6-only clusters

### Non-Goals

- Service CIDRs: Dual-stack service CIDRs will not be supported for this proposal. Service access within a cluster will be done via all IPv4 service IPs or all IPv6 service IPs.
- Single-Family Applications: There may be some some clients or applications that only work with (bind to) IPv4 or or only work with (bind to) IPv6. A cluster can support either IPv4-only applications or IPv6-only applications (not both), depending upon the cluster CIDR's IP family. For example, if a cluster uses an IPv6 service CIDR, then IPv6-only applications will work fine, but IPv4-only applications in that cluster will not have IPv4 service IPs (and corresponding DNS A records) with which to access Kubernetes services. If a cluster needs to support legacy IPv4-only applications, but not IPv6-only applications, then the cluster should be configured with an IPv4 service CIDR.
- Cross-family connectivity: IPv4-to-IPv6 and IPv6-to-IPv4 connectivity is considered outside of the scope of this proposal, except in the case of ingress access, where cross-family access may be supported depending upon the chosen ingress controller.
- CNI network plugins: Plugins other than the Bridge, PTP, and Host-Local IPAM plugins should support Kubernetes dual stack, but the development and testing of dual stack support for these other plugins is considered outside of the scope of this proposal.
- Multiple IPs vs. Dual-Stack: Code changes will be done in a way to facilitate future expansion to more general multiple-IPs-per-pod and multiple-IPs-per-node support. However, this initial release will impose "dual-stack-centric" IP address limits as follows:
  - Pod addresses: 1 IPv4 address and 1 IPv6 addresses per pod maximum
  - Node addresses: 1 IPv4 address and 1 IPv6 addresses per pod maximum
  - Service addresses: 1 IP address pod service
- For simplicity, the [Kubernetes Endpoints API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#endpoints-v1-core) will not be modified. Each IPv4/IPv6 address that is assigned to a backend pod will be treated and displayed as a separate endpoint for the corresponding service. Service load balancing (e.g. using kube-proxy) will be performed on a per-address basis, rather than being done on a per-dual-stack-endpoint basis.
- Kube-DNS is expected to be End-of-Life soon, so dual-stack testing will be performed using coreDNS.
- External load balancers that rely on Kubernetes services for load balancing functionality will only work with the IP family that matches the IP family of the cluster's service CIDR.
- Dual-stack support for Kubernetes orchestration tools other than kubeadm (e.g. miniKube, KubeSpray, etc.) are considered outside of the scope of this proposal.

## Proposal

In order to support dual-stack in Kubernetes clusters, Kubernetes needs to have awareness of and support dual-stack addresses for pods and nodes. Here is a summary of the proposal (details follow in subsequent sections):

- Kubernetes needs to be made aware of multiple IPs per pod (limited to one IPv4 and one IPv6 address per pod maximum).
- Link Local Addresses (LLAs) on a pod will remain implicit (Kubernetes will not display nor track these addresses).
- Backend pods for a service can be dual stack. For the first release of dual-stack support, each IPv4/IPv6 address of a backend pod will be treated as a separate Kubernetes endpoint.
- Kube-proxy needs to drive iptables and ip6tables in parallel. This is required in order to allow exposing services via both IPv4 and IPv6, e.g. using Kubernetes:
  - NodePort
  - ExternalIPs
- "Nice to Have" or stretch goal: IPVS proxying needs to support IPv4 and IPv6 services in parallel. However, this support depends upon resolution of [cloudnativelabs/kube-router Issue #307](https://github.com/cloudnativelabs/kube-router/issues/307).
- For health/liveness/readiness probe support, a kubelet configuration will be added to allow a cluster administrator to select a preferred IP family to use for implementing probes on dual-stack pods.
- The pod status API changes will include a per-IP string map for arbitrary annotations, as a placeholder for future Kubernetes enhancements. This mapping is not required for this dual-stack design, but will allow future annotations, e.g. allowing a CNI network plugin to indicate to which network a given IP address applies.
- Kubectl commands and output displays will need to be modified for dual-stack.
- Kubeadm support will need to be added to enable spin-up of dual-stack clusters. Kubeadm support is required for implementing dual-stack continuous integration (CI) tests.
- New e2e test cases will need to be added to test parallel IPv4/IPv6 connectivity between pods, nodes, and services.

### Awareness of Multiple IPs per Pod

Since Kubernetes Version 1.9, Kubernetes users have had the capability to use dual-stack-capable CNI network plugins (e.g. Bridge + Host Local, Calico, etc.), using the 
[0.3.1 version of the CNI Networking Plugin API](https://github.com/containernetworking/cni/blob/spec-v0.3.1/SPEC.md), to configure multiple IPv4/IPv6 addresses on pods. However, Kubernetes currently captures and uses only IP address from the pod's main interface.

This proposal aims to extend the Kubernetes Pod Status API so that Kubernetes can track and make use of up to one IPv4 address and up to one IPv6 address assignment per pod.

#### Versioned API Change: PodStatus v1 core
In order to maintain backwards compatibility for the core V1 API, this proposal retains the existing (singular) "PodIP" field in the core V1 version of the [PodStatus V1 core API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#podstatus-v1-core), and adds a new array of structures that store pod IPs along with associated metadata for that IP. The metadata for each IP (refer to the "Properties" map below) will not be used by the dual-stack feature, but is added as a placeholder for future enhancements, e.g. to allow CNI network plugins to indicate to which physical network that an IP is associated. Retaining the existing "PodIP" field for backwards compatibility is in accordance with the [Kubernetes API change quidelines](https://github.com/kubernetes/community/blob/master/contributors/devel/api_changes.md).
```
    // Default IP address allocated to the pod. Routable at least within the
    // cluster. Empty if not yet allocated.
    PodIP string `json:"podIP,omitempty" protobuf:"bytes,6,opt,name=podIP"`

    // IP address information. Each entry includes:
    //    IP: An IP address allocated to the pod. Routable at least within
    //        the cluster.
    //    Properties: Arbitrary metadata associated with the allocated IP.
    // This list is inclusive, i.e. it includes the default IP stored in the
    // "PodIP" field. It is empty if no IPs have been allocated yet.
    type PodIPInfo struct {
        IP string
        Properties map[string]string
    }

    // IP addresses allocated to the pod with associated metadata.
    PodIPs []PodIPInfo `json:"podIPs,omitempty" protobuf:"bytes,6,opt,name=podIPs"`
```

##### Default Pod IP Selection
Older servers and clients that were built before the introduction of full dual stack will only be aware of and make use of the original, singular "PodIP" field above. It is therefore considered to be the default IP address for the pod. If there are multiple entries in the PodIPs slice (i.e. there's both an IPv4 and an IPv6 entry), then the IP family of the cluster's configured service CIDR will be used to determine which IP will be used as the default for the pod. In other words, if the service CIDR is IPv4(IPv6), then the IPv4(IPv6) entry in the slice will be used as the default.

#### PodStatus Internal Representation
The PodStatus internal representation will be modified to use a slice of PodIPInfo structs rather than a singular IP ("PodIP"):
```
    // IP address information. Each entry includes:
    //    IP: An IP address allocated to the pod. Routable at least within
    //        the cluster.
    //    Properties: Arbitrary metadata associated with the allocated IP.
    // Empty if no IPs have been allocated yet.
    type PodIPInfo struct {
        IP string
        Properties map[string]string
    }

    // IP addresses allocated to the pod with associated metadata.
    PodIPs []PodIPInfo `json:"podIPs,omitempty" protobuf:"bytes,6,opt,name=podIPs"`
```
This internal representation should eventually become part of a versioned API (after a period of deprecation for the singular "PodIP" field).

#### kubelet Startup Configuration for Dual-Stack Pod CIDRs
The existing "--pod-cidr" option for the [kubelet startup configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) will be modified to support multiple IP CIDRs in a comma-separated list (rather than a single IP string), i.e.:
```
  --pod-cidr  ipNetSlice   [IP CIDRs, comma separated list of CIDRs, Default: []]
```
Only the first address of each IP family will be used; all others will be ignored.

#### kube-proxy Startup Configuration for Dual-Stack Pod CIDRs
The existing "cluster-cidr" option for the [kube-proxy startup configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) will be modified to support multiple cluster CIDRs in a comma-separated list (rather than a single IP string), i.e:
```
  --cluster-cidr  ipNetSlice   [IP CIDRs, comma separated list of CIDRs, Default: []]
```
Only the first address of each IP family will be used; all others will be ignored.

#### 'kubectl get pods -o wide' Command Display for Dual-Stack Pod Addresses
The output for the 'kubectl get pods -o wide' command will need to be modified to display a comma-separated list of IPs for each pod, e.g.:
```
       kube-master# kubectl get pods -o wide
       NAME               READY     STATUS    RESTARTS   AGE       IP                          NODE
       nginx-controller   1/1       Running   0          20m       fd00:db8:1::2,192.168.1.3   kube-minion-1
       kube-master#
```

#### 'kubectl describe pod ...' Command Display for Dual-Stack Pod Addresses
The output for the 'kubectl describe pod ...' command will need to be modified to display a comma-separated list of IPs for each pod, e.g.:
```
       kube-master# kubectl describe pod nginx-controller
       .
       .
       .
       IPs:     fd00:db8:1::2,192.168.1.3
       .
       .
       .
```

### Container Networking Interface (CNI) Plugin Considerations

This feature requires the use of CNI API 0.3.1 or later. The dual-stack feature requires no changes to this API.

### Configuration of IP Family in Service Definitions
This proposal adds an option to configure an IP family for each port configuration within each service definition.
```
    ipFamily: <ipv4|ipv6|dual-stack>       [Default: dual-stack]
```
For example, a ports definition for an application that only binds to IPv4 might look like this:
```
  ports:
  - port: 8080
    nodePort: 30302
    targetPort: 80
    protocol: TCP
    ipFamily: ipv4
```
This per-service-port IP family configuration is required because:
- IPv4-only and IPv6-only Application Support: If there is a legacy application that only binds to IPv4 addresses (IPv4-only application) that is running in an otherwise dual-stack cluster, then we don't want IPv6 endpoints created for the service (even though a CNI plugin configured for dual stack would be assigning both IPv4 and IPv6 pod addresses to the service's backend pods). Otherwise, any IPv6 endpoints that are created would be "dead ends" (no response will be received from the server) for ingress controller and external load balancer operation. The same reasoning applies to IPv6-only applications.
- Per-Family Port Assignments: A user might want to configure a different nodePort or a different targetPort for IPv4 vs what's configured for IPv6.

If a service definition only includes ipFamily settings of "ipv4" ("ipv6"), then no IPv6 (IPv4) service IP will be assigned for that service.

If a port definition is configured with an ipFamily of "dual-stack", then both IPv4 and IPv6 endpoints will be created, but the service will only have a single service IP allocated (dependent on the cluster's configured service CIDR).

### Endpoints

For the first release of dual-stack support, the [Kubernetes Endpoints API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#endpoints-v1-core) will not be modified. The addresses that are included in the "Subsets" field of an "Endpoints" object will continue to be a list of separate addresses, rather than changing this to a list of dual-stack N-tuples of addresses.

Behaviorlly, this means that each IPv4/IPv6 address that is assigned to a backend pod will be treated as a separate endpoint for the corresponding service. Service load balancing (e.g. using kube-proxy) will be performed on a per-address basis, rather than being done on a per-dual-stack-endpoint basis.

#### 'kubectl get endpoints' Command Display for Dual-Stack Backend Pods
There should be no changes required for the 'kubectl get endpoints ...' command display. The command output should include both IPv4 and IPv6 addresses as individual endpoints, e.g.:
```
       kube-master# kubectl get endpoints
       NAME            ENDPOINTS                           AGE
       kubernetes      [fd00::100]:6443,10.0.0.2:6643      15m
       nginx-service   [fd00:db8:1::2]:80,192.168.1.3:80   20m
       kube-master#
```

#### 'kubectl describe service' Command Display for Dual-Stack Backend Pods
There should be no changes required for the 'kubectl describe service ...' command display. The command output should include all IPv4 and IPv6 addresses as individual endpoints, e.g.:
```
       kube-master# kubectl describe service nginx-service
       .
       .
       .
       Endpoints:   [fd00:db8:1::2]:80,192.168.1.3:80
       .
       .
       .
       kube-master#
```

### kube-proxy Operation

Kube-proxy will be modified to drive iptables and ip6tables in parallel. This will require the implementation of a second "proxier" interface in the Kube-Proxy server in order to modify and track changes to both tables. This is required in order to allow exposing services via both IPv4 and IPv6, e.g. using Kubernetes:
  - NodePort
  - ExternalIPs

#### Kube-Proxy Startup Configuration Changes

##### Multiple bind addresses configuration
The existing "--bind-address" option for the [kube-proxy startup configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) will be modified to support multiple IP addresses in a comma-separated list (rather than a single IP string).
```
  --bind-address  stringSlice   (IP addresses, in a comma separated list, Default: [0.0.0.0,])
```
Only the first address of each IP family will be used; all others will be ignored.

##### Multiple cluster CIDRs configuration
The existing "--cluster-cidr" option for the [kube-proxy startup configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) will be modified to support multiple IP CIDRs in a comma-separated list (rather than a single IP CIDR).
A new [kube-proxy configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) argument will be added to allow a user to specify multiple cluster CIDRs.
```
  --cluster-cidr  ipNetSlice   (IP CIDRs, in a comma separated list, Default: [])
```
Only the first CIDR for each IP family will be used; all others will be ignored.

### IPVS Support and Operation

Since IPVS functionality does not yet include IPv6 support (see [cloudnativelabs/kube-router Issue #307](https://github.com/cloudnativelabs/kube-router/issues/307)), support for IPVS functionality in a dual-stack cluster is considered a "nice-to-have" or stretch goal.

### Health/Liveness/Readiness Probes for Dual-Stack Pods

Currently, health, liveness, and readiness probes are defined without any concern for IP addresses or families. For the first release of dual-stack support, a cluster administrator will be able to select the preferred IP family to use for probes when a pod has both IPv4 and IPv6 addresses. For this selection, a new "--preferred-probe-ip-family" argument for the for the [kubelet startup configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) will be added:
```
  --preferred-probe-ip-family  string   ["ipv4" or "ipv6". Default: "ipv4"]
```
When a pod has only one IP address, that address will be used for probes regardless of the "--preferred-probe-ip-family" setting.

In the future, we may want to consider adding a "dual-stack" option for the "--preferred-probe-ip-family" argument, indicating that a kubelet should test for probes using both IPv4 and IPv6 addresses for a pod, and consider the probe successful if a response is received via either address.

### CoreDNS Operation

Because this proposal retains support for only a single Kubernetes service CIDR, CoreDNS support should work with very little change, if any.

### Ingress Controller Operation

The [Kubernetes ingress feature](https://kubernetes.io/docs/concepts/services-networking/ingress/) relies on the use of an ingress controller. The two "reference" ingress controllers that are considered here are the [GCE ingress controller](https://github.com/kubernetes/ingress-gce/blob/master/README.md#glbc) and the [NGINX ingress controller](https://github.com/kubernetes/ingress-nginx/blob/master/README.md#nginx-ingress-controller).

#### GCE Ingress Controller: Out-of-Scope, Testing Deferred For Now
It is not clear whether the [GCE ingress controller](https://github.com/kubernetes/ingress-gce/blob/master/README.md#glbc) supports external, dual-stack access. Testing of dual-stack access to Kubernetes services via a GCE ingress controller is considered out-of-scope until after the initial implementation of dual-stack support for Kubernetes.

#### NGINX Ingress Controller - Dual-Stack Support for Bare Metal Clusters
The [NGINX ingress controller](https://github.com/kubernetes/ingress-nginx/blob/master/README.md#nginx-ingress-controller) should provide dual-stack external access to Kubernetes services that are hosted on baremetal clusters, with little or no changes.

- Dual-stack external access to NGINX ingress controllers is not supported with GCE/GKE or AWS cloud platforms.
- NGINX ingress controller needs to be run on a pod with dual-stack external access.
- On the load balancer (internal) side of the NGINX ingress controller, the controller will load balance to backend service pods on a per-address basis, rather than load balancing on a per-dual-stack-endpoint basis. For example, if a given backend pod has both an IPv4 and an IPv6 address, the ingress controller will treat the IPv4 and IPv6 address endpoints as separate load-balance targets. (Changing this behavior would require upstream changes to the NGINX ingress controller.)
- Ingress access can cross IP families. For example, an incoming L7 request that is received via IPv4 can be load balanced to an IPv6 endpoint address in the cluster, and vice versa. 

### Load Balancer Operation
\<TBD\>

### Cloud Provider Plugins Considerations
\<TBD\>

### Container Environment Variables
-\<TBD\>
The [container environmental variables](https://kubernetes.io/docs/concepts/containers/container-environment-variables/#container-environment) should support dual stack.

The Downward API [status.podIP](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/#capabilities-of-the-downward-api) will preserve the existing single IP address, and will be set to the default IP for each pod. A new environmental variable named status.podIPs will contain a comma-separated list of IP addresses. The new pod API will have a slice of structures for the additional IP addresses. Kubelet will translate the pod structures and return podIPs as a comma-delimited string.

Here is an example of how to define a pluralized MY_POD_IPS environmental variable in a pod definition yaml file:
```
   - name: MY_POD_IPS
      valueFrom:
        fieldRef:
          fieldPath: status.podIPs
```

This definition will cause an environmental variable setting in the pod similar to the following:
```
 MY_POD_IPS=fd00:10:20:0:3::3,10.20.3.3
```

### Kubeadm Support
Dual-stack support will need to be added to kubeadm both for dual-stack development purposes, and for use in dual-stack continuous integration tests.

- The Kubeadm config options and config file will support dual stack options
for apiserver-advertise-address, and podSubnet.

#### Kubeadm Configuration Options
The kubeadm configuration options for advertiseAddress and podSubnet will need to be changed to handle a comma-separated list of CIDRs:
```
    api:
      advertiseAddress: "fd00:90::2,10.90.0.2" [Multiple IP CIDRs, comma separated list of CIDRs]
    networking:
      podSubnet: "fd00:10:20::/72,10.20.0.0/16" [Multiple IP CIDRs, comma separated list of CIDRs]
```

#### Kubeadm-Generated Manifests
Kubeadm will need to generate dual-stack CIDRs for the --service-cluster-ip-range command line argument in kube-apiserver.yaml:
```
    spec:
      containers:
      - command:
        - kube-apiserver
        - --service-cluster-ip-range=fd00:1234::/110,10.96.0.0/12
```

Kubeadm will also need to generate dual-stack CIDRs for the --cluster-cidr argument in kube-apiserver.yaml:
```
    spec:
      containers:
      - command:
        - kube-controller-manager
        - --cluster-cidr=fd00:10:20::/72,10.20.0.0/16
```

### vendor/github.com/spf13/pflag
This dual-stack proposal will introduce a new IPNetSlice object to spf13.pflag to allow parsing of comma separated CIDRs. Refer to [https://github.com/spf13/pflag/pull/170](https://github.com/spf13/pflag/pull/170)

### End-to-End Test Support
\<TBD\>

### User Stories
\<TBD\>

### Risks and Mitigations
\<TBD\>

## Graduation Criteria
\<TBD\>

## Implementation History
\<TBD\>

## Alternatives

### Dual Stack at the Edge
Instead of modifying Kubernetes to provide dual-stack functionality within the cluster, one alternative is to run a cluster in IPv6-only mode, and instantiate IPv4-to-IPv6 translation mechanisms at the edge of the cluster. Such an approach can be called "Dual Stack at the Edge". Since the translation mechanisms are mostly external to the cluster, very little changes (or integration) would be required to the Kubernetes cluster itself. (This may be quicker for Kubernetes users to implement than waiting for the changes proposed in this proposal to be implemented).

For example, a cluster administrator could configure a Kubernetes cluster in IPv6-only mode, and then instantiate the following external to the cluster:
- Stateful NAT64 and DNS64 servers: These would handle connections from IPv6 pods to external IPv4-only servers. The NAT64/DNS64 servers would be in the data center, but functionally external to the cluster. (Although one variation to consider would be to implement the DNS64 server inside the cluster as a CoreDNS plugin.)
- Dual-stack ingress controllers (e.g. Nginx): The ingress controller would need dual-stack access on the external side, but would load balance to IPv6-only endpoints inside the cluster.
- Stateless NAT46 servers: For access from IPv4-only, external clients to Kubernetes pods, or to exposed services (e.g. via NodePort or ExternalIPs). This may require some static configuration for IPv4-to-IPv6 mappings.

### Variation: Dual-Stack Service CIDRs (a.k.a. Full Dual Stack)

As a variation to the "Dual-Stack Pods / Single-Family Services" approach outlined above, we can consider supporting IPv4 and IPv6 service CIDRs in parallel (a.k.a. the "full" dual stack approach).

#### Benefits
Providing dual-stack service CIDRs would add the following functionality:
- Dual-Stack Pod-to-Services. Clients would have a choice of using A or AAAA DNS records when resolving Kubernetes services.
- Simultaneous support for both IPv4-only and IPv6-only applications internal to the cluster. Without dual-stack service CIDRs, a cluster can support either IPv4-only applications or IPv6-only applications, depending upon the cluster CIDR's IP family. For example, if a cluster uses an IPv6 service CIDR, then IPv4-only applications in that cluster will not have IPv4 service IPs (and corresponding DNS A records) with which to access Kubernetes services.
- External load balancers that use Kubernetes services for load balancing functionality (i.e. by mapping to service IPs) would work in dual stack mode. (Without dual-stack service CIDRs, these external load balancers would only work for the IP family that matches the cluster service CIDR's family.)

#### Changes Required
- [controller-manager startup configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/): The "--service-cluster-ip-range" startup argument would need to be modified to accept a comma-separated list of CIDRs.
- [kube-apiserver startup configuration](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/): The "--service-cluster-ip-range" would need to be modified to accept a comma-separated list of CIDRs.
- [Service V1 core API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#service-v1-core): This versioned API object would need to be modified to support multiple cluster IPs for each service. This would require, for example, the addition of an "ExtraClusterIPs" slice of strings, and the designation of one of the cluster IPs as the default cluster IP for a given service (similar to changes described above for the PodStatus v1 core API).
- The service allocator: This would need to be modified to allocate a service IP from each service CIDR for each service that is created.
- 'kubectl get service' command: The display output for this command would need to be modified to return multiple service IPs for each service.
- CoreDNS may need to be modified to loop through both (IPv4 and IPv6) service IPs for each given Kubernetes service, and advertise both IPs as A and AAAA records accordingly in DNS responses.

