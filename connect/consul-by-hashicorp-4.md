# Consul by HashiCorp

**1.9.0 and later:** This guide only applies in Consul versions 1.9.0 and later. The documentation for the legacy intentions system is [here](consul-by-hashicorp-5.md).

Intentions define access control for services via Connect and are used to control which services may establish connections or make requests. Intentions can be managed via the API, CLI, or UI.

Intentions are enforced on inbound connections or requests by the [proxy](consul-by-hashicorp-2.md) or within a [natively integrated application](consul-by-hashicorp-11.md).

Depending upon the [protocol](https://www.consul.io/docs/agent/config-entries/service-defaults#protocol) in use by the destination service, you can define intentions to control Connect traffic authorization either at networking layer 4 \(e.g. TCP\) and networking layer 7 \(e.g. HTTP\):

* **Identity-based** - All intentions may enforce access based on identities encoded within [TLS certificates](consul-by-hashicorp-1.md#mutual-transport-layer-security-mtls). This allows for coarse all-or-nothing access control between pairs of services. These work with for services with any [protocol](https://www.consul.io/docs/agent/config-entries/service-defaults#protocol) as they only require awareness of the TLS handshake that wraps the opaque TCP connection. These can also be thought of as **L4 intentions**.
* **Application-aware** - Some intentions may additionally enforce access based on [L7 request attributes](https://www.consul.io/docs/agent/config-entries/service-intentions#permissions) in addition to connection identity. These may only be defined for services with a [protocol](https://www.consul.io/docs/agent/config-entries/service-defaults#protocol) that is HTTP-based. These can also be thought of as **L7 intentions**.

At any given point in time, between any pair of services **only one intention controls authorization**. This may be either an L4 intention or an L7 intention, but at any given point in time only one of those applies.

The [intention match API](https://www.consul.io/api/connect/intentions#list-matching-intentions) should be periodically called to retrieve all relevant intentions for the target destination. After verifying the TLS client certificate, the cached intentions should be consulted for each incoming connection/request to determine if it should be accepted or rejected.

The default intention behavior is defined by the default [ACL policy](https://www.consul.io/docs/agent/options#acl_default_policy). If the default ACL policy is "allow all", then all Connect connections are allowed by default. If the default ACL policy is "deny all", then all Connect connections or requests are denied by default.

## [»](consul-by-hashicorp-4.md#intention-basics)Intention Basics

Intentions are managed primarily via [`service-intentions`](https://www.consul.io/docs/agent/config-entries/service-intentions) config entries or the UI. Some simpler tasks can also be achieved with the older [API](https://www.consul.io/api-docs/connect/intentions) or [CLI](https://www.consul.io/commands/intention). Please see the respective documentation for each for full details on options, flags, etc.

Below is an example of a basic [`service-intentions`](https://www.consul.io/docs/agent/config-entries/service-intentions) config entry representing two simple intentions. The full data model complete with more examples can be found in the [`service-intentions`](https://www.consul.io/docs/agent/config-entries/service-intentions) config entry documentation.

```text
Kind = "service-intentions"
Name = "db"
Sources = [
  {
    Name   = "web"
    Action = "deny"
  },
  {
    Name   = "api"
    Action = "allow"
  }
]
```

This config entry defines two intentions with a common destination of "db". The first intention above is a deny intention with a source of "web". This says that connections from web to db are not allowed and the connection will be rejected. The second intention is an allow intention with a source of "api". This says that connections from api to db are allowed and connections will be accepted.

### [»](consul-by-hashicorp-4.md#wildcard-intentions)Wildcard Intentions

An intention source or destination may also be the special wildcard value `*`. This matches _any_ value and is used as a catch-all.

This example says that the "web" service cannot connect to _any_ service:

```text
Kind = "service-intentions"
Name = "*"
Sources = [
  {
    Name   = "web"
    Action = "deny"
  }
]
```

And this example says that _no_ service can connect to the "db" service:

```text
Kind = "service-intentions"
Name = "db"
Sources = [
  {
    Name   = "*"
    Action = "deny"
  }
]
```

### [»](consul-by-hashicorp-4.md#enforcement)Enforcement

For services that define their [protocol](https://www.consul.io/docs/agent/config-entries/service-defaults#protocol) as TCP, intentions mediate the ability to **establish new connections**. When an intention is modified, existing connections will not be affected. This means that changing a connection from "allow" to "deny" today _will not_ kill the connection.

For services that define their protocol as HTTP-based, intentions mediate the ability to **issue new requests**.

When an intention is modified, requests received after the modification will use the latest intention rules to enforce access. This means that though changing a connection from "allow" to "deny" today will not kill the connection, it will correctly block new requests from being processed.

## [»](consul-by-hashicorp-4.md#precedence-and-match-order)Precedence and Match Order

Intentions are matched in an implicit order based on specificity, preferring deny over allow. Specificity is determined by whether a value is an exact specified value or is the wildcard value `*`. The full precedence table is shown below and is evaluated top to bottom, with larger numbers being evaluated first.

| Source Namespace | Source Name | Destination Namespace | Destination Name | Precedence |
| :--- | :--- | :--- | :--- | :--- |
| Exact | Exact | Exact | Exact | 9 |
| Exact | `*` | Exact | Exact | 8 |
| `*` | `*` | Exact | Exact | 7 |
| Exact | Exact | Exact | `*` | 6 |
| Exact | `*` | Exact | `*` | 5 |
| `*` | `*` | Exact | `*` | 4 |
| Exact | Exact | `*` | `*` | 3 |
| Exact | `*` | `*` | `*` | 2 |
| `*` | `*` | `*` | `*` | 1 |

The precedence value can be read from a [field](https://www.consul.io/docs/agent/config-entries/service-intentions#precedence) on the `service-intentions` config entry after it is modified. Precedence cannot be manually overridden today.

The numbers in the table above are not stable. Their ordering will remain fixed but the actual number values may change in the future.

Enterprise

 - Namespaces are an Enterprise feature. In Consul OSS the only allowable value for either namespace field is `"default"`. Other rows in this table are not applicable.

## [»](consul-by-hashicorp-4.md#intention-management-permissions)Intention Management Permissions

Intention management can be protected by [ACLs](https://www.consul.io/docs/security/acl). Permissions for intentions are _destination-oriented_, meaning the ACLs for managing intentions are looked up based on the destination value of the intention, not the source.

Intention permissions are by default implicitly granted at `read` level when granting `service:read` or `service:write`. This is because a service registered that wants to use Connect needs `intentions:read` for its own service name in order to know whether or not to authorize connections. The following ACL policy will implicitly grant `intentions:read` \(note _read_\) for service `web`.

```text
service "web" {
  policy = "write"
}
```

It is possible to explicitly specify intention permissions. For example, the following policy will allow a service to be discovered without granting access to read intentions for it.

```text
service "web" {
  policy = "read"
  intentions = "deny"
}
```

Note that `intentions:read` is required for a token that a Connect-enabled service uses to register itself or its proxy. If the token used does not have `intentions:read` then the agent will be unable to resolve intentions for the service and so will not be able to authorize any incoming connections.

**Security Note:** Explicitly allowing `intentions:write` on the token you provide to a service instance at registration time opens up a significant additional vulnerability. Although you may trust the service _team_ to define which inbound connections they accept, using a combined token for registration allows a compromised instance to to redefine the intentions which allows many additional attack vectors and may be hard to detect. We strongly recommend only delegating `intentions:write` using tokens that are used by operations teams or orchestrators rather than spread via application config, or only manage intentions with management tokens.

## [»](consul-by-hashicorp-4.md#performance-and-intention-updates)Performance and Intention Updates

The intentions for services registered with a Consul agent are cached locally on that agent. They are then updated via a background blocking query against the Consul servers.

Supported [proxies](consul-by-hashicorp-2.md) \(such as [Envoy](proxies/consul-by-hashicorp.md)\) also cache this data within their own configuration so that inbound connections or requests require no Consul agent involvement during authorization. All actions in the data path of connections happen within the proxy.

Updates to intentions are propagated nearly instantly to agents since agents maintain a continuous blocking query in the background for intention updates for registered services. Proxies similarly use blocking queries to update their local configurations quickly.

Because all the intention data is cached locally, the agents or proxies can fail static. Even if the agents are severed completely from the Consul servers, or the proxies are severed completely from their local Consul agent, inbound connection authorization continues to work indefinitely. Changes to intentions will not be picked up until the partition heals, but will then automatically take effect when connectivity is restored.

