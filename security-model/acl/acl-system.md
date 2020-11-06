# ACL系统

1.4.0及以后的版本。本指南只适用于Consul 1.4.0及以后的版本。旧版ACL系统的文档在[这里](https://www.consul.io/docs/acl/acl-legacy)。

Consul提供了一个可选的访问控制列表（ACL）系统，可用于控制对数据和API的访问。ACL是[基于能力的](https://en.wikipedia.org/wiki/Capability-based_security)\(_系统的安全模型之一_\)，依靠与策略相关联的令牌来确定可以应用哪些细粒度的规则。Consul的基于能力的ACL系统与[AWS IAM](https://aws.amazon.com/iam/)的设计非常相似。

To learn how to setup the ACL system on an existing Consul datacenter, use the [Bootstrapping The ACL System tutorial](https://learn.hashicorp.com/tutorials/consul/access-control-setup?utm_source=consul.io&utm_medium=docs).

要学习如何在现有的Consul数据中心上设置ACL系统，请使用[Bootstrapping The ACL系统教程](https://learn.hashicorp.com/tutorials/consul/access-control-setup?utm_source=consul.io&utm_medium=docs)。

## [»](acl-system.md#acl-system-overview)ACL系统概览

ACL系统的设计是为了便于使用和快速执行，同时提供管理方面的见解。宏观来看，ACL系统有两个主要组成部分：

* **ACL Policies** - Policies allow the grouping of a set of rules into a logical unit that can be reused and linked with many tokens.
* **ACL Tokens** - Requests to Consul are authorized by using bearer token. Each ACL token has a public Accessor ID which is used to name a token, and a Secret ID which is used as the bearer token used to make requests to Consul.

For many scenarios policies and tokens are sufficient, but more advanced setups may benefit from additional components in the ACL system:

* **ACL Roles** - Roles allow for the grouping of a set of policies and service identities into a reusable higher-level entity that can be applied to many tokens. \(Added in Consul 1.5.0\)
* **ACL Service Identities** - Service identities are a policy template for expressing a link to a policy suitable for use in [Consul Connect](https://www.consul.io/docs/connect). At authorization time this acts like an additional policy was attached, the contents of which are described further below. These are directly attached to tokens and roles and are not independently configured. \(Added in Consul 1.5.0\)
* **ACL Node Identities** - Node identities are a policy template for expressing a link to a policy suitable for use as an [Consul `agent` token](https://www.consul.io/docs/agent/options#acl_tokens_agent) . At authorization time this acts like an additional policy was attached, the contents of which are described further below. These are directly attached to tokens and roles and are not independently configured. \(Added in Consul 1.8.1\)
* **ACL Auth Methods and Binding Rules** - To learn more about these topics, see the [auth methods documentation page](https://www.consul.io/docs/acl/auth-methods).

ACL tokens, policies, roles, auth methods, and binding rules are managed by Consul operators via Consul's [ACL API](https://www.consul.io/api/acl/acl), [ACL CLI](https://www.consul.io/commands/acl), or systems like [HashiCorp's Vault](https://www.vaultproject.io/docs/secrets/consul).

If the ACL system becomes inoperable, you can follow the [reset procedure](https://learn.hashicorp.com/tutorials/consul/access-control-troubleshoot#reset-the-acl-system) at any time.

### [»](acl-system.md#acl-policies)ACL Policies

An ACL policy is a named set of rules and is composed of the following elements:

* **ID** - The policy's auto-generated public identifier.
* **Name** - A unique meaningful name for the policy.
* **Description** - A human readable description of the policy. \(Optional\)
* **Rules** - Set of rules granting or denying permissions. See the [Rule Specification](https://www.consul.io/docs/acl/acl-rules#rule-specification) documentation for more details.
* **Datacenters** - A list of datacenters the policy is valid within.
* **Namespace** -

  Enterprise - The namespace this policy resides within. \(Added in Consul Enterprise 1.7.0\)

**Consul Enterprise Namespacing** - Rules defined in a policy in any namespace other than `default` will be [restricted](https://www.consul.io/docs/acl/acl-rules#namespace-rules) to being able to grant a subset of the overall privileges and only affecting that single namespace.

#### [»](acl-system.md#builtin-policies)Builtin Policies

* **Global Management** - Grants unrestricted privileges to any token that uses it. When created it will be named `global-management` and will be assigned the reserved ID of `00000000-0000-0000-0000-000000000001`. This policy can be renamed but modification of anything else including the rule set and datacenter scoping will be prevented by Consul.
* **Namespace Management** -

  Enterprise - Every namespace created will have a policy injected with the name `namespace-management`. This policy gets injected with a randomized UUID and may be managed like any other user-defined policy within the Namespace. \(Added in Consul Enterprise 1.7.0\)

### [»](acl-system.md#acl-service-identities)ACL Service Identities

Added in Consul 1.5.0

An ACL service identity is an [ACL policy](https://www.consul.io/docs/acl/acl-system#acl-policies) template for expressing a link to a policy suitable for use in [Consul Connect](https://www.consul.io/docs/connect). They are usable on both tokens and roles and are composed of the following elements:

* **Service Name** - The name of the service.
* **Datacenters** - A list of datacenters the effective policy is valid within. \(Optional\)

Services participating in the service mesh will need privileges to both _be discovered_ and to _discover other healthy service instances_. Suitable policies tend to all look nearly identical so a service identity is a policy template to aid in avoiding boilerplate policy creation.

During the authorization process, the configured service identity is automatically applied as a policy with the following preconfigured [ACL rules](https://www.consul.io/docs/acl/acl-system#acl-rules-and-scope):

```text

service "" {
    policy = "write"
}
service "-sidecar-proxy" {
    policy = "write"
}


service_prefix "" {
    policy = "read"
}
node_prefix "" {
    policy = "read"
}
```

The [API documentation for roles](https://www.consul.io/api/acl/roles#sample-payload) has some examples of using a service identity.

**Consul Enterprise Namespacing** - Service Identity rules will be scoped to the single namespace that the corresponding ACL Token or Role resides within.

### [»](acl-system.md#acl-node-identities)ACL Node Identities

Added in Consul 1.8.1

An ACL node identity is an [ACL policy](https://www.consul.io/docs/acl/acl-system#acl-policies) template for expressing a link to a policy suitable for use as an [Consul `agent` token](https://www.consul.io/docs/agent/options#acl_tokens_agent). They are usable on both tokens and roles and are composed of the following elements:

* **Node Name** - The name of the node to grant access to.
* **Datacenter** - The datacenter that the node resides within.

During the authorization process, the configured node identity is automatically applied as a policy with the following preconfigured [ACL rules](https://www.consul.io/docs/acl/acl-system#acl-rules-and-scope):

```text

node "" {
  policy = "write"
}




service_prefix "" {
  policy = "read"
}
```

**Consul Enterprise Namespacing** - Node Identities can only be applied to tokens and roles in the `default` namespace. The synthetic policy rules allow for `service:read` permissions on all services in all namespaces.

### [»](acl-system.md#acl-roles)ACL Roles

Added in Consul 1.5.0

An ACL role is a named set of policies and service identities and is composed of the following elements:

* **ID** - The role's auto-generated public identifier.
* **Name** - A unique meaningful name for the role.
* **Description** - A human readable description of the role. \(Optional\)
* **Policy Set** - The list of policies that are applicable for the role.
* **Service Identity Set** - The list of service identities that are applicable for the role.
* **Namespace**

  Enterprise - The namespace this policy resides within. \(Added in Consul Enterprise 1.7.0\)

**Consul Enterprise Namespacing** - Roles may only link to policies defined in the same namespace as the role itself.

### [»](acl-system.md#acl-tokens)ACL Tokens

ACL tokens are used to determine if the caller is authorized to perform an action. An ACL token is composed of the following elements:

* **Accessor ID** - The token's public identifier.
* **Secret ID** -The bearer token used when making requests to Consul.
* **Description** - A human readable description of the token. \(Optional\)
* **Policy Set** - The list of policies that are applicable for the token.
* **Role Set** - The list of roles that are applicable for the token. \(Added in Consul 1.5.0\)
* **Service Identity Set** - The list of service identities that are applicable for the token. \(Added in Consul 1.5.0\)
* **Locality** - Indicates whether the token should be local to the datacenter it was created within or created in the primary datacenter and globally replicated.
* **Expiration Time** - The time at which this token is revoked. \(Optional; Added in Consul 1.5.0\)
* **Namespace**

  Enterprise - The namespace this policy resides within. \(Added in Consul Enterprise 1.7.0\)

**Consul Enterprise Namespacing** - Tokens may only link to policies and roles defined in the same namespace as the token itself.

#### [»](acl-system.md#builtin-tokens)Builtin Tokens

During cluster bootstrapping when ACLs are enabled both the special `anonymous` and the `master` token will be injected.

* **Anonymous Token** - The anonymous token is used when a request is made to Consul without specifying a bearer token. The anonymous token's description and policies may be updated but Consul will prevent this token's deletion. When created, it will be assigned `00000000-0000-0000-0000-000000000002` for its Accessor ID and `anonymous` for its Secret ID.
* **Master Token** - When a master token is present within the Consul configuration, it is created and will be linked With the builtin Global Management policy giving it unrestricted privileges. The master token is created with the Secret ID set to the value of the configuration entry.

#### [»](acl-system.md#authorization)Authorization

The token Secret ID is passed along with each RPC request to the servers. Consul's [HTTP endpoints](https://www.consul.io/api) can accept tokens via the `token` query string parameter, the `X-Consul-Token` request header, or an [RFC6750](https://tools.ietf.org/html/rfc6750) authorization bearer token. Consul's [CLI commands](https://www.consul.io/docs/commands) can accept tokens via the `token` argument, or the `CONSUL_HTTP_TOKEN` environment variable. The CLI commands can also accept token values stored in files with the `token-file` argument, or the `CONSUL_HTTP_TOKEN_FILE` environment variable.

If no token is provided for an HTTP request then Consul will use the default ACL token if it has been configured. If no default ACL token was configured then the anonymous token will be used.

#### [»](acl-system.md#acl-rules-and-scope)ACL Rules and Scope

The rules from all policies, roles, and service identities linked with a token are combined to form that token's effective rule set. Policy rules can be defined in either an allowlist or denylist mode depending on the configuration of [`acl_default_policy`](https://www.consul.io/docs/agent/options#acl_default_policy). If the default policy is to "deny" access to all resources, then policy rules can be set to allowlist access to specific resources. Conversely, if the default policy is “allow” then policy rules can be used to explicitly deny access to resources.

The following table summarizes the ACL resources that are available for constructing rules:

| Resource | Scope |
| :--- | :--- |
| [`acl`](https://www.consul.io/docs/acl/acl-rules#acl-resource-rules) | Operations for managing the ACL system [ACL API](https://www.consul.io/api/acl/acl) |
| [`agent`](https://www.consul.io/docs/acl/acl-rules#agent-rules) | Utility operations in the [Agent API](https://www.consul.io/api/agent), other than service and check registration |
| [`event`](https://www.consul.io/docs/acl/acl-rules#event-rules) | Listing and firing events in the [Event API](https://www.consul.io/api/event) |
| [`key`](https://www.consul.io/docs/acl/acl-rules#key-value-rules) | Key/value store operations in the [KV Store API](https://www.consul.io/api/kv) |
| [`keyring`](https://www.consul.io/docs/acl/acl-rules#keyring-rules) | Keyring operations in the [Keyring API](https://www.consul.io/api/operator/keyring) |
| [`node`](https://www.consul.io/docs/acl/acl-rules#node-rules) | Node-level catalog operations in the [Catalog API](https://www.consul.io/api/catalog), [Health API](https://www.consul.io/api/health), [Prepared Query API](https://www.consul.io/api/query), [Network Coordinate API](https://www.consul.io/api/coordinate), and [Agent API](https://www.consul.io/api/agent) |
| [`operator`](https://www.consul.io/docs/acl/acl-rules#operator-rules) | Cluster-level operations in the [Operator API](https://www.consul.io/api/operator), other than the [Keyring API](https://www.consul.io/api/operator/keyring) |
| [`query`](https://www.consul.io/docs/acl/acl-rules#prepared-query-rules) | Prepared query operations in the [Prepared Query API](https://www.consul.io/api/query) |
| [`service`](https://www.consul.io/docs/acl/acl-rules#service-rules) | Service-level catalog operations in the [Catalog API](https://www.consul.io/api/catalog), [Health API](https://www.consul.io/api/health), [Prepared Query API](https://www.consul.io/api/query), and [Agent API](https://www.consul.io/api/agent) |
| [`session`](https://www.consul.io/docs/acl/acl-rules#session-rules) | Session operations in the [Session API](https://www.consul.io/api/session) |

Since Consul snapshots actually contain ACL tokens, the [Snapshot API](https://www.consul.io/api/snapshot) requires a token with "write" privileges for the ACL system.

The following resources are not covered by ACL policies:

1. The [Status API](https://www.consul.io/api/status) is used by servers when bootstrapping and exposes basic IP and port information about the servers, and does not allow modification of any state.
2. The datacenter listing operation of the [Catalog API](https://www.consul.io/api/catalog#list-datacenters) similarly exposes the names of known Consul datacenters, and does not allow modification of any state.
3. The [connect CA roots endpoint](https://www.consul.io/api/connect/ca#list-ca-root-certificates) exposes just the public TLS certificate which other systems can use to verify the TLS connection with Consul.

Constructing rules from these policies is covered in detail on the [ACL Rules](https://www.consul.io/docs/acl/acl-rules) page.

**Consul Enterprise Namespacing** - In addition to directly linked policies, roles and service identities, Consul Enterprise will include the ACL policies and roles defined in the [Namespaces definition](https://www.consul.io/docs/enterprise/namespaces#namespace-definition). \(Added in Consul Enterprise 1.7.0\)

## [»](acl-system.md#configuring-acls)Configuring ACLs

ACLs are configured using several different configuration options. These are marked as to whether they are set on servers, clients, or both.

| Configuration Option | Servers | Clients | Purpose |
| :--- | :--- | :--- | :--- |
| [`acl.enabled`](https://www.consul.io/docs/agent/options#acl_enabled) | `REQUIRED` | `REQUIRED` | Controls whether ACLs are enabled |
| [`acl.default_policy`](https://www.consul.io/docs/agent/options#acl_default_policy) | `OPTIONAL` | `N/A` | Determines allowlist or denylist mode |
| [`acl.down_policy`](https://www.consul.io/docs/agent/options#acl_down_policy) | `OPTIONAL` | `OPTIONAL` | Determines what to do when the remote token or policy resolution fails |
| [`acl.role_ttl`](https://www.consul.io/docs/agent/options#acl_role_ttl) | `OPTIONAL` | `OPTIONAL` | Determines time-to-live for cached ACL Roles |
| [`acl.policy_ttl`](https://www.consul.io/docs/agent/options#acl_policy_ttl) | `OPTIONAL` | `OPTIONAL` | Determines time-to-live for cached ACL Policies |
| [`acl.token_ttl`](https://www.consul.io/docs/agent/options#acl_token_ttl) | `OPTIONAL` | `OPTIONAL` | Determines time-to-live for cached ACL Tokens |

A number of special tokens can also be configured which allow for bootstrapping the ACL system, or accessing Consul in special situations:

| Special Token | Servers | Clients | Purpose |
| :--- | :--- | :--- | :--- |
| [`acl.tokens.agent_master`](https://www.consul.io/docs/agent/options#acl_tokens_agent_master) | `OPTIONAL` | `OPTIONAL` | Special token that can be used to access [Agent API](https://www.consul.io/api/agent) when remote bearer token resolution fails; used for setting up the cluster such as doing initial join operations, see the [ACL Agent Master Token](acl-system.md#acl-agent-master-token) section for more details |
| [`acl.tokens.agent`](https://www.consul.io/docs/agent/options#acl_tokens_agent) | `OPTIONAL` | `OPTIONAL` | Special token that is used for an agent's internal operations, see the [ACL Agent Token](acl-system.md#acl-agent-token) section for more details |
| [`acl.tokens.master`](https://www.consul.io/docs/agent/options#acl_tokens_master) | `OPTIONAL` | `N/A` | Special token used to bootstrap the ACL system, check the [Bootstrapping ACLs](https://learn.hashicorp.com/tutorials/consul/access-control-setup-production) tutorial for more details |
| [`acl.tokens.default`](https://www.consul.io/docs/agent/options#acl_tokens_default) | `OPTIONAL` | `OPTIONAL` | Default token to use for client requests where no token is supplied; this is often configured with read-only access to services to enable DNS service discovery on agents |

All of these tokens except the `master` token can all be introduced or updated via the [/v1/agent/token API](https://www.consul.io/api/agent#update-acl-tokens).

#### [»](acl-system.md#acl-agent-master-token)ACL Agent Master Token

Since the [`acl.tokens.agent_master`](https://www.consul.io/docs/agent/options#acl_tokens_agent_master) is designed to be used when the Consul servers are not available, its policy is managed locally on the agent and does not need to have a token defined on the Consul servers via the ACL API. Once set, it implicitly has the following policy associated with it

```text
agent "" {
  policy = "write"
}
node_prefix "" {
  policy = "read"
}
```

#### [»](acl-system.md#acl-agent-token)ACL Agent Token

The [`acl.tokens.agent`](https://www.consul.io/docs/agent/options#acl_tokens_agent) is a special token that is used for an agent's internal operations. It isn't used directly for any user-initiated operations like the [`acl.tokens.default`](https://www.consul.io/docs/agent/options#acl_tokens_default), though if the `acl.tokens.agent_token` isn't configured the `acl.tokens.default` will be used. The ACL agent token is used for the following operations by the agent:

1. Updating the agent's node entry using the [Catalog API](https://www.consul.io/api/catalog), including updating its node metadata, tagged addresses, and network coordinates
2. Performing [anti-entropy](https://www.consul.io/docs/internals/anti-entropy) syncing, in particular reading the node metadata and services registered with the catalog
3. Reading and writing the special `_rexec` section of the KV store when executing [`consul exec`](https://www.consul.io/commands/exec) commands

Here's an example policy sufficient to accomplish the above for a node called `mynode`:

```text
node "mynode" {
  policy = "write"
}
service_prefix "" {
  policy = "read"
}
key_prefix "_rexec" {
  policy = "write"
}
```

The `service_prefix` policy needs read access for any services that can be registered on the agent. If [remote exec is disabled](https://www.consul.io/docs/agent/options#disable_remote_exec), the default, then the `key_prefix` policy can be omitted.

## [»](acl-system.md#next-steps)Next Steps

Setup ACLs with the [Bootstrapping the ACL System tutorial](https://learn.hashicorp.com/tutorials/consul/access-control-setup-production?utm_source=consul.io&utm_medium=docs) or continue reading about [ACL rules](https://www.consul.io/docs/acl/acl-rules).

