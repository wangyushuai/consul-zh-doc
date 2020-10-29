# Consul by HashiCorp

Consul comes with a built-in L4 proxy for testing and development with Consul Connect.

**Note:** [Envoy](consul-by-hashicorp.md) should be used for production deployments, or when [layer 7 traffic management](https://www.consul.io/docs/connect/l7-traffic-management) features are needed.

## [»](consul-by-hashicorp-1.md#getting-started)Getting Started

To get started with the built-in proxy and see a working example you can follow the [Getting Started](https://learn.hashicorp.com/tutorials/consul/get-started-service-networking) tutorial.

## [»](consul-by-hashicorp-1.md#proxy-config-key-reference)Proxy Config Key Reference

Below is a complete example of all the configuration options available for the built-in proxy.

```text
{
  "service": {
    ...
    "connect": {
      "proxy": {
        "config": {
          "bind_address": "0.0.0.0",
          "bind_port": 20000,
          "tcp_check_address": "192.168.0.1",
          "disable_tcp_check": false,
          "local_service_address": "127.0.0.1:1234",
          "local_connect_timeout_ms": 1000,
          "handshake_timeout_ms": 10000,
          "upstreams": [...]
        },
        "upstreams": [
          {
            ...
            "config": {
              "connect_timeout_ms": 1000
            }
          }
        ]
      }
    }
  }
}
```

All fields are optional with a sane default.

* [`bind_address`](consul-by-hashicorp-1.md#bind_address) - The address the proxy will bind its _public_ mTLS listener to. It defaults to the same address the agent binds to.
* [`bind_port`](consul-by-hashicorp-1.md#bind_port) - The port the proxy will bind its _public_ mTLS listener to. If not provided, the agent will attempt to assign one from its [configured proxy port range](https://www.consul.io/docs/agent/options#sidecar_min_port) if available. By default the range is \[20000, 20255\] and the port is selected at random from that range.
* [`tcp_check_address`](consul-by-hashicorp-1.md#tcp_check_address) - The address the agent will run a [TCP health check](https://www.consul.io/docs/agent/checks) against. By default this is the same as the proxy's [bind address](consul-by-hashicorp-1.md#bind_address) except if the bind address is `0.0.0.0` or `[::]` in which case this defaults to `127.0.0.1` and assumes the agent can dial the proxy over loopback. For more complex configurations where agent and proxy communicate over a bridge for example, this configuration can be used to specify a different _address_ \(but not port\) for the agent to use for health checks if it can't talk to the proxy over localhost or its publicly advertised port. The check always uses the same port that the proxy is bound to.
* [`disable_tcp_check`](consul-by-hashicorp-1.md#disable_tcp_check) - If true, this disables a TCP check being setup for the proxy. Default is false.
* [`local_service_address`](consul-by-hashicorp-1.md#local_service_address)- The `[address]:port` that the proxy should use to connect to the local application instance. By default it assumes `127.0.0.1` as the address and takes the port from the service definition's `port` field. Note that allowing the application to listen on any non-loopback address may expose it externally and bypass Connect's access enforcement. It may be useful though to allow non-standard loopback addresses or where an alternative known-private IP is available for example when using internal networking between containers.
* [`local_connect_timeout_ms`](consul-by-hashicorp-1.md#local_connect_timeout_ms) - The number of milliseconds the proxy will wait to establish a connection to the _local application_ before giving up. Defaults to `1000` or 1 second.
* [`handshake_timeout_ms`](consul-by-hashicorp-1.md#handshake_timeout_ms) - The number of milliseconds the proxy will wait for _incoming_ mTLS connections to complete the TLS handshake. Defaults to `10000` or 10 seconds.
* [`upstreams`](consul-by-hashicorp-1.md#upstreams)- **Deprecated** Upstreams are now specified in the `connect.proxy` definition. Upstreams specified in the opaque config map here will continue to work for compatibility but it's strongly recommended that you move to using the higher level [upstream configuration](../registration/consul-by-hashicorp.md#upstream-configuration-reference).

## [»](consul-by-hashicorp-1.md#proxy-upstream-config-key-reference)Proxy Upstream Config Key Reference

All fields are optional with a sane default.

* [`connect_timeout_ms`](consul-by-hashicorp-1.md#connect_timeout_ms) - The number of milliseconds the proxy will wait to establish a TLS connection to the discovered upstream instance before giving up. Defaults to `10000` or 10 seconds.
