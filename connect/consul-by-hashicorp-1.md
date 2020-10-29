# Consul by HashiCorp

This page details the inner workings of some of Connect's core features. Understanding how these features work isn't a prerequisite for using Connect, but will help you build a mental model of what's going on under the hood, which may help you reason about Connect's behavior in more complex deployment scenarios.

To try Connect locally, complete the [Getting Started with Consul service mesh](https://learn.hashicorp.com/tutorials/consul/service-mesh?utm_source=WEBSITE&utm_medium=WEB_IO&utm_offer=ARTICLE_PAGE&utm_content=DOCS) tutorial.

## [»](consul-by-hashicorp-1.md#mutual-transport-layer-security-mtls)Mutual Transport Layer Security \(mTLS\)

The core of Connect is based on [mutual TLS](https://en.wikipedia.org/wiki/Mutual_authentication).

Connect provides each service with an identity encoded as a TLS certificate. This certificate is used to establish and accept connections to and from other services. The identity is encoded in the TLS certificate in compliance with the [SPIFFE X.509 Identity Document](https://github.com/spiffe/spiffe/blob/master/standards/X509-SVID.md). This enables Connect services to establish and accept connections with other SPIFFE-compliant systems.

The client service verifies the destination service certificate against the [public CA bundle](https://www.consul.io/api/connect/ca#list-ca-root-certificates). This is very similar to a typical HTTPS web browser connection. In addition to this, the client provides its own client certificate to show its identity to the destination service. If the connection handshake succeeds, the connection is encrypted and authorized.

The destination service verifies the client certificate against the [public CA bundle](https://www.consul.io/api/connect/ca#list-ca-root-certificates). After verifying the certificate, the next step depends upon the configured application protocol of the destination service. TCP \(L4\) services must authorize incoming _connections_ against the configured set of Consul [intentions](consul-by-hashicorp-4.md), whereas HTTP \(L7\) services must authorize incoming _requests_ against those same intentions. If the intention check responds successfully, the connection/request is established. Otherwise the connection/request is rejected.

To generate and distribute certificates, Consul has a built-in CA that requires no other dependencies, and also ships with built-in support for [Vault](ca/consul-by-hashicorp-1.md). The PKI system is designed to be pluggable and can be extended to support any system by adding additional CA providers.

All APIs required for Connect typically respond in microseconds and impose minimal overhead to existing services. To ensure this, Connect-related API calls are all made to the local Consul agent over a loopback interface, and all [agent Connect endpoints](https://www.consul.io/api/agent/connect) implement local caching, background updating, and support blocking queries. Most API calls operate on purely local in-memory data.

## [»](consul-by-hashicorp-1.md#agent-caching-and-performance)Agent Caching and Performance

To enable fast responses on endpoints such as the [agent Connect API](https://www.consul.io/api/agent/connect), the Consul agent locally caches most Connect-related data and sets up background [blocking queries](https://www.consul.io/api/features/blocking) against the server to update the cache in the background. This allows most API calls such as retrieving certificates or authorizing connections to use in-memory data and respond very quickly.

All data cached locally by the agent is populated on demand. Therefore, if Connect is not used at all, the cache does not store any data. On first request, the data is loaded from the server and cached. The set of data cached is: public CA root certificates, leaf certificates, intentions, and service discovery results for upstreams. For leaf certificates and intentions, only data related to the service requested is cached, not the full set of data.

Further, the cache is partitioned by ACL token and datacenters. This is done to minimize the complexity of the cache and prevent bugs where an ACL token may see data it shouldn't from the cache. This results in higher memory usage for cached data since it is duplicated per ACL token, but with the benefit of simplicity and security.

With Connect enabled, you'll likely see increased memory usage by the local Consul agent. The total memory is dependent on the number of intentions related to the services registered with the agent accepting Connect-based connections. The other data \(leaf certificates and public CA certificates\) is a relatively fixed size per service. In most cases, the overhead per service should be relatively small: single digit kilobytes at most.

The cache does not evict entries due to memory pressure. If memory capacity is reached, the process will attempt to swap. If swap is disabled, the Consul agent may begin failing and eventually crash. Cache entries do have TTLs associated with them and will evict their entries if they're not used. Given a long period of inactivity \(3 days by default\), the cache will empty itself.

## [»](consul-by-hashicorp-1.md#connections-across-datacenters)Connections Across Datacenters

A sidecar proxy's [upstream configuration](registration/consul-by-hashicorp.md#upstream-configuration-reference) may specify an alternative datacenter or a prepared query that can address services in multiple datacenters \(such as the [geo failover](https://learn.hashicorp.com/tutorials/consul/automate-geo-failover) pattern\).

[Intentions](consul-by-hashicorp-4.md) verify connections between services by source and destination name seamlessly across datacenters.

Connections can be made via gateways to enable communicating across network topologies, allowing connections between services in each datacenter without externally routable IPs at the service level.

## [»](consul-by-hashicorp-1.md#intention-replication)Intention Replication

Intention replication happens automatically but requires the [`primary_datacenter`](https://www.consul.io/docs/agent/options#primary_datacenter) configuration to be set to specify a datacenter that is authoritative for intentions. In production setups with ACLs enabled, the [replication token](https://www.consul.io/docs/agent/options#acl_tokens_replication) must also be set in the secondary datacenter server's configuration.

## [»](consul-by-hashicorp-1.md#certificate-authority-federation)Certificate Authority Federation

The primary datacenter also acts as the root Certificate Authority \(CA\) for Connect. The primary datacenter generates a trust-domain UUID and obtains a root certificate from the configured CA provider which defaults to the built-in one.

Secondary datacenters fetch the root CA public key and trust-domain ID from the primary and generate their own key and Certificate Signing Request \(CSR\) for an intermediate CA certificate. This CSR is signed by the root in the primary datacenter and the certificate is returned. The secondary datacenter can now use this intermediate to sign new Connect certificates in the secondary datacenter without WAN communication. CA keys are never replicated between datacenters.

The secondary maintains watches on the root CA certificate in the primary. If the CA root changes for any reason such as rotation or migration to a new CA, the secondary automatically generates new keys and has them signed by the primary datacenter's new root before initiating an automatic rotation of all issued certificates in use throughout the secondary datacenter. This makes CA root key rotation fully automatic and with zero downtime across multiple datacenters.
