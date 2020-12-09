# ACL系统

1.4.0及更高版本：本指南仅适用于Consul 1.4.0及更高版本。 旧版ACL系统的文档在[这里](https://www.consul.io/docs/acl/acl-legacy)。

Consul提供了一个可选的访问控制列表（ACL）系统，可用于控制对数据和API的访问。ACL是[基于能力的](https://en.wikipedia.org/wiki/Capability-based_security)\(_系统的安全模型之一_\)，依靠与策略相关联的令牌来确定可以应用哪些细粒度的规则。Consul的基于能力的ACL系统与[AWS IAM](https://aws.amazon.com/iam/)的设计非常相似。

 要了解如何在现有Consul数据中心上设置ACL系统，请使[用Bootstrapping ACL System教程](https://learn.hashicorp.com/tutorials/consul/access-control-setup?utm_source=consul.io&utm_medium=docs)。

## [»](acl-system.md#acl-system-overview)ACL系统概览

ACL系统的设计是为了便于使用和快速执行，同时提供管理方面的能力。宏观来看，ACL系统有两个主要组成部分：

* **ACL策略** -  策略允许将一组规则分组为一个逻辑单元，该逻辑单元可以重复使用并与许多令牌链接。
* **ACL令牌** - 对Consul的请求是通过使用不记名令牌授权的。每个ACL令牌都有一个公共的Accessor ID，用来命名令牌，还有一个Secret ID，作为向Consul提出请求的不记名令牌。

对于许多场景来说，政策和令牌就足够了，但更高级的设置可能会受益于ACL系统中的其他组件。

*  **ACL角色** - 角色允许将一组策略和服务标识归为一个可重复使用的较高级实体，可应用于许多令牌。\(在Consul 1.5.0中添加\) 
* **ACL服务标识** - 服务标识是一种策略模板，用于表达适合在[Consul Connect](https://www.consul.io/docs/connect)中使用的策略链接。在授权时，这就像附加了一个额外的策略，其内容将在下面进一步描述。这些都是直接附加到令牌和角色上的，而不是独立配置的。\(在Consul 1.5.0中添加\)
*  **ACL节点身份** - 节点身份是一个策略模板，用于表达一个适合作为[Consul代理令牌](https://www.consul.io/docs/agent/options#acl_tokens_agent)使用的策略链接。在授权时，这就像附加了一个额外的策略，其内容将在下面进一步描述。这些都是直接附加到令牌和角色上的，而不是独立配置的。\(在Consul 1.8.1中添加\) 
* **ACL 认证方法和绑定规则** - 要了解更多关于这些主题的内容，请参见[认证方法文档页](https://www.consul.io/docs/security/acl/auth-methods)。 

ACL令牌、策略、角色、授权方法和绑定规则由Consul操作员通过Consul的ACL API、ACL CLI或HashiCorp的Vault等系统进行管理。 

如果ACL系统无法使用，您可以在任何时候遵循[重置程序](https://learn.hashicorp.com/tutorials/consul/access-control-troubleshoot#reset-the-acl-system)。

### [»](acl-system.md#acl-policies)ACL 策略

一个ACL策略是一个命名的规则集，由以下元素组成。 

* **ID** - 策略的自动生成的公共标识符。
* **Name** - 策略的唯一有意义的名称。
* **Description** - -策略的可读描述。\(可选\)
* **Rules** - 授予或拒绝权限的一组规则。请参阅[规则规范文档](https://www.consul.io/docs/acl/acl-rules#rule-specification)了解更多详情。
* **Datacenters** - 策略有效的数据中心列表。
* **Namespace** - 企业版本，这个策略所在的命名空间。\(在Consul Enterprise 1.7.0中添加\)

**Consul Enterprise 命名空间** - 策略中定义的规则在除缺省之外的任何命名空间中都将被限制为能够授予整体权限的一个子集，并且只影响该单一命名空间。 

#### [»](acl-system.md#builtin-policies)内置策略

* **全局管理** - 授予任何使用它的令牌不受限制的权限。创建时，它将被命名为 `global-management`，并将被分配保留的 ID `00000000-0000-0000-0000-0000-00000001`。该策略可以重新命名，但其他任何东西的修改，包括规则集和数据中心范围，都将被Consul阻止。
* 命名空间管理 -

  （**企业版本**） - 每个创建的命名空间都会有一个策略，注入名称为_namespace-management_。这个策略会被注入一个随机的UUID，并且可以像其他用户定义的策略一样在命名空间中进行管理。\(在Consul Enterprise 1.7.0中添加\)

### [»](acl-system.md#acl-service-identities)ACL服务标识

Consul 1.5.0增加。

ACL服务标识是一个[ACL策略](https://www.consul.io/docs/acl/acl-system#acl-policies)模板，用于表达适合在Consul Connect中使用的策略链接。它们既可用于令牌。

* **Service Name** - 服务名称
* **Datacenters** - 策略有效的数据中心列表。 \(可选参数\)

参与服务网格的服务将需要特权来发现， 并且发现其他健康的服务实例。合适的策略往往看起来都几乎相同，所以服务标识是一个策略模板，以帮助避免创建模板策略。

在授权过程中，将使用以下预配置的ACL规则自动将配置的服务标识作为策略应用：

```text
# Allow the service and its sidecar proxy to register into the catalog.
service "<Service Name>" {
    policy = "write"
}
service "<Service Name>-sidecar-proxy" {
    policy = "write"
}

# Allow for any potential upstreams to be resolved.
service_prefix "" {
    policy = "read"
}
node_prefix "" {
    policy = "read"
}
```

[这篇用于角色的API文档](https://www.consul.io/api/acl/roles#sample-payload)有一些身份标识的例子。 

**Consul Enterprise Namespacing** - 服务标识规则的范围将限于相应ACL令牌或角色所在的单个命名空间。

### **ACL 节点标识**

在Consul 1.8.1中新增。

ACL节点标识是[ACL策略](https://www.consul.io/docs/acl/acl-system#acl-policies)模板，用于表达指向适合用作Consul代理令牌的策略的链接。 它们在令牌和角色上均可用，并且由以下元素组成：

* **Node Name** - 授权访问的节点名称
* **Datacenter** - 节点所在的数据中心

在授权过程中，配置的节点标识将自动应用为具有以下[预配置 ACL 规则策略](https://www.consul.io/docs/acl/acl-system#acl-rules-and-scope)：

```text
# Allow the agent to register its own node in the Catalog and update its network coordinates
node "<Node Name>" {
  policy = "write"
}

# Allows the agent to detect and diff services registered to itself. This is used during
# anti-entropy to reconcile difference between the agents knowledge of registered
# services and checks in comparison with what is known in the Catalog.
service_prefix "" {
  policy = "read"
}
```

**Consul Enterprise Namespacing** -节点标识只能应用于默认名称空间中的令牌和角色。综合策略规则允许对所有名称空间中的所有服务赋予`service：read`权限。



#### ACL角色

Consul 1.5.0 增加

An ACL role is a named set of policies and service identities and is composed of the following elements:

一个ACL角色是一组命名的策略和服务标识，由以下元素组成：

* **ID** - 角色自动生成的公共标识符
* **Name** - 角色的**唯一**有意义的名称
* **Description** - 对角色可读描述. \(可选\)
* **Policy Set** - 角色包含策略列表
* **Service Identity Set** - 角色包含的服务标识.
* **Namespace** ENTERPRISE -  策略所在的命名空间 \(Consul 企业版本 1.7.0增加\)

**Consul Enterprise Namespacing** - 角色只能链接到在与角色本身相同的名称空间中定义的策略。

#### ACL 令牌

ACL令牌被用于判断调用者是否有权执行响应的操作。 一个ACL令牌由以下元素组成：

* **Accessor ID** - 令牌公共的身份标识；
* **Secret ID** - 请求Consul时不记名令牌；
* **Description** - 可读的不记名令牌描述；\(可选\)
* **Policy Set** - 令牌适用的策略集合；
* **Role Set** - 令牌适用的角色列表；（Consul 1.5.0增加\)
* **Service Identity Set** -  令牌适用的服务标识集合. \(Consul 1.5.0增加\)
* **Locality** - 令牌是在本地数据中心内创建的，还是在主数据中心创建并全局复制的。
* **Expiration Time** - Token失效时间 \(可选; Consul 1.5.0增加\)
* **Namespace** ENTERPRISE - The namespace this policy resides within. \(Added in Consul Enterprise 1.7.0\)

**Consul Enterprise Namespacing** - 令牌只能链接到在与临牌本身相同的名称空间中定义的策略和角色。

**内置令牌**

在集群启动期间，当启用ACL时，特殊的`anonymous`\(匿名\)和 `master`\(主\) 令牌都将被注入。

* 匿名令牌 - 当向Consul发出请求而未指定承载令牌时，将使用匿名令牌。匿名令牌的描述和策略可能会更新，但是Consul将阻止该令牌的删除。创建后，`00000000-0000-0000-0000-000000000002`将作为匿名令牌的 Accessor ID，`anonymous` 作为Secret ID。
* 主令牌 -  当Consul配置中存在主令牌时，将创建该主令牌并将其与内置的全局管理策略链接，从而赋予其不受限制的特权。创建主令牌时，会将Secret ID设置为配置条目的值。

**授权**

令牌秘密ID与每个RPC请求一起传递到服务器。 Consul的[HTTP端点](https://www.consul.io/api)可以通过令牌查询字符串参数，`X-Consul-Token`请求Header或[RFC6750](https://tools.ietf.org/html/rfc6750)授权承载令牌接受令牌。 Consul的[CLI命令](https://www.consul.io/docs/commands)可以通过`token`参数或`CONSUL_HTTP_TOKEN`环境变量接受令牌。 CLI命令还可以接受带有`token-file`参数或`CONSUL_HTTP_TOKEN_FILE`环境变量的文件中存储的令牌值。

如果没有为HTTP请求提供令牌，那么Consul将使用默认的ACL令牌（如果已配置）。如果未配置默认ACL令牌

**ACL规则和范围** 

来自与令牌关联的所有**策略**、**角色**和**服务标识**的规则被合并以形成该令牌的有效规则集。根据[`acl_default_policy`](https://www.consul.io/docs/agent/options#acl_default_policy)的配置, 策略规则可以定义成**允许**列表或**拒绝**列表模式。如果默认策略是“拒绝”访问所有资源，则可以将策略规则设置为允许列表访问特定资源。相反，如果默认策略为“允许”，则可以使用策略规则显式拒绝对资源的访问。

下表总结了可用于构造规则的ACL资源：

| 资源资源 | 范围 |
| :--- | :--- |
| [`acl`](https://www.consul.io/docs/acl/acl-rules#acl-resource-rules) | 管理ACL系统[ACL API的操作](https://www.consul.io/api/acl/acl) |
| [`agent`](https://www.consul.io/docs/acl/acl-rules#agent-rules) | [代理API](https://www.consul.io/api/agent)中的实用程序操作（服务和检查注册除外） |
| [`event`](https://www.consul.io/docs/acl/acl-rules#event-rules) | 列出和触发[事件API](https://www.consul.io/api/event)中的[事件](https://www.consul.io/api/event) |
| [`key`](https://www.consul.io/docs/acl/acl-rules#key-value-rules) | [KV Store API](https://www.consul.io/api/kv)中的键/值存储操作 |
| [`keyring`](https://www.consul.io/docs/acl/acl-rules#keyring-rules) | [Keyring API](https://www.consul.io/api/operator/keyring)中的[密钥环](https://www.consul.io/api/operator/keyring)操作 |
| [`node`](https://www.consul.io/docs/acl/acl-rules#node-rules) | [Catalog API](https://www.consul.io/api/catalog)，[Health API](https://www.consul.io/api/health)，[Prepared Query API](https://www.consul.io/api/query)，[Network Coordinate API](https://www.consul.io/api/coordinate)和[Agent API中的](https://www.consul.io/api/agent)节点级目录操作 |
| [`operator`](https://www.consul.io/docs/acl/acl-rules#operator-rules) | 除了[Keyring API](https://www.consul.io/api/operator/keyring)之外，[Operator API](https://www.consul.io/api/operator)中的集群级操作 |
| [`query`](https://www.consul.io/docs/acl/acl-rules#prepared-query-rules) | 准备查询[API](https://www.consul.io/api/query)中的[准备](https://www.consul.io/api/query)查询操作 |
| [`service`](https://www.consul.io/docs/acl/acl-rules#service-rules) | [Catalog API](https://www.consul.io/api/catalog)，[Health API](https://www.consul.io/api/health)，[Prepared Query API](https://www.consul.io/api/query)和[Agent API中的](https://www.consul.io/api/agent)服务级别目录操作 |
| [`session`](https://www.consul.io/docs/acl/acl-rules#session-rules) | [会话API](https://www.consul.io/api/session)中的[会话](https://www.consul.io/api/session)操作 |

由于Consul快照实际上包含ACL令牌，因此[Snapshot API](https://www.consul.io/api/snapshot) 要求ACL系统具有“写”特权的令牌。

ACL策略未涵盖以下资源：

1. 该[状态API](https://www.consul.io/api/status)所使用的服务器，正在启动并公开基本IP以及有关服务器端口信息时，不允许任何状态的修改。
2. [Catalog API](https://www.consul.io/api/catalog#list-datacenters)的数据中心列表操作，  类似查询已知Consul数据中心的名称，不允许修改任何状态。
3. 该[Connect CA roo](https://www.consul.io/api/connect/ca#list-ca-root-certificates)[ts endpoint](https://www.consul.io/api/connect/ca#list-ca-root-certificates)仅公开了其他系统可以使用它来验证与Consul TLS连接公共TLS证书。

在“ [ACL规则”](https://www.consul.io/docs/acl/acl-rules)页面上详细介绍了根据这些策略构造规则 。

**Consul Enterprise命名空间**-除了直接链接的策略，角色和服务标识之外，Consul Enterprise将包括在[Namespaces定义中定义](https://www.consul.io/docs/enterprise/namespaces#namespace-definition)的ACL策略和角色。（在Consul Enterprise 1.7.0中添加）

### 配置ACL

使用多个不同的配置选项配置ACL。这些配置是区分它们是在服务端，客户端还是两者上设置。

| 配置选项 | Servers | Clients | 作用 |
| :--- | :--- | :--- | :--- |
| [`acl.enabled`](https://www.consul.io/docs/agent/options#acl_enabled) | `REQUIRED` | `REQUIRED` | 控制是否启用ACL |
| [`acl.default_policy`](https://www.consul.io/docs/agent/options#acl_default_policy) | `OPTIONAL` | `N/A` | 确定允许列表或拒绝列表模式 |
| [`acl.down_policy`](https://www.consul.io/docs/agent/options#acl_down_policy) | `OPTIONAL` | `OPTIONAL` | 确定在远程令牌或策略解析失败时该怎么办 |
| [`acl.role_ttl`](https://www.consul.io/docs/agent/options#acl_role_ttl) | `OPTIONAL` | `OPTIONAL` | 确定缓存的ACL角色的生存时间 |
| [`acl.policy_ttl`](https://www.consul.io/docs/agent/options#acl_policy_ttl) | `OPTIONAL` | `OPTIONAL` | 确定缓存的ACL策略的生存时间 |
| [`acl.token_ttl`](https://www.consul.io/docs/agent/options#acl_token_ttl) | `OPTIONAL` | `OPTIONAL` | 确定缓存的ACL令牌的生存时间 |

还可以配置许多特殊令牌，这些令牌允许启动ACL系统或在特殊情况下访问Consul：

| 特殊令牌 | 伺服器 | 客户群 | 目的 |
| :--- | :--- | :--- | :--- |
| [`acl.tokens.agent_master`](https://www.consul.io/docs/agent/options#acl_tokens_agent_master) | `OPTIONAL` | `OPTIONAL` | 远程承载令牌解析失败时可用于访问[Agent API的](https://www.consul.io/api/agent)特殊令牌；用于设置集群（例如执行初始加入操作），请参阅“ [ACL代理主令牌”](https://www.consul.io/docs/security/acl/acl-system#acl-agent-master-token)部分以了解更多详细信息 |
| [`acl.tokens.agent`](https://www.consul.io/docs/agent/options#acl_tokens_agent) | `OPTIONAL` | `OPTIONAL` | 用于代理程序内部操作的特殊令牌，有关更多详细信息，请参见“ [ACL代理程序令牌”](https://www.consul.io/docs/security/acl/acl-system#acl-agent-token)部分 |
| [`acl.tokens.master`](https://www.consul.io/docs/agent/options#acl_tokens_master) | `OPTIONAL` | `N/A` | 用于启动ACL系统的特殊令牌，请查看[Bootstrapping ACL](https://learn.hashicorp.com/tutorials/consul/access-control-setup-production)教程以获取更多详细信息 |
| [`acl.tokens.default`](https://www.consul.io/docs/agent/options#acl_tokens_default) | `OPTIONAL` | `OPTIONAL` | 在没有提供令牌的情况下用于客户端请求的默认令牌；通常将其配置为具有对服务的只读访问权限，以在代理上启用DNS服务发现 |

除令牌外，所有这些令牌`master`都可以通过[/v1/agent/token API](https://www.consul.io/api/agent#update-acl-tokens)引入或更新。

**ACL Agent Master Tokern**

由于[`acl.tokens.agent_master`](https://www.consul.io/docs/agent/options#acl_tokens_agent_master)旨在在Consul服务器不可用时使用，因此，其策略是在代理上本地管理的，不需要通过ACL API在Consul服务器上定义令牌。设置后，它暗中具有与之关联的以下策略

```text
agent "<node name of agent>" {
  policy = "write"
}
node_prefix "" {
  policy = "read"
}
```

**ACL Agent Token**

该[`acl.tokens.agent`](https://www.consul.io/docs/agent/options#acl_tokens_agent)是用于代理的内部操作的特殊标记。它不会直接用于任何用户启动的操作（例如）[`acl.tokens.default`](https://www.consul.io/docs/agent/options#acl_tokens_default)，但是如果`acl.tokens.agent_token`未配置，`acl.tokens.default`则会使用。ACL代理令牌由代理用于以下操作：

1. 使用[目录API](https://www.consul.io/api/catalog)更新代理的节点条目，包括更新其节点元数据，标记的地址和网络坐标
2. 执行[反熵](https://www.consul.io/docs/internals/anti-entropy)同步，尤其是读取在目录中注册的节点元数据和服务
3. `_rexec`执行[`consul exec`](https://www.consul.io/commands/exec)命令时读写KV存储区的特殊部分

这是一个示例策略，足以对称为的节点完成上述操作`mynode`：

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

该`service_prefix`策略需要可以在代理上注册的任何服务的读取访问权限。如果将[remote exec禁用](https://www.consul.io/docs/agent/options#disable_remote_exec)为默认值，则`key_prefix`可以省略该策略。

### [»](https://www.consul.io/docs/security/acl/acl-system#next-steps)下一步

使用[Bootstrapping ACL System教程](https://learn.hashicorp.com/tutorials/consul/access-control-setup-production?utm_source=consul.io&utm_medium=docs)设置ACL或继续阅读有关 [ACL规则的信息](https://www.consul.io/docs/acl/acl-rules)。

















































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































，也可用于角色，并由以下元素组成：

* **Service Name** - 服务名称
* **Datacenters** - 生效的数据中心列表

参与服务网格的的服务将需要特权来发现和发现其他健康的服务实例。合适的策略往往看起来都几乎相同，所以服务标识是一个策略模板，以帮助避免创建模板策略。

在授权过程中，配置好的**服务标识**会自动作为策略应用，其预置的ACL规则如下。

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

[角色的API文档](https://www.consul.io/api/acl/roles#sample-payload)中有一些使用服务身份的例子。

**Consul Enterprise Namespacing** - 服务身份规则将被限定在对应的ACL令牌或角色所在的单一命名空间内。

### [»](acl-system.md#acl-node-identities)ACL节点身份

1.8.1版本中增加。 

ACL节点标识是一个ACL策略模板，用于表达一个适合作为Consul代理令牌使用的策略链接。它们既可用于令牌，也可用于角色，由以下要素组成：

* **Node Name** - 授予访问权限的节点名称.
* **Datacenter** - 节点所在的数据中心.

在授权过程中，节点将自动将以下ACL规则作为策略应用。 

```text

node "" {
  policy = "write"
}




service_prefix "" {
  policy = "read"
}
```

**Consul Enterprise Namespacing** - 节点身份只能应用于默认命名空间中的令牌和角色。合成策略规则允许在所有命名空间的所有服务上拥有 `service:read`权限。

### [»](acl-system.md#acl-roles)ACL角色

在Consul 1.5.0中增加。

在ACL 角色

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

