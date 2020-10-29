# Consul by HashiCorp

**Note**: The features shown below are extensions of Consul’s service mesh capabilities. If you are not utilizing Consul service mesh then these features will not be relevant to your task.

## [»](consul-by-hashicorp-8.md#service-to-service-traffic-between-consul-datacenters)Service-to-service traffic between Consul datacenters

**1.6.0+:** This feature is available in Consul versions 1.6.0 and newer.

Mesh gateways enable routing of service mesh traffic between different Consul datacenters. Those datacenters can reside in different clouds or runtime environments where general interconnectivity between all services in all datacenters isn't feasible. One scenario where this is useful is when connecting networks with overlapping IP address space.

These gateways operate by sniffing the SNI header out of the mTLS connection and then routing the connection to the appropriate destination based on the server name requested. The data within the mTLS session is not decrypted by the Gateway.

As of Consul 1.8.0, mesh gateways can also forward gossip and RPC traffic between Consul servers. This is enabled by [WAN federation via mesh gateways](https://www.consul.io/docs/connect/gateways/wan-federation-via-mesh-gateways).

For more information about mesh gateways, review the [complete documentation](gateways/consul-by-hashicorp.md) and the [mesh gateway tutorial](https://learn.hashicorp.com/tutorials/consul/service-mesh-gateways).

## [»](consul-by-hashicorp-8.md#traffic-from-outside-the-consul-service-mesh-to-services-in-the-mesh)Traffic from outside the Consul service mesh to services in the mesh

**1.8.0+:** This feature is available in Consul versions 1.8.0 and newer.

Ingress gateways are an entrypoint for outside traffic. They enable potentially unauthenticated ingress traffic from services outside the Consul service mesh to services inside the service mesh.

These gateways allow you to define what services should be exposed, on what port, and by what hostname. You configure an ingress gateway by defining a set of listeners that can map to different sets of backing services.

Ingress gateways are tightly integrated with Consul’s L7 configuration and enable dynamic routing of HTTP requests by attributes like the request path.

For more information about ingress gateways, review the [complete documentation](gateways/consul-by-hashicorp-1.md) and the [ingress gateway tutorial](https://learn.hashicorp.com/tutorials/consul/service-mesh-gateways).

## [»](consul-by-hashicorp-8.md#traffic-from-services-in-the-consul-service-mesh-to-external-services)Traffic from services in the Consul service mesh to external services

**1.8.0+:** This feature is available in Consul versions 1.8.0 and newer.

Terminating gateways enable connectivity from services in the Consul service mesh to services outside the mesh. Services outside the mesh do not have sidecar proxies or are not [integrated natively](consul-by-hashicorp-11.md). These may be services running on legacy infrastructure or managed cloud services running on infrastructure you do not control.

Terminating gateways effectively act as egress proxies that can represent one or more services. They terminate Connect mTLS connections, enforce Consul intentions, and forward requests to the appropriate destination.

These gateways also simplify authorization from dynamic service addresses. Consul’s intentions determine whether connections through the gateway are authorized. Then traditional tools like firewalls or IAM roles can authorize the connections from the known gateway nodes to the destination services.

For more information about terminating gateways, review the [complete documentation](gateways/consul-by-hashicorp-2.md) and the [terminating gateway tutorial](https://learn.hashicorp.com/tutorials/consul/teminating-gateways-connect-external-services).
