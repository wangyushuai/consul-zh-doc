# Consul by HashiCorp

To function as a Connect proxy, proxies must be declared as a proxy types in their service definitions, and provide information about the service they represent.

To declare a service as a proxy, the service definition must contain the following fields:

* [`kind`](consul-by-hashicorp.md#kind) `(string)` must be set to `connect-proxy`. This declares that the service is a proxy type.
* [`proxy.destination_service_name`](consul-by-hashicorp.md#proxy-destination_service_name) `(string)` must be set to the service that this proxy is representing. Note that this replaces `proxy_destination` in versions 1.2.0 to 1.3.0.

  **Deprecation Notice:** From version 1.2.0 to 1.3.0, proxy destination was specified using `proxy_destination` at the top level. This will continue to work until at least 1.5.0 but it's highly recommended to switch to using `proxy.destination_service_name`.

* [`port`](consul-by-hashicorp.md#port) `(int)` must be set so that other Connect services can discover the exact address for connections. `address` is optional if the service is being registered against an agent, since it'll inherit the node address.

Minimal Example:

```text
{
  "name": "redis-proxy",
  "kind": "connect-proxy",
  "proxy": {
    "destination_service_name": "redis"
  },
  "port": 8181
}
```

With this service registered, any Connect clients searching for a Connect-capable endpoint for "redis" will find this proxy.

## [»](consul-by-hashicorp.md#sidecar-proxy-fields)Sidecar Proxy Fields

Most Connect proxies are deployed as "sidecars" which means they are co-located with a single service instance which they represent and proxy all inbound traffic to. In this case the following fields should also be set if you are deploying your proxy as a sidecar but defining it in its own service registration:

* [`proxy.destination_service_id`](consul-by-hashicorp.md#proxy-destination_service_id) `(string:)` is set to the _id_ \(and not the _name_ if they are different\) of the specific service instance that is being proxied. The proxied service is assumed to be registered on the same agent although it's not strictly validated to allow for un-coordinated registrations.
* [`proxy.local_service_port`](consul-by-hashicorp.md#proxy-local_service_port) `(int:)` must specify the port the proxy should use to connect to the _local_ service instance.
* [`proxy.local_service_address`](consul-by-hashicorp.md#proxy-local_service_address) `(string: "")` can be set to override the IP or hostname the proxy should use to connect to the _local_ service. Defaults to `127.0.0.1`.

## [»](consul-by-hashicorp.md#complete-configuration-example)Complete Configuration Example

The following is a complete example showing all the options available when registering a proxy instance.

```text
{
  "name": "redis-proxy",
  "kind": "connect-proxy",
  "proxy": {
    "destination_service_name": "redis",
    "destination_service_id": "redis1",
    "local_service_address": "127.0.0.1",
    "local_service_port": 9090,
    "config": {},
    "upstreams": [],
    "mesh_gateway": {},
    "expose": {}
  },
  "port": 8181
}
```

### [»](consul-by-hashicorp.md#proxy-parameters)Proxy Parameters

* [`destination_service_name`](consul-by-hashicorp.md#destination_service_name) `(string:)` - Specifies the _name_ of the service this instance is proxying. Both side-car and centralized load-balancing proxies must specify this. It is used during service discovery to find the correct proxy instances to route to for a given service name.
* [`destination_service_id`](consul-by-hashicorp.md#destination_service_id) `(string: "")` - Specifies the _ID_ of a single specific service instance that this proxy is representing. This is only valid for side-car style proxies that run on the same node. It is assumed that the service instance is registered via the same Consul agent so the ID is unique and has no node qualifier. This is useful to show in tooling which proxy instance is a side-car for which application instance and will enable fine-grained analysis of the metrics coming from the proxy.
* [`local_service_address`](consul-by-hashicorp.md#local_service_address) `(string: "")` - Specifies the address a side-car proxy should attempt to connect to the local application instance on. Defaults to 127.0.0.1.
* [`local_service_port`](consul-by-hashicorp.md#local_service_port) `(int:)` - Specifies the port a side-car proxy should attempt to connect to the local application instance on. Defaults to the port advertised by the service instance identified by `destination_service_id` if it exists otherwise it may be empty in responses.
* [`config`](consul-by-hashicorp.md#config) `(object: {})` - Specifies opaque config JSON that will be stored and returned along with the service instance from future API calls.
* [`upstreams`](consul-by-hashicorp.md#upstreams) `(array: [])` - Specifies the upstream services this proxy should create listeners for. The format is defined in [Upstream Configuration Reference](consul-by-hashicorp.md#upstream-configuration-reference).
* [`mesh_gateway`](consul-by-hashicorp.md#mesh_gateway) `(object: {})` - Specifies the mesh gateway configuration for this proxy. The format is defined in the [Mesh Gateway Configuration Reference](consul-by-hashicorp.md#mesh-gateway-configuration-reference).
* [`expose`](consul-by-hashicorp.md#expose) `(object: {})` - Specifies the configuration to expose HTTP paths through this proxy. The format is defined in the [Expose Paths Configuration Reference](consul-by-hashicorp.md#expose-paths-configuration-reference), and is only compatible with an Envoy proxy.

## [»](consul-by-hashicorp.md#upstream-configuration-reference)Upstream Configuration Reference

The following examples show all possible upstream configuration parameters.

Note that `snake_case` is used here as it works in both [config file and API registrations](https://www.consul.io/docs/agent/services#service-definition-parameter-case).

Upstreams support multiple destination types. Both examples are shown below followed by documentation for each attribute.

### [»](consul-by-hashicorp.md#service-destination)Service Destination

```text
{
  "destination_type": "service",
  "destination_name": "redis",
  "datacenter": "dc1",
  "local_bind_address": "127.0.0.1",
  "local_bind_port": 1234,
  "config": {},
  "mesh_gateway": {
    "mode": "local"
  }
},
```

### [»](consul-by-hashicorp.md#prepared-query-destination)Prepared Query Destination

```text
{
  "destination_type": "prepared_query",
  "destination_name": "database",
  "local_bind_address": "127.0.0.1",
  "local_bind_port": 1234,
  "config": {}
},
```

* [`destination_name`](consul-by-hashicorp.md#destination_name) `(string:)` - Specifies the name of the service or prepared query to route connect to. The prepared query should be the name or the ID of the prepared query.
* [`destination_namespace`](consul-by-hashicorp.md#destination_namespace) `(string: "")` -

  Enterprise Specifies the namespace of the upstream service.

* [`local_bind_port`](consul-by-hashicorp.md#local_bind_port) `(int:)` - Specifies the port to bind a local listener to for the application to make outbound connections to this upstream.
* [`local_bind_address`](consul-by-hashicorp.md#local_bind_address) `(string: "")` - Specifies the address to bind a local listener to for the application to make outbound connections to this upstream. Defaults to `127.0.0.1`.
* [`destination_type`](consul-by-hashicorp.md#destination_type) `(string: "")` - Specifies the type of discovery query to use to find an instance to connect to. Valid values are `service` or `prepared_query`. Defaults to `service`.
* [`datacenter`](consul-by-hashicorp.md#datacenter) `(string: "")` - Specifies the datacenter to issue the discovery query too. Defaults to the local datacenter.
* [`config`](consul-by-hashicorp.md#config-1) `(object: {})` - Specifies opaque configuration options that will be provided to the proxy instance for this specific upstream. Can contain any valid JSON object. This might be used to configure proxy-specific features like timeouts or retries for the given upstream. See the [built-in proxy configuration reference](../proxies/consul-by-hashicorp-1.md#proxy-upstream-config-key-reference) for options available when using the built-in proxy. If using Envoy as a proxy, see [Envoy configuration reference](../proxies/consul-by-hashicorp.md#proxy-upstream-config-options)
* [`mesh_gateway`](consul-by-hashicorp.md#mesh_gateway-1) `(object: {})` - Specifies the mesh gateway configuration for this proxy. The format is defined in the [Mesh Gateway Configuration Reference](consul-by-hashicorp.md#mesh-gateway-configuration-reference).

## [»](consul-by-hashicorp.md#mesh-gateway-configuration-reference)Mesh Gateway Configuration Reference

The following examples show all possible mesh gateway configurations.

Note that `snake_case` is used here as it works in both [config file and API registrations](https://www.consul.io/docs/agent/services#service-definition-parameter-case).

### [»](consul-by-hashicorp.md#using-a-local-egress-gateway-in-the-local-datacenter)Using a Local/Egress Gateway in the Local Datacenter

```text
{
  "mode": "local"
}
```

### [»](consul-by-hashicorp.md#direct-to-a-remote-ingress-in-a-remote-datacenter)Direct to a Remote/Ingress in a Remote Datacenter

```text
{
  "mode": "remote"
}
```

### [»](consul-by-hashicorp.md#prevent-using-a-mesh-gateway)Prevent Using a Mesh Gateway

```text
{
  "mode": "none"
}
```

### [»](consul-by-hashicorp.md#default-mesh-gateway-mode)Default Mesh Gateway Mode

```text
{
  "mode": ""
}
```

* [`mode`](consul-by-hashicorp.md#mode) `(string: "")` - This defines the mode of operation for how upstreams with a remote destination datacenter get resolved.
  * [`"local"`](consul-by-hashicorp.md#local) - Mesh gateway services in the local datacenter will be used as the next-hop destination for the upstream connection.
  * [`"remote"`](consul-by-hashicorp.md#remote) - Mesh gateway services in the remote/target datacenter will be used as the next-hop destination for the upstream connection.
  * [`"none"`](consul-by-hashicorp.md#none) - No mesh gateway services will be used and the next-hop destination for the connection will be directly to the final service\(s\).
  * [`""`](consul-by-hashicorp.md) - Default mode. The default mode will be `"none"` if no other configuration enables them. The order of precedence for setting the mode is
    1. Upstream
    2. Proxy Service's `Proxy` configuration
    3. The `service-defaults` configuration for the service.
    4. The `global` `proxy-defaults`.

## [»](consul-by-hashicorp.md#expose-paths-configuration-reference)Expose Paths Configuration Reference

The following examples show possible configurations to expose HTTP paths through Envoy.

Exposing paths through Envoy enables a service to protect itself by only listening on localhost, while still allowing non-Connect-enabled applications to contact an HTTP endpoint. Some examples include: exposing a `/metrics` path for Prometheus or `/healthz` for kubelet liveness checks.

Note that `snake_case` is used here as it works in both [config file and API registrations](https://www.consul.io/docs/agent/services#service-definition-parameter-case).

### [»](consul-by-hashicorp.md#expose-listeners-in-envoy-for-http-and-grpc-checks-registered-with-the-local-consul-agent)Expose listeners in Envoy for HTTP and GRPC checks registered with the local Consul agent

```text
{
  "expose": {
    "checks": true
  }
}
```

### [»](consul-by-hashicorp.md#expose-an-http-listener-in-envoy-at-port-21500-that-routes-to-an-http-server-listening-at-port-8080)Expose an HTTP listener in Envoy at port 21500 that routes to an HTTP server listening at port 8080

```text
{
  "expose": {
    "paths": [
      {
        "path": "/healthz",
        "local_path_port": 8080,
        "listener_port": 21500
      }
    ]
  }
}
```

### [»](consul-by-hashicorp.md#expose-an-http2-listener-in-envoy-at-port-21501-that-routes-to-a-grpc-server-listening-at-port-9090)Expose an HTTP2 listener in Envoy at port 21501 that routes to a gRPC server listening at port 9090

```text
{
  "expose": {
    "paths": [
      {
        "path": "/grpc.health.v1.Health/Check",
        "protocol": "http2",
        "local_path_port": 9090,
        "listener_port": 21501
      }
    ]
  }
}
```

* [`checks`](consul-by-hashicorp.md#checks) `(bool: false)` - If enabled, all HTTP and gRPC checks registered with the agent are exposed through Envoy. Envoy will expose listeners for these checks and will only accept connections originating from localhost or Consul's [advertise address](https://www.consul.io/docs/agent/options#advertise). The port for these listeners are dynamically allocated from [expose\_min\_port](https://www.consul.io/docs/agent/options#expose_min_port) to [expose\_max\_port](https://www.consul.io/docs/agent/options#expose_max_port). This flag is useful when a Consul client cannot reach registered services over localhost. One example is when running Consul on Kubernetes, and Consul agents run in their own pods.
* [`paths`](consul-by-hashicorp.md#paths) `array: []` - A list of paths to expose through Envoy.
  * [`path`](consul-by-hashicorp.md#path) `(string: "")` - The HTTP path to expose. The path must be prefixed by a slash. ie: `/metrics`.
  * [`local_path_port`](consul-by-hashicorp.md#local_path_port) `(int: 0)` - The port where the local service is listening for connections to the path.
  * [`listener_port`](consul-by-hashicorp.md#listener_port) `(int: 0)` - The port where the proxy will listen for connections. This port must be available for the listener to be set up. If the port is not free then Envoy will not expose a listener for the path, but the proxy registration will not fail.
  * [`protocol`](consul-by-hashicorp.md#protocol) `(string: "http")` - Sets the protocol of the listener. One of `http` or `http2`. For gRPC use `http2`.

