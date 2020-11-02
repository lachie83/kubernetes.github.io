---
reviewers:
- lachie83
- khenidak
- aramase
- bridgetkromhout
title: IPv4/IPv6 dual-stack
feature:
  title: IPv4/IPv6 dual-stack
  description: >
    Allocation of IPv4 and IPv6 addresses to Pods and Services

content_type: concept
weight: 70
---

<!-- overview -->

{{< feature-state for_k8s_version="v1.20" state="alpha" >}}

 IPv4/IPv6 dual-stack enables the allocation of both IPv4 and IPv6 addresses to {{< glossary_tooltip text="Pods" term_id="pod" >}} and {{< glossary_tooltip text="Services" term_id="service" >}}.

If you enable IPv4/IPv6 dual-stack networking for your Kubernetes cluster, the cluster will support the simultaneous assignment of both IPv4 and IPv6 addresses.



<!-- body -->

## Supported Features

Enabling IPv4/IPv6 dual-stack on your Kubernetes cluster provides the following features:

   * Dual-stack Pod networking (a single IPv4 and IPv6 address assignment per Pod)
   * IPv4 and IPv6 enabled Services
   * Pod off-cluster egress routing (eg. the Internet) via both IPv4 and IPv6 interfaces

## Prerequisites

The following prerequisites are needed in order to utilize IPv4/IPv6 dual-stack Kubernetes clusters:

   * Kubernetes 1.20 or later (Dual-stack as alpha was re-implemented in 1.20)
   * Provider support for dual-stack networking (Cloud provider or otherwise must be able to provide Kubernetes nodes with routable IPv4/IPv6 network interfaces)
   * A network plugin that supports dual-stack (such as Kubenet or Calico)

## Enable IPv4/IPv6 dual-stack

To enable IPv4/IPv6 dual-stack, enable the `IPv6DualStack` [feature gate](/docs/reference/command-line-tools-reference/feature-gates/) for the relevant components of your cluster, and set dual-stack cluster network assignments:

   * kube-apiserver:
      * `--feature-gates="IPv6DualStack=true"`
      * `--service-cluster-ip-range=<IPv4 CIDR>,<IPv6 CIDR>`
   * kube-controller-manager:
      * `--feature-gates="IPv6DualStack=true"`
      * `--cluster-cidr=<IPv4 CIDR>,<IPv6 CIDR>`
      * `--service-cluster-ip-range=<IPv4 CIDR>,<IPv6 CIDR>`
      * `--node-cidr-mask-size-ipv4|--node-cidr-mask-size-ipv6` defaults to /24 for IPv4 and /64 for IPv6
   * kubelet:
      * `--feature-gates="IPv6DualStack=true"`
   * kube-proxy:
      * `--cluster-cidr=<IPv4 CIDR>,<IPv6 CIDR>`
      * `--feature-gates="IPv6DualStack=true"`

{{< note >}}
An example of an IPv4 CIDR: `10.244.0.0/16` (though you would supply your own address range)

