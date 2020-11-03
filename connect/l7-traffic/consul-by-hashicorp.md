# Consul by HashiCorp

**1.6.0+:** This feature is available in Consul versions 1.6.0 and newer.

This topic is part of a [low-level API](https://www.consul.io/api/discovery-chain) primarily targeted at developers building external [Connect proxy integrations](../proxies/consul-by-hashicorp-2.md).

The service discovery process can be modeled as a "discovery chain" which passes through three distinct stages: routing, splitting, and resolution. Each of these stages is controlled by a set of [configuration entries](https://www.consul.io/docs/agent/config-entries). By configuring different phases of the discovery chain a user can control how proxy upstreams are ultimately resolved to specific instances for load balancing.

**Note:** The discovery chain is currently only used to discover [Connect]() proxy upstreams.

## [»](consul-by-hashicorp.md#configuration)Configuration

The configuration entries used in the discovery chain are designed to be simple to read and modify for narrowly tailored changes, but at discovery-time the various configuration entries interact in more complex ways. For example:

* If a [`service-resolver`](https://www.consul.io/docs/agent/config-entries/service-resolver) is created with a [service redirect](https://www.consul.io/docs/agent/config-entries/service-resolver#service) defined, then all references made to the original service in any other configuration entry is replaced with the redirect destination.
* If a [`service-resolver`](https://www.consul.io/docs/agent/config-entries/service-resolver) is created with a [default subset](https://www.consul.io/docs/agent/config-entries/service-resolver#defaultsubset) defined then all references made to the original service in any other configuration entry that did not specify a subset will be replaced with the default.
* If a [`service-splitter`](https://www.consul.io/docs/agent/config-entries/service-splitter) is created with a [service split](https://www.consul.io/docs/agent/config-entries/service-splitter#splits), and the target service has its own `service-splitter` then the overall effect is flattened and only a single aggregate traffic split is ultimately configured in the proxy.
* [`service-resolver`](https://www.consul.io/docs/agent/config-entries/service-resolver) redirect loops must be rejected as invalid.
* [`service-router`](https://www.consul.io/docs/agent/config-entries/service-router) and [`service-splitter`](https://www.consul.io/docs/agent/config-entries/service-splitter) configuration entries require an L7 compatible protocol be set for the service via either a [`service-defaults`](https://www.consul.io/docs/agent/config-entries/service-defaults) or [`proxy-defaults`](https://www.consul.io/docs/agent/config-entries/proxy-defaults) config entry. Violations must be rejected as invalid.
* If an [upstream configuration](../registration/consul-by-hashicorp.md#upstream-configuration-reference) [`datacenter`](../registration/consul-by-hashicorp.md#datacenter) parameter is defined then any configuration entry that does not explicitly refer to a desired datacenter should use that value from the upstream.

## [»](consul-by-hashicorp.md#compilation)Compilation

To correctly interpret a collection of configuration entries as a valid discovery chain, we first compile them into a form more directly usable by the layers responsible for configuring Connect sidecar proxies.

You can interact with the compiler directly using the [discovery-chain API](https://www.consul.io/api/discovery-chain).

### [»](consul-by-hashicorp.md#compilation-parameters)Compilation Parameters

* **Service Name** - The service being discovered by name.
* **Datacenter** - The datacenter to use as the basis of compilation.
* **Overrides** - Discovery-time tweaks to apply when compiling. These should be derived from either the [proxy](../registration/consul-by-hashicorp.md#complete-configuration-example) or [upstream](../registration/consul-by-hashicorp.md#upstream-configuration-reference) configurations if either are set.

### [»](consul-by-hashicorp.md#compilation-results)Compilation Results

The response is a single wrapped `CompiledDiscoveryChain` field:

```text
{
    "Chain": {......}
}
```

#### [»](consul-by-hashicorp.md#compileddiscoverychain)`CompiledDiscoveryChain`

The chain encodes a digraph of [nodes](consul-by-hashicorp.md#discoverygraphnode) and [targets](consul-by-hashicorp.md#discoverytarget). Nodes are the compiled representation of various discovery chain stages and targets are instructions on how to use the [health API](https://www.consul.io/api/health#list-nodes-for-connect-capable-service) to retrieve relevant service instance lists.

You should traverse the nodes starting with [`StartNode`](consul-by-hashicorp.md#startnode). The nodes can be resolved by name using the [`Nodes`](consul-by-hashicorp.md#nodes) field. Targets can be resolved by name using the [`Targets`](consul-by-hashicorp.md#targets) field.

* [`ServiceName`](consul-by-hashicorp.md#servicename) `(string)` - The requested service.
* [`Namespace`](consul-by-hashicorp.md#namespace) `(string)` - The requested namespace.
* [`Datacenter`](consul-by-hashicorp.md#datacenter) `(string)` - The requested datacenter.
* [`CustomizationHash`](consul-by-hashicorp.md#customizationhash) `(string:)` - A unique hash of any overrides that affected the compilation of the discovery chain.

  If set, this value should be used to prefix/suffix any generated load balancer data plane objects to avoid sharing customized and non-customized versions.

* [`Protocol`](consul-by-hashicorp.md#protocol) `(string)` - The overall protocol shared by everything in the chain.
* [`StartNode`](consul-by-hashicorp.md#startnode) `(string)` - The first key into the `Nodes` map that should be followed when traversing the discovery chain.
* [`Nodes`](consul-by-hashicorp.md#nodes) `(map)` - All nodes available for traversal in the chain keyed by a unique name. You can walk this by starting with `StartNode`.

  The names should be treated as opaque values and are only guaranteed to be consistent within a single compilation.

* [`Targets`](consul-by-hashicorp.md#targets) `(map)` - A list of all targets used in this chain.

  The names should be treated as opaque values and are only guaranteed to be consistent within a single compilation.

#### [»](consul-by-hashicorp.md#discoverygraphnode)`DiscoveryGraphNode`

A single node in the compiled discovery chain.

* [`Type`](consul-by-hashicorp.md#type) `(string)` - The type of the node. Valid values are: `router`, `splitter`, and `resolver`.
* [`Name`](consul-by-hashicorp.md#name) `(string)` - The unique name of the node.
* [`Routes`](consul-by-hashicorp.md#routes) `(array)` - Only set for `Type:router`. List of routes to render.
  * [`Definition`](consul-by-hashicorp.md#definition) `(ServiceRoute)` - Relevant portion of underlying `service-router` [route](https://www.consul.io/docs/agent/config-entries/service-router#routes).
  * [`NextNode`](consul-by-hashicorp.md#nextnode) `(string)` - The name of the next node in the chain in [`Nodes`](consul-by-hashicorp.md#nodes).
* [`Splits`](consul-by-hashicorp.md#splits) `(array)` - Only set for `Type:splitter`. List of traffic splits.
  * [`Weight`](consul-by-hashicorp.md#weight) `(float32)` - Copy of underlying `service-splitter` [`weight`](https://www.consul.io/docs/agent/config-entries/service-splitter#weight) field.
  * [`NextNode`](consul-by-hashicorp.md#nextnode-1) `(string)` - The name of the next node in the chain in [`Nodes`](consul-by-hashicorp.md#nodes).
* [`Resolver`](consul-by-hashicorp.md#resolver) `(DiscoveryResolver:)` - Only set for `Type:resolver`. How to resolve the service instances.
  * [`Default`](consul-by-hashicorp.md#default) `(bool)` - Set to true if no `service-resolver` config entry is defined for this node and the default was synthesized.
  * [`ConnectTimeout`](consul-by-hashicorp.md#connecttimeout) `(duration)` - Copy of the underlying `service-resolver` [`ConnectTimeout`](https://www.consul.io/docs/agent/config-entries/service-resolver#connecttimeout) field. If one is not defined the default of `5s` is returned.
  * [`Target`](consul-by-hashicorp.md#target) `(string)` - The name of the target to use found in [`Targets`](consul-by-hashicorp.md#targets).
  * [`Failover`](consul-by-hashicorp.md#failover) `(DiscoveryFailover:)` - Compiled form of the underlying `service-resolver` [`Failover`](https://www.consul.io/docs/agent/config-entries/service-resolver#failover) definition to use for this request.
    * [`Targets`](consul-by-hashicorp.md#targets-1) `(array)` - List of targets found in [`Targets`](consul-by-hashicorp.md#targets) to failover to in order of preference.
  * [`LoadBalancer`](consul-by-hashicorp.md#loadbalancer) `(LoadBalancer:`\) - Copy of the underlying `service-resolver` [`LoadBalancer`](https://www.consul.io/docs/agent/config-entries/service-resolver#loadbalancer) field.

    If a `service-splitter` splits between services with differing `LoadBalancer` configuration the first hash-based load balancing policy is copied.

#### [»](consul-by-hashicorp.md#discoverytarget)`DiscoveryTarget`

* [`ID`](consul-by-hashicorp.md#id) `(string)` - The unique name of this target.
* [`Service`](consul-by-hashicorp.md#service) `(string)` - The service to query when resolving a list of service instances.
* [`ServiceSubset`](consul-by-hashicorp.md#servicesubset) `(string:)` - The [subset](https://www.consul.io/docs/agent/config-entries/service-resolver#service-subsets) of the service to resolve.
* [`Namespace`](consul-by-hashicorp.md#namespace-1) `(string)` - The namespace to use when resolving a list of service instances.
* [`Datacenter`](consul-by-hashicorp.md#datacenter-1) `(string)` - The datacenter to use when resolving a list of service instances.
* [`Subset`](consul-by-hashicorp.md#subset) `(ServiceResolverSubset)` - Copy of the underlying `service-resolver` [`Subsets`](https://www.consul.io/docs/agent/config-entries/service-resolver#subsets) definition for this target.
  * [`Filter`](consul-by-hashicorp.md#filter) `(string: "")` - The [filter expression](https://www.consul.io/api/features/filtering) to be used for selecting instances of the requested service. If empty all healthy instances are returned.
  * [`OnlyPassing`](consul-by-hashicorp.md#onlypassing) `(bool: false)` - Specifies the behavior of the resolver's health check interpretation. If this is set to false, instances with checks in the passing as well as the warning states will be considered healthy. If this is set to true, only instances with checks in the passing state will be considered healthy.
* [`MeshGateway`](consul-by-hashicorp.md#meshgateway) `(MeshGatewayConfig)` - The [mesh gateway configuration](https://www.consul.io/docs/connect/mesh-gateway#connect-proxy-configuration) to use when connecting to this target's service instances.
  * [`Mode`](consul-by-hashicorp.md#mode) `(string: "")` - One of `none`, `local`, or `remote`.
* [`External`](consul-by-hashicorp.md#external) `(bool: false)` - True if this target is outside of this consul cluster.
* [`SNI`](consul-by-hashicorp.md#sni) `(string)` - This value should be used as the [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) value when connecting to this set of endpoints over TLS.
* [`Name`](consul-by-hashicorp.md#name-1) `(string)` - The unique name for this target for use when generating load balancer objects. This has a structure similar to [SNI](consul-by-hashicorp.md#sni), but will not be affected by SNI customizations such as [`ExternalSNI`](https://www.consul.io/docs/agent/config-entries/service-defaults#externalsni).

