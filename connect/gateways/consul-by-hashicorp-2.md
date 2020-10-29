# Consul by HashiCorp

**1.8.0+:** This feature is available in Consul versions 1.8.0 and newer.

Terminating gateways enable connectivity from services in the Consul service mesh to services outside the mesh. These gateways effectively act as Connect proxies that can represent more than one service. They terminate Connect mTLS connections, enforce intentions, and forward requests to the appropriate destination.

For additional use cases and usage patterns, review the tutorial for [understanding terminating gateways](https://learn.hashicorp.com/tutorials/consul/service-mesh-terminating-gateways).

**Known limitations:** Terminating gateways currently do not support targeting service subsets with [L7 configuration](https://www.consul.io/docs/connect/l7-traffic-management). They route to all instances of a service with no capabilities for filtering by instance.

## [»](consul-by-hashicorp-2.md#security-considerations)Security Considerations

We recommend that terminating gateways are not exposed to the WAN or open internet. This is because terminating gateways hold certificates to decrypt Consul Connect traffic directed at them and may be configured with credentials to connect to linked services. Connections over the WAN or open internet should flow through [mesh gateways](https://www.consul.io/docs/connect/mesh-gateway) whenever possible since they are not capable of decrypting traffic or connecting directly to services.

By specifying a path to a [CA file](https://www.consul.io/docs/agent/config-entries/terminating-gateway#cafile) connections from the terminating gateway will be encrypted using one-way TLS authentication. If a path to a [client certificate](https://www.consul.io/docs/agent/config-entries/terminating-gateway#certfile) and [private key](https://www.consul.io/docs/agent/config-entries/terminating-gateway#keyfile) are also specified connections from the terminating gateway will be encrypted using mutual TLS authentication.

If none of these are provided, Consul will **only** encrypt connections to the gateway and not from the gateway to the destination service.

When certificates for linked services are rotated, the gateway must be restarted to pick up the new certificates from disk. To avoid downtime, perform a rolling restart to reload the certificates. Registering multiple terminating gateway instances with the same [name](https://www.consul.io/commands/connect/envoy#service) provides additional fault tolerance as well as the ability to perform rolling restarts.

**Note:** If certificates and keys are configured the terminating gateway will upgrade HTTP connections to TLS. Client applications can issue plain HTTP requests even when connecting to servers that require HTTPS.

## [»](consul-by-hashicorp-2.md#prerequisites)Prerequisites

Each terminating gateway needs:

1. A local Consul client agent to manage its configuration.
2. General network connectivity to services within its local Consul datacenter.

Terminating gateways also require that your Consul datacenters are configured correctly:

* You'll need to use Consul version 1.8.0 or newer.
* Consul [Connect](https://www.consul.io/docs/agent/options#connect) must be enabled on the datacenter's Consul servers.
* [gRPC](https://www.consul.io/docs/agent/options#grpc_port) must be enabled on all client agents.

Currently, [Envoy](https://www.envoyproxy.io/) is the only proxy with terminating gateway capabilities in Consul.

* Terminating gateway proxies receive their configuration through Consul, which automatically generates it based on the gateway's registration. Currently Consul can only translate terminating gateway registration information into Envoy configuration, therefore the proxies acting as terminating gateways must be Envoy.

Connect proxies that send upstream traffic through a gateway aren't affected when you deploy terminating gateways. If you are using non-Envoy proxies as Connect proxies they will continue to work for traffic directed at services linked to a terminating gateway as long as they discover upstreams with the [/health/connect](https://www.consul.io/api/health#list-nodes-for-connect-capable-service) endpoint.

## [»](consul-by-hashicorp-2.md#running-and-using-a-terminating-gateway)Running and Using a Terminating Gateway

For a complete example of how to enable connections from services in the Consul service mesh to services outside the mesh, review the [terminating gateway tutorial](https://learn.hashicorp.com/tutorials/consul/teminating-gateways-connect-external-services).

## [»](consul-by-hashicorp-2.md#terminating-gateway-configuration)Terminating Gateway Configuration

Terminating gateways are configured in service definitions and registered with Consul like other services, with two exceptions. The first is that the [kind](https://www.consul.io/api/agent/service#kind) must be "terminating-gateway". Second, the terminating gateway service definition may contain a `Proxy.Config` entry just like a Connect proxy service, to define opaque configuration parameters useful for the actual proxy software. For Envoy there are some supported [gateway options](../proxies/consul-by-hashicorp.md#gateway-options) as well as [escape-hatch overrides](../proxies/consul-by-hashicorp.md#escape-hatch-overrides).

**Note:** If ACLs are enabled, terminating gateways must be registered with a token granting `node:read` on the nodes of all services in its configuration entry. The token must also grant `service:write` for the terminating gateway's service name **and** the names of all services in the terminating gateway's configuration entry. These privileges will authorize the gateway to terminate mTLS connections on behalf of the linked services and then route the traffic to its final destination. If the Consul client agent on the gateway's node is not configured to use the default gRPC port, 8502, then the gateway's token must also provide `agent:read` for its node's name in order to discover the agent's gRPC port. gRPC is used to expose Envoy's xDS API to Envoy proxies.

Linking services to a terminating gateway is done with a `terminating-gateway` [configuration entry](https://www.consul.io/docs/agent/config-entries/terminating-gateway). This config entry can be applied via the [CLI](https://www.consul.io/commands/config/write) or [API](https://www.consul.io/api/config#apply-configuration).

Gateways with the same name in Consul's service catalog are configured with a single configuration entry. This means that additional gateway instances registered with the same name will determine their routing based on the existing configuration entry. Adding replicas of a gateway that routes to a particular set of services requires running the [envoy subcommand](https://www.consul.io/commands/connect/envoy#terminating-gateways) on additional hosts and specifying the same gateway name with the `service` flag.

[Configuration entries](https://www.consul.io/docs/agent/config-entries) are global in scope. A configuration entry for a gateway name applies across all federated Consul datacenters. If terminating gateways in different Consul datacenters need to route to different sets of services within their datacenter then the terminating gateways **must** be registered with different names.

The services that the terminating gateway will proxy for must be registered with Consul, even the services outside the mesh. They must also be registered in the same Consul datacenter as the terminating gateway. Otherwise the terminating gateway will not be able to discover the services' addresses. These services can be registered with a local Consul agent. If there is no agent present, the services can be registered [directly in the catalog](https://www.consul.io/api/catalog#register-entity) by sending the registration request to a client or server agent on a different host.

All services registered in the Consul catalog must be associated with a node, even when their node is not managed by a Consul client agent. All agent-less services with the same address can be registered under the same node name and address. However, ensure that the [node name](https://www.consul.io/api/catalog#node) for external services registered directly in the catalog does not match the node name of any Consul client agent node. If the node name overlaps with the node name of a Consul client agent, Consul's [anti-entropy sync](https://www.consul.io/docs/internals/anti-entropy) will delete the services registered via the `/catalog/register` HTTP API endpoint.

For a complete example of how to register external services review the [external services tutorial](https://learn.hashicorp.com/tutorials/consul/service-registration-external-services).
