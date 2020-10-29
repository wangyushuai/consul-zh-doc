# Consul by HashiCorp

Connect proxies are typically deployed as "sidecars" that run on the same node as the single service instance that they handle traffic for. They might be on the same VM or running as a separate container in the same network namespace.

To simplify the configuration experience when deploying a sidecar for a service instance, Consul 1.3 introduced a new field in the Connect block of the [service definition](https://www.consul.io/docs/agent/services).

To deploy a service and sidecar proxy locally, complete the [Getting Started guide](https://learn.hashicorp.com/tutorials/consul/get-started-service-networking?utm_source=consul.io&utm_medium=docs).

The `connect.sidecar_service` field is a complete nested service definition on which almost any regular service definition field can be set. The exceptions are [noted below](consul-by-hashicorp-1.md#limitations). If used, the service definition is treated identically to another top-level service definition. The value of the nested definition is that _all fields are optional_ with some opinionated defaults applied that make setting up a sidecar proxy much simpler.

## [»](consul-by-hashicorp-1.md#minimal-example)Minimal Example

To register a service instance with a sidecar, all that's needed is:

```text
{
  "service": {
    "name": "web",
    "port": 8080,
    "connect": { "sidecar_service": {} }
  }
}
```

This will register the `web` service as normal, but will also register another [proxy service](../consul-by-hashicorp-2.md) with defaults values used.

The above expands out to be equivalent to the following explicit service definitions:

```text
{
  "service": {
    "name": "web",
    "port": 8080,
  }
}
{
  "name": "web-sidecar-proxy",
  "port": 20000,
  "kind": "connect-proxy",
  "checks": [
    {
      "Name": "Connect Sidecar Listening",
      "TCP": "127.0.0.1:20000",
      "Interval": "10s"
    },
    {
      "name": "Connect Sidecar Aliasing web",
      "alias_service": "web"
    }
  ],
  "proxy": {
    "destination_service_name": "web",
    "destination_service_id": "web",
    "local_service_address": "127.0.0.1",
    "local_service_port": 8080,
  }
}
```

Details on how the defaults are determined are [documented below](consul-by-hashicorp-1.md#sidecar-service-defaults).

**Note:** Sidecar service registrations are only a shorthand for registering multiple services. Consul will not start up or manage the actual proxy processes for you.

## [»](consul-by-hashicorp-1.md#overridden-example)Overridden Example

The following example shows a service definition where some fields are overridden to customize the proxy configuration.

```text
{
  "name": "web",
  "port": 8080,
  "connect": {
    "sidecar_service": {
      "proxy": {
        "upstreams": [
          {
            "destination_name": "db",
            "local_bind_port": 9191
          }
        ],
        "config": {
          "handshake_timeout_ms": 1000
        }
      }
    }
  }
}
```

This example customizes the [proxy upstreams](consul-by-hashicorp.md#upstream-configuration-reference) and some [built-in proxy configuration](../proxies/consul-by-hashicorp-1.md).

## [»](consul-by-hashicorp-1.md#sidecar-service-defaults)Sidecar Service Defaults

The following fields are set by default on a sidecar service registration. With [the exceptions noted](consul-by-hashicorp-1.md#limitations) any field may be overridden explicitly in the `connect.sidecar_service` definition to customize the proxy registration. The "parent" service refers to the service definition that embeds the sidecar proxy.

* [`id`](consul-by-hashicorp-1.md#id) - ID defaults to being`-sidecar-proxy`. This can't be overridden as it is used to [manage the lifecycle](consul-by-hashicorp-1.md#lifecycle) of the registration.
* [`name`](consul-by-hashicorp-1.md#name) - Defaults to being`-sidecar-proxy`.
* [`tags`](consul-by-hashicorp-1.md#tags) - Defaults to the tags of the parent service.
* [`meta`](consul-by-hashicorp-1.md#meta) - Defaults to the service metadata of the parent service.
* [`port`](consul-by-hashicorp-1.md#port) - Defaults to being auto-assigned from a [configurable range](https://www.consul.io/docs/agent/options#sidecar_min_port) that is by default `[21000, 21255]`.
* [`kind`](consul-by-hashicorp-1.md#kind) - Defaults to `connect-proxy`. This can't be overridden currently.
* [`check`](consul-by-hashicorp-1.md#check), `checks` - By default we add a TCP check on the local address and port for the proxy, and a [service alias check](https://www.consul.io/docs/agent/checks#alias) for the parent service. If either `check` or `checks` fields are set, only the provided checks are registered.
* [`proxy.destination_service_name`](consul-by-hashicorp-1.md#proxy-destination_service_name) - Defaults to the parent service name.
* [`proxy.destination_service_id`](consul-by-hashicorp-1.md#proxy-destination_service_id) - Defaults to the parent service ID.
* [`proxy.local_service_address`](consul-by-hashicorp-1.md#proxy-local_service_address) - Defaults to `127.0.0.1`.
* [`proxy.local_service_port`](consul-by-hashicorp-1.md#proxy-local_service_port) - Defaults to the parent service port.

## [»](consul-by-hashicorp-1.md#limitations)Limitations

Almost all fields in a [service definition](https://www.consul.io/docs/agent/services) may be set on the `connect.sidecar_service` except for the following:

* [`id`](consul-by-hashicorp-1.md#id-1) - Sidecar services get an ID assigned and it is an error to override this. This ensures the agent can correctly deregister the sidecar service later when the parent service is removed.
* [`kind`](consul-by-hashicorp-1.md#kind-1) - Kind defaults to `connect-proxy` and there is currently no way to unset this to make the registration be for a regular non-connect-proxy service.
* [`connect.sidecar_service`](consul-by-hashicorp-1.md#connect-sidecar_service) - Service definitions can't be nested recursively.
* [`connect.proxy`](consul-by-hashicorp-1.md#connect-proxy) - \(Deprecated\) [Managed proxies](https://www.consul.io/docs/connect/proxies/managed-deprecated) can't be defined on a sidecar.
* [`connect.native`](consul-by-hashicorp-1.md#connect-native) - Currently the `kind` is fixed to `connect-proxy` and it's an error to register a `connect-proxy` that is also Connect-native.

## [»](consul-by-hashicorp-1.md#lifecycle)Lifecycle

Sidecar service registration is mostly a configuration syntax helper to avoid adding lots of boiler plate for basic sidecar options, however the agent does have some specific behavior around their lifecycle that makes them easier to work with.

The agent fixes the ID of the sidecar service to be based on the parent service's ID. This enables the following behavior.

* A service instance can _only ever have one_ sidecar service registered.
* When re-registering via API or reloading from configuration file:
  * If something changes in the nested sidecar service definition, the change will _update_ the current sidecar registration instead of creating a new one.
  * If a service registration removes the nested `sidecar_service` then the previously registered sidecar for that service will be deregistered automatically.
* When reloading the configuration files, if a service definition changes its ID, then a new service instance _and_ a new sidecar instance will be registered. The old ones will be removed since they are no longer found in the config files.

