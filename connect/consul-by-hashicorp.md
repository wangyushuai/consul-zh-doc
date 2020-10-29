# Consul by HashiCorp

There are many configuration options exposed for Connect. The only option that must be set is the "enabled" option on Consul Servers to enable Connect. All other configurations are optional and have reasonable defaults.

**Tip:** Connect is enabled by default when running Consul in dev mode with `consul agent -dev`.

## [»](consul-by-hashicorp.md#agent-configuration)Agent Configuration

The first step to use Connect is to enable Connect for your Consul cluster. By default, Connect is disabled. Enabling Connect requires changing the configuration of only your Consul _servers_ \(not client agents\). To enable Connect, add the following to a new or existing [server configuration file](https://www.consul.io/docs/agent/options). In an existing cluster, this configuration change requires a Consul server restart, which you can perform one server at a time to maintain availability. In HCL:

```text
connect {
  enabled = true
}
```

This will enable Connect and configure your Consul cluster to use the built-in certificate authority for creating and managing certificates. You may also configure Consul to use an external [certificate management system](consul-by-hashicorp-12.md), such as [Vault](https://vaultproject.io/).

Services and proxies may always register with Connect settings, but they will fail to retrieve or verify any TLS certificates. This causes all Connect-based connection attempts to fail until Connect is enabled on the server agents.

Other optional Connect configurations that you can set in the server configuration file include:

* [certificate authority settings](https://www.consul.io/docs/agent/options#connect)
* [token replication](https://www.consul.io/docs/agent/options#acl_tokens_replication)
* [dev mode](https://www.consul.io/docs/agent/options#_dev)
* [server host name verification](https://www.consul.io/docs/agent/options#verify_server_hostname)

If you would like to use Envoy as your Connect proxy you will need to [enable gRPC](https://www.consul.io/docs/agent/options#grpc_port).

Additionally if you plan on using the observability features of Connect, it can be convenient to configure your proxies and services using [configuration entries](https://www.consul.io/docs/agent/config-entries) which you can interact with using the CLI or API, or by creating configuration entry files. You will want to enable [centralized service configuration](https://www.consul.io/docs/agent/options#enable_central_service_config) on clients, which allows each service's proxy configuration to be managed centrally via API.

**Security note:** Enabling Connect is enough to try the feature but doesn't automatically ensure complete security. Please read the [Connect production tutorial](https://learn.hashicorp.com/tutorials/consul/service-mesh-production-checklist) to understand the additional steps needed for a secure deployment.

## [»](consul-by-hashicorp.md#centralized-proxy-and-service-configuration)Centralized Proxy and Service Configuration

To account for common Connect use cases where you have many instances of the same service, and many colocated sidecar proxies, Consul allows you to customize the settings for all of your proxies or all the instances of a given service at once using [Configuration Entries](https://www.consul.io/docs/agent/config-entries).

You can override centralized configurations for individual proxy instances in their [sidecar service definitions](registration/consul-by-hashicorp-1.md), and the default protocols for service instances in their [service registrations](https://www.consul.io/docs/agent/services).

## [»](consul-by-hashicorp.md#schedulers)Schedulers

Consul Connect is especially useful if you are using an orchestrator like Nomad or Kubernetes, because these orchestrators can deploy thousands of service instances which frequently move hosts. Sidecars for each service can be configured through these schedulers, and in some cases they can automate Consul configuration, sidecar deployment, and service registration.

### [»](consul-by-hashicorp.md#nomad)Nomad

Connect can be used with Nomad to provide secure service-to-service communication between Nomad jobs and task groups. The ability to use the dynamic port feature of Nomad makes Connect particularly easy to use. Learn about how to configure Connect on Nomad by reading the [integration documentation](platform/consul-by-hashicorp.md)

### [»](consul-by-hashicorp.md#kubernetes)Kubernetes

The Consul Helm chart can automate much of Consul Connect's configuration, and makes it easy to automatically inject Envoy sidecars into new pods when they are deployed. Learn about the [Helm chart](https://www.consul.io/docs/platform/k8s/helm) in general, or if you are already familiar with it, check out its [connect specific configurations](https://www.consul.io/docs/platform/k8s/connect).
