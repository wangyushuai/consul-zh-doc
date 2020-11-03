# Consul by HashiCorp

To make Connect aware of proxies you will need to register them in a [service definition](https://www.consul.io/docs/agent/services), just like you would register any other service with Consul. This section outlines your options for registering Connect proxies, either using independent registrations, or in nested sidecar registrations.

## [»](consul-by-hashicorp-3.md#proxy-service-registration)Proxy Service Registration

To register proxies with independent proxy service registrations, you can define them in either in config files or via the API just like any other service. Learn more about all of the options you can define when registering your proxy service in the [proxy registration documentation](registration/consul-by-hashicorp.md).

## [»](consul-by-hashicorp-3.md#sidecar-service-registration)Sidecar Service Registration

To reduce the amount of boilerplate needed for a sidecar proxy, application service definitions may define an inline sidecar service block. This is an opinionated shorthand for a separate full proxy registration as described above. For a description of how to configure the sidecar proxy as well as the opinionated defaults, see the [sidecar service registrations documentation](registration/consul-by-hashicorp-1.md).