An example of an IPv6 CIDR: `fdXY:IJKL:MNOP:15::/64` (this shows the format but is not a valid address - see [RFC 4193](https://tools.ietf.org/html/rfc4193))

{{< /note >}}

## Services

If your cluster has dual-stack enabled, you can create {{< glossary_tooltip text="Services" term_id="service" >}} which can use IPv4, IPv6, or both. 

Services will default to the address family of the first service cluster IP range configured  via the `--service-cluster-ip-range` flag to the kube-controller-manager.

If you want to explicitly specify which IP address is the primary IP address (which is the address assigned for the Service as `.spec.ClusterIP`), you can optionally use the `.spec.ipFamilyPolicy` field. This field accepts one of the following values:

   * `SingleStack`: Allocates the Service cluster IP from the first configured service cluster IP range
   * `PreferDualStack`: 
      * Only used if the cluster has dual-stack enabled. Allocates IPv4 and IPv6 cluster IPs for the Service
      * If the cluster does not have dual-stack enabled, this setting follows the same behavior as `SingleStack`.
   * `RequireDualStack`: Allocates Service `.spec.ClusterIPs` from both IPv4 and IPv6 address ranges 
      * Selects the `.spec.ClusterIP` from the list of `.spec.ClusterIPs` based on the address family of the first element in the `.spec.ipFamilies` array

If you would like to define which IP family to use for single stack or define the order of IP families for dual-stack, you can choose the address families by setting an optional field, `.spec.ipFamilies`, on the Service. 

{{< note >}}
The `.spec.ipFamilies` field is immutable because the `.spec.ClusterIP` cannot be reallocated on a Service that already exists. If you want to change `.spec.ipFamilies`, delete and recreate the Service.
{{< /note >}}

   * The `.spec.ipFamilies` field is an array that accepts element values of `IPv4` and `IPv6`
   * The `.spec.ipFamilies` array field accepts one or more of the following element values:
      * `IPv4`: The API server will assign an IP from a `service-cluster-ip-range` that is `ipv4`
      * `IPv6`: The API server will assign an IP from a `service-cluster-ip-range` that is `ipv6`

### Example dual-stack Service configurations

Let's look at several examples to demonstrate the behavior of various dual-stack Service configurations.

   * This Service specification does not explicitly define `.spec.ipFamilyPolicy`. Kubernetes will assign a cluster IP for the Service from the first configured `service-cluster-ip-range` and set the `.spec.ipFamilyPolicy` to `SingleStack`. [Services without selectors](/docs/concepts/services-networking/service/#services-without-selectors) and [Headless Services](/docs/concepts/services-networking/service/#headless-services) with selectors will behave in this same way.

{{< codenew file="service/networking/dual-stack-default-svc.yaml" >}}

   * This Service specification explicitly defines `PreferDualStack` in `.spec.ipFamilyPolicy`. Kubernetes will assign both IPv4 and IPv6 addresses (as this cluster has dual-stack enabled) and select the `.spec.ClusterIP` from the list of `.spec.ClusterIPs` based on the address family of the first element in the `.spec.ipFamilies` array. Specifying `RequireDualStack` in `.spec.ipFamilyPolicy` on a cluster with dual-stack enabled behaves the same way.

{{< codenew file="service/networking/dual-stack-preferred-svc.yaml" >}}

This Service specification explicitly defines `IPv6` and `IPv4` in `.spec.ipFamilies` as well as defining `PreferDualStack` in `.spec.ipFamilyPolicy`. Kubernetes will assign an IPv6 and IPv4 address in `.spec.ClusterIPs` and `.spec.ClusterIP` will be set to the IPv6 address because it is the first element in the `.spec.ClusterIPs` array. 

{{< codenew file="service/networking/dual-stack-preferred-ipfamilies-svc.yaml" >}}

### Headless Services without selector

For [Headless Services without selectors](/docs/concepts/services-networking/service/#without-selectors) and without `.spec.ipFamilyPolicy` explicitly set, the `.spec.ipFamilyPolicy` field defaults to `RequireDualStack`.

### Service Type LoadBalancer

To provision a dual-stack LoadBalancer for your Service:
   * Set the `.spec.type` field to `LoadBalancer`
   * Set `.spec.ipFamilyPolicy` field to `PreferDualStack` or `RequireDualStack`

{{< note >}}
IPv6-enabled external load balancers must be supported by your cloud provider.
{{< /note >}}

## Egress Traffic

Either publicly routable or non-publicly routable IPv6 address blocks will work if the underlying {{< glossary_tooltip text="CNI" term_id="cni" >}} provider implements the transport. If you have a Pod that uses non-publicly routable IPv6 addresses and want that Pod to reach off-cluster destinations (eg. the public Internet), you must set up IP masquerading for the egress traffic. The [ip-masq-agent](https://github.com/kubernetes-sigs/ip-masq-agent) project supports dual-stack for IP masquerading on dual-stack clusters.

## {{% heading "whatsnext" %}}


* [Validate IPv4/IPv6 dual-stack](/docs/tasks/network/validate-dual-stack) networking
