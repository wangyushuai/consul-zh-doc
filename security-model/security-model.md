# 概览

Consul依靠轻量级的gossip机制和RPC系统来提供各种功能。这两个系统都有不同的安全机制，这源于它们的设计。然而，Consul的安全机制有一个共同的目标：[提供保密性、完整性和认证功能](https://en.wikipedia.org/wiki/Information_security)。

 [Gossip协议](https://www.consul.io/docs/internals/gossip)由[Serf](https://www.serf.io)提供支持，它使用对称密钥或共享秘密的密码系统。这里有更多关于Serf安全性的[细节](https://www.serf.io/docs/internals/security.html)。关于如何在Consul中启用Serf的gossip加密的细节，请看这里的[文档](https://www.consul.io/docs/agent/encryption)。 

RPC系统支持使用端到端TLS，并可选择客户端认证。TLS是一种广泛部署的非对称加密系统，是Web上安全的基础。 

这意味着Consul的通信是受保护的，可以防止窃听，篡改和欺骗。这使得Consul可以在不受信任的网络上运行，如EC2和其他共享主机提供商。

## 安全配置

Consul[威胁模型](https://baike.baidu.com/item/threat%20modeling)只适用于Consul在安全配置下运行的情况。在默认安全配置下不会运行。如果下面的任何设置没有启用，那么这个威胁模型的部分内容将不会生效。对于Consul威胁模型之外的项目，还必须采取额外的安全预防措施，如下文各节所述。

* **Consul的运行就像任何其他二进制文件一样。**Consul作为一个单一的进程运行，并遵守与您系统上任何其他应用程序相同的安全要求。Consul不会与主机系统交互，以任何方式改变或操纵安全参数。根据您的操作系统，采取您通常对单个进程采取的任何预防措施或补救步骤。下文概述了一些您可以采取的修复步骤示例。 
  * 以非root用户的身份运行应用程序，包括Consul，并进行适当的配置。 
  * 使用内核安全模块（如SELinux）实现强制访问控制。
  *  防止无权用户成为root用户。 
* **启用ACLs\(默认拒绝\)。**必须将Consul配置为使用允许列表（默认拒绝）方式的ACL。这将迫使所有请求具有明确的匿名访问或提供ACL令牌。 
* **启用加密。**必须启用并配置TCP和UDP加密，以防止Consul代理之间的明文通信。至少应该启用`verify_outgoing`，以验证服务器的真实性，每个服务器都有一个唯一的TLS证书。`verify_server_hostname`也是必需的，以防止被入侵的代理重新启动为服务器，并获得所有密钥。 `verify_incoming`通过[相互认证](https://learn.akamai.com/en-us/webhelp/iot/internet-of-things-over-the-air-user-guide/GUID-21EC6B74-28C8-4CE1-980E-D5EE57AD9653.html)提供了额外的代理验证，但并不是严格意义上强制执行威胁模型所必需的，因为请求也必须包含有效的ACL令牌。微妙的是，目前`verify_incoming = false`将允许服务器仍然接受来自客户端的未加密连接（以允许逐步推出TLS）。这本身并不违反威胁模型，但任何选择不使用TLS的错误配置的客户端都会违反模型。我们建议将此设置为true。如果将其设置为false，则必须注意确保所有的consul客户端使用`verify_outgoing = true`，如上所述，但同时所有外部API/UI访问必须通过HTTPS，并禁用HTTP监听器。

### 已知非安全配置

除了配置上述非默认设置外，Consul还有几个非默认选项可能会带来额外的安全风险。

*  **启用网络暴露的API的脚本检查。**如果Consul代理（Client或Server模式）将其HTTP API暴露在localhost以外的网络上，`enable_script_checks`必须为false，否则，即使配置了ACL，脚本检查也会带来远程代码执行的威胁。如果HTTP API必须暴露，`enable_local_script_checks`提供了一个安全的替代方案，并且从1.3.0开始就可以使用。这个功能也被回移植到0.9.4、1.1.1和1.2.4补丁版本中，如[这里](https://www.hashicorp.com/blog/protecting-consul-from-rce-risk-in-specific-configurations)所述。 
* **启用远程执行。**Consul 包含了一个 `consul exec` [功能](https://www.consul.io/commands/exec)，允许跨集群执行任意命令。从0.8.0开始，这个功能默认是禁用的。我们建议将其禁用。如果启用，必须非常小心，以确保正确的ACL限制访问，例如，任何管理令牌授予在集群上执行任意代码的访问权限。 
* **验证服务器主机名单独使用。**从0.5.1版本到1.4.0版本，我们记录了`verify_server_hostname`为`true`意味着`verify_outgoing`，但是由于一个bug，情况并非如此，所以只设置`verify_server_hostname`会导致客户端和服务器之间的纯文本通信。更多细节请参见[CVE-2018-19653](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-19653)。这个问题在1.4.1中得到了修复。

## 威胁模型

以下是Consul威胁模型的部分内容。 

* **Consul Agent与Agent之间的通信。**Consul代理之间的通信应该是安全的，不会被窃听。这需要在集群上启用传输加密，并涵盖TCP和UDP流量。 
* **Consul Agent到CA的通信**。Consul Server与为Connect配置的证书授权提供商之间的通信始终是加密的。 
* **传输中的数据被篡改。**任何篡改都应该可以检测到，并导致Consul避免处理请求。
*  **未经认证或授权而访问数据。**所有请求必须经过认证和授权。这需要在集群上启用ACL，并采用默认的拒绝模式。
*  **恶意消息导致的状态修改或损坏。**不良格式的消息会被丢弃，格式良好的消息需要认证和授权。
*  **非服务器成员访问原始数据。**所有服务器必须加入群集（具有适当的身份验证和授权）才能开始参与Raft。Raft数据是通过TLS传输的。 
* **针对节点的拒绝服务**。针对节点的DoS攻击不应损害软件的安全立场。 
* **基于Connect的服务对服务通信。**两个支持Connect的服务之间的通信（原生或代理）应该是安全的，可以防止窃听并提供认证。这是通过相互的TLS实现的。 

以下不属于Consul Server Agent的Consul威胁模型。 

* **访问（读或写）Consul数据目录。**所有Consul Server，包括Non-Leaders，都会将Consul的全套状态持久化到这个目录\(catalog\)中。数据包括所有KV、服务注册、ACL令牌、Connect CA配置等。对这个目录的任何读写都允许攻击者访问和篡改这些数据。 
* **访问（读或写）Consul配置Catalog。**Consul配置可以启用或禁用ACL系统，修改数据目录路径等。对该目录的任何读或写都允许攻击者重新配置Consul的许多方面。通过禁用ACL系统，可能会让攻击者访问所有Consul数据。
*  **对运行中的Consul Server Agent的内存访问。**如果攻击者能够检查正在运行的Consul服务器代理的内存状态，几乎所有Consul数据的保密性都可能被破坏。如果你使用的是外部的Connect CA，Consul进程永远无法获得根私钥材料，可以认为是安全的。服务Connect TLS证书应该被认为是妥协的；它们从不被服务器代理持久化，但至少在Sign请求的持续时间内确实存在于内存中。 

以下不属于Consul客户端代理的Consul威胁模型。

*  **访问（读或写）Consul数据目录。**Consul客户端将使用数据目录来缓存本地状态。这包括本地服务、相关ACL令牌、Connect TLS证书等。对该目录的读或写访问将允许攻击者访问这些数据。这些数据通常是集群完整数据的一个较小的子集。 
* **访问（读或写）Consul配置目录。**Consul客户端配置文件包含服务的地址和端口信息、代理的默认ACL令牌等。访问Consul配置可以使攻击者将服务的端口改为恶意端口，注册新的服务等。此外，一些服务定义还附加了ACL令牌，可以在整个集群范围内用来冒充该服务。攻击者无法改变集群范围内的配置，如禁用ACL系统。
*  **对运行中的Consul客户端代理进行内存访问。**这一点的爆炸半径比服务器代理小得多，但数据子集的保密性还是会被泄露。特别是针对代理的API请求的任何数据，包括服务、KV和连接信息都可能被泄露。如果服务器上的某一组数据从未被代理请求过，那么它永远不会进入代理的内存，因为复制只存在于服务器之间。攻击者也有可能提取用于在该代理上注册服务的ACL令牌，因为这些令牌必须与注册服务一起存储在内存中。 
* **对本地Connect代理或服务的网络访问。**服务与 Connect 感知的代理之间的通信通常是未加密的，必须通过受信任的网络进行。这通常是一个环回设备。这就要求同一台机器上的其他进程是受信任的，或者使用更复杂的隔离机制，如网络命名空间。这也要求外部进程不能与Connect服务或代理进行通信（除了入站端口）。因此，非本地的Connect应用应该只绑定到非公共地址。
*  **不正确执行的Connect代理或服务。**Connect  Proxy或本机集成服务必须正确地提供有效的叶子证书，验证入站 TLS 客户端证书，并调用 Consul 代理-本地授权端点。如果其中任何一项没有正确执行，代理或服务可能允许未经认证或未经授权的连接。

## 外部威胁概览

影响Consul威胁模型的有四个组件：Server Agent、Client Agent、Connect CA和Consul API客户端（包括Connect的代理）。 

Server Agent通过Raft协议参与领导者选举和数据复制。与其他代理的所有通信都是加密的。数据在静止时未加密地存储在配置的数据目录中。存储的数据包括ACL令牌和TLS证书。如果 Connect 使用内置 CA，根证书私钥也会存储在磁盘上。外部CA提供商不在此目录中存储数据。这个数据目录必须小心保护，以防止攻击者冒充服务器或特定ACL用户。我们计划随着时间的推移，对数据目录引入进一步的缓解措施（至少包括部分数据加密），但数据目录应始终被视为秘密。 

客户端代理要加入集群，必须提供具有节点写能力的有效ACL令牌。Client和 Server代理之间的加入请求和所有其他API请求都通过TLS进行通信。客户端服务于Consul API，并通过共享的TLS连接将所有请求转发给Server Agent。每个请求都包含一个ACL令牌，用于认证和授权。没有提供ACL令牌的请求将继承Agent配置的默认ACL令牌。

 Connect CA提供商负责存储用于签署和验证通过Connect建立的连接的根（或中间）证书的私钥。Consul服务器代理通过加密方法与CA提供者通信。这种方法取决于所使用的CA提供者。Consul提供了一个内置的CA，它在Server Agent的本地执行所有操作。除了内置CA，Consul本身不存储任何私钥信息。

 Consul API客户端（代理本身、内置UI、外部软件）必须通过TLS与Consul代理进行通信，并且每次请求都必须提供一个ACL令牌进行认证和授权。

## 网络端口

关于配置网络规则以支持Consul，请参见[Ports Used](https://www.consul.io/docs/agent/options#ports)，查看Consul使用的网络端口列表，以及它们用于哪些功能的详细信息。

