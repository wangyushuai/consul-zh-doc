# Consul by HashiCorp

Any proxy can be extended to support Connect. Consul ships with a built-in proxy for a good development and out of the box experience, but production users will require other proxy solutions.

A proxy must serve one or both of the following two roles: it must accept inbound connections or establish outbound connections identified as a particular service. One or both of these may be implemented depending on the case, although generally both must be supported for full sidecar functionality.

There are also two different levels of compatibility as a sidecar: L4 or L7. L4 integration is simpler and adequate to secure all traffic but treats all traffic as TCP so no advanced routing or metrics features can be supported. Full L7 support is built on top of L4 support and includes supporting most or all of the L7 traffic routing features in Connect by dynamically configuring routing, retries and more L7 features. Currently The built-in proxy only supports L4 while Envoy supports the full L7 feature set.

Places where the integration approach diverges for L4/L7 support is indicated below.

## [»](consul-by-hashicorp-2.md#accepting-inbound-connections)Accepting Inbound Connections

For inbound connections, the proxy must accept TLS connections on some port. The certificate served should be obtained from the [`/v1/agent/connect/ca/leaf/`](https://www.consul.io/api/agent/connect#service-leaf-certificate) API endpoint. The client certificate should be validated against the root certificates provided by the [`/v1/agent/connect/ca/roots`](https://www.consul.io/api/agent/connect#certificate-authority-ca-roots) endpoint. After validating the client certificate from the caller, depending upon the [protocol](https://www.consul.io/docs/agent/config-entries/service-defaults#protocol) of the proxied service service the proxy must either authorize the entire connection \(L4\) or each request \(L7\).

Connection authorization can be performed one of two ways:

1. The first is by calling the [`/v1/agent/connect/authorize`](https://www.consul.io/api/agent/connect) endpoint. The authorize endpoint is expected to be called in the connection path, so if the local Consul agent is down or unresponsive it will impact the success rate of new connections. The agent uses locally cached data to authorize the connection and typically responds in microseconds. Therefore, the impact to the TLS handshake is typically microseconds.

   **Note:** This endpoint is only suited for networking layer 4 \(e.g. TCP\) integration. The endpoint will always treat intentions with Permissions defined \(i.e., layer 7 criteria\) as deny intentions during evaluation.

2. Alternatively, proxies may list intentions that match the destination by querying the [intention match API](https://www.consul.io/api/connect/intentions#list-matching-intentions) endpoint, and represent them in the native configuration of the proxy itself \(such as RBAC for Envoy\). For performance and reliability reasons this is the desirable method for implementing intention enforcement. The cached intentions should be consulted for each incoming connection \(L4\) or request \(L7\) to determine if the should be accepted or rejected.

All of these API endpoints operate on agent-local data that is updated in the background. The leaf, roots, and intentions should be updated in the background by the proxy.

The leaf cert, root cert, and intentions endpoints support [blocking queries](https://www.consul.io/api/features/blocking), which should be used to get near-immediate updates for root key rotations, new leaf certs before expiry, and intention changes.

Although Consul follows the SPIFFE spec for certificates, some currently supported CA providers don't allow strict adherence. For example, CA certificates may not have the correct trust-domain SPIFFE URI SAN for the cluster. If SPIFFE validation is performed in the proxy, be aware that it should be possible to opt out, otherwise certain CA providers supported by Consul will not be compatible with the use of that proxy. Currently neither Envoy nor the built-in proxy validate the SPIFFE URI of the chain beyond the leaf certificate.

### [»](consul-by-hashicorp-2.md#connection-authorization)Connection Authorization

Authentication is based on "service identity" \(TLS\), and is implemented at the transport layer. Depending upon the [protocol](https://www.consul.io/docs/agent/config-entries/service-defaults#protocol) of the proxied service, authorization is performed either on a per-connection \(L4\) or per-request \(L7\) basis.

**Note:** Features like \(local\) rate limiting or max connections are configurations that we expect to push into proxies and have them enforce separately to the AuthZ call based on the state they already have about request rates etc.

#### [»](consul-by-hashicorp-2.md#persistent-tcp-connections-and-intentions)Persistent TCP Connections and Intentions

For a proxied service configured with a [protocol](https://www.consul.io/docs/agent/config-entries/service-defaults#protocol) of TCP, potentially long-lived TCP connections will be authorized only when they are established. Since many services \(e.g. databases\) typically use persistent connection pools, a change in intentions that newly denies access currently does not terminate existing connections in violation of the updated intention. In this case it may appear as if the intention is not being enforced.

Consul eventually may support a mechanism for tracking specific connections in the agent and then allow the agent to tell the proxy to close those connections when their authorization state changes, but for now that is not on the roadmap.

It is recommended therefore to do one of the following:

1. Have connections terminate after a configurable maximum lifetime of say several hours. This balances the overhead of establishing new connections while keeping an upper bound on how long after Intention changes existing connections remain open.
2. Periodically re-authorize every open connection. The AuthZ call itself is not expensive and should be a local, in-memory operation so authorizing thousands of open connections once every minute or so is likely to be negligible overhead, but enforces a tighter upper bound on how long it takes to enforce Intention changes without affecting protocol efficiency of persistent connections.

#### [»](consul-by-hashicorp-2.md#certificate-serial-in-authz)Certificate Serial in AuthZ

Intentions currently utilize TLS' URI Subject Alternative Name \(SAN\) for enforcement. In the future, Consul will support revoking specific certificates by serial number. The AuthZ API in the Go SDK has a field to pass the serial number \([`consul/connect/tls.go`](https://github.com/hashicorp/consul/blob/v1.8.3/connect/tls.go#L232-L237)\). Proxies may provide this value during authorization.

## [»](consul-by-hashicorp-2.md#establishing-outbound-connections)Establishing Outbound Connections

For outbound connections, the proxy should communicate to a Connect-capable endpoint for a service and provide a client certificate from the [`/v1/agent/connect/ca/leaf/`](https://www.consul.io/api/agent/connect#service-leaf-certificate) API endpoint. The certificate served by the remote endpoint may be verified against the root certificates from the [`/v1/agent/connect/ca/roots`](https://www.consul.io/api/agent/connect#certificate-authority-ca-roots) endpoint.

## [»](consul-by-hashicorp-2.md#configuration-discovery)Configuration Discovery

Any proxy can discover proxy configuration registered with a local service instance using the [`/v1/agent/service/:service_id`](https://www.consul.io/api/agent/service#get-service-configuration) API endpoint.

This endpoint supports hash-based blocking, enabling long-polling for changes to the registration/configuration. Any changes to the registration/config will result in the new config being returned immediately. An example implementation may be found in our [built-in proxy](consul-by-hashicorp-1.md) which utilizes our Go SDK, and uses the HTTP "pull" API \(via our `watch` package\): [`consul/connect/proxy/config.go`](https://github.com/hashicorp/consul/blob/v1.8.3/connect/proxy/config.go#L187).

The [discovery chain](../l7-traffic/consul-by-hashicorp.md) for each upstream service should be fetched from the [`/v1/discovery-chain/:service_id`](https://www.consul.io/api/discovery-chain#read-compiled-discovery-chain) API endpoint. This will return a compiled graph of configurations needed by sidecars for a particular upstream service. If you are only implementing L4 support in your proxy, set the [`OverrideProtocol`](https://www.consul.io/api/discovery-chain#overrideprotocol) value to "tcp" when fetching the discovery chain so that L7 features such as HTTP routing rules are not returned.

For each [target](https://www.consul.io/docs/internals/discovery-chain#targets) in the resulting discovery chain, a list of healthy, Connect-capable endpoints may be fetched from the [`/v1/health/connect/:service_id`](https://www.consul.io/api/health#list-nodes-for-connect-capable-service) API endpoint per the [Service Discovery](consul-by-hashicorp-2.md#service-discovery) section below.

The rest of the nodes in the chain include configurations that should be translated into the nearest equivalent for things like HTTP routing, connection timeouts, connection pool settings, rate limits, etc. See the full [discovery chain](../l7-traffic/consul-by-hashicorp.md) documentation and relevant [config entry](https://www.consul.io/docs/agent/config-entries) documentation for details of supported configuration parameters.

We expect config here to evolve reasonably rapidly. While we do not intend to make backwards incompatible API changes, there are likely to be new configurations and features added regularly. Some proxies may not be able to support all features or may have differing semantics with the way they support them. We intend to find a suitable format to document the behavior differences between proxy implementations as they mature.

### [»](consul-by-hashicorp-2.md#service-discovery)Service Discovery

Proxies can use Consul's service discovery API [`/v1/health/connect/:service_id`](https://www.consul.io/api/health#list-nodes-for-connect-capable-service) to return all available, Connect-capable endpoints for a given service. This endpoint supports a `?cached` parameter which makes use of [agent caching](https://www.consul.io/api/features/caching) and thus has performance benefits. The API package provides a [`UseCache`](https://github.com/hashicorp/consul/blob/v1.8.3/api/api.go#L99-L102) query option to leverage this. In addition to performance improvements, use of the cache makes the mesh more resilient to Consul server outages - the mesh "fails static" with the last known set of service instances still used rather than errors on new connections.

Proxies can decide whether to perform just-in-time queries to the API when a new connection needs to be routed, or to use blocking queries to load the current set of endpoints for a service and keep that list updated. The SDK and built-in proxy currently use just-in-time resolution however many existing proxies are likely to find it easier to integrate by pulling the set of endpoints and maintaining it in local memory using blocking queries.

Upstreams can be defined with Prepared Query target types. These upstreams should use Consul's [prepared query](https://www.consul.io/api/query) API. It's worth noting that the PreparedQuery API does not support blocking, so proxies choosing to populate endpoints in memory will need to poll the endpoint at a suitable and ideally configurable frequency.

**Note:** Long-term the [`service-resolver` config entries](https://www.consul.io/docs/agent/config-entries/service-resolver) are intended to replace Prepared Queries in Consul entirely, but for now these are still used in some configurations.

## [»](consul-by-hashicorp-2.md#sidecar-instantiation)Sidecar instantiation

Consul does not start or manage sidecar proxies processes. Proxies running on a physical host or VM are designed to be started and run by process supervisor systems such as init, systemd, supervisord, etc. Or, if deployed within a cluster scheduler \(Kubernetes, Nomad\) running as a sidecar container in the same namespace.

The proxy will use the [`CONSUL_HTTP_TOKEN`](https://www.consul.io/commands#consul_http_token) and [`CONSUL_HTTP_ADDR`](https://www.consul.io/commands#consul_http_addr) environment variables to contact Consul to fetch certificates, provided the `CONSUL_HTTP_TOKEN` environment variable contains a Consul ACL that has the necessary permissions to read configuration for that service. If you use our Go [`api` package](https://github.com/hashicorp/consul/tree/master/api) then those environment variables will be read and the client configured for you automatically.

The ID of the proxy service comes from the user. See [`consul connect envoy`](https://www.consul.io/commands/connect/envoy) as an example. You may start it with the `-proxy-id` flag and pass the ID of the proxy service you registered elsewhere. A nicer UX is available for end-users using the `-sidecar-for=` argument, which causes the command to query Consul for a proxy that is registered as a sidecar for the specified. If there is exactly one such proxy, that ID will be used to start the proxy. Your controller only needs to accept `-proxy-id` as an argument; the Consul CLI will handle resolving the ID for the name specified in `-sidecar-for`.

