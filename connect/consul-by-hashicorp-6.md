# Consul by HashiCorp

In order to take advantage of Connect's L7 observability features you will need to:

* Deploy sidecar proxies that are capable of emitting metrics with each of your services. We have first class support for Envoy.
* Define where your proxies should send metrics that they collect.
* Define the protocols for each of your services.
* Define the upstreams for each of your services.

If you are using Envoy as your sidecar proxy, you will need to [enable gRPC](https://www.consul.io/docs/agent/options#grpc_port) on your client agents. To define the metrics destination and service protocol you may want to enable [configuration entries](https://www.consul.io/docs/agent/options#config_entries) and [centralized service configuration](https://www.consul.io/docs/agent/options#enable_central_service_config).

If you are using Kubernetes, the Helm chart can simplify much of the necessary configuration, which you can learn about in the [observability tutorial](https://learn.hashicorp.com/tutorials/consul/kubernetes-layer7-observability).

## [»](consul-by-hashicorp-6.md#metrics-destination)Metrics Destination

For Envoy the metrics destination can be configured in the proxy configuration entry's `config` section.

```text
kind = "proxy-defaults"
name = "global"
config {
   "envoy_dogstatsd_url": "udp://127.0.0.1:9125"
}
```

Find other possible metrics syncs in the [Connect Envoy documentation](proxies/consul-by-hashicorp.md#bootstrap-configuration).

## [»](consul-by-hashicorp-6.md#service-protocol)Service Protocol

You can specify the [service protocol](https://www.consul.io/docs/agent/config-entries/service-defaults#protocol) in the `service-defaults` configuration entry. You can override it in the [service registration](https://www.consul.io/docs/agent/services). By default, proxies only give you L4 metrics. This protocol allows proxies to handle requests at the right L7 protocol and emit richer L7 metrics. It also allows proxies to make per-request load balancing and routing decisions.

## [»](consul-by-hashicorp-6.md#service-upstreams)Service Upstreams

You can set the upstream for each service using the proxy's [`upstreams`](registration/consul-by-hashicorp.md#upstreams) sidecar parameter, which can be defined in a service's [sidecar registration](registration/consul-by-hashicorp-1.md).

