# 访问控制（ACLs）

Consul使用访问控制列表\(ACL\)来保护UI、API、CLI、服务通信和代理（Agent）通信的安全。ACL的核心是通过将规则分组到策略中，然后将一个或多个策略与令牌关联起来来操作。

接的文档将帮助你理解和实现ACLS

## ACL文档

### ACL系统

Consul提供了一个可选的访问控制列表（ACL）系统，可用于控制对数据和API的访问。ACL系统是一个基于能力的系统，它依赖于令牌，令牌可以应用细粒度的规则。[ACL系统文档](https://www.consul.io/docs/security/acl/acl-system)详细介绍了Consul ACL的功能。

### ACL规则

ACL系统的核心部分是规则语言，它用于描述必须执行的策略。阅读[ACL规则文档](https://www.consul.io/docs/security/acl/acl-rules)，了解规则规范。

### ACL鉴权方法

Auth方法是Consul中的一个组件，它针对受信任的外部方执行验证，以授权创建可在本地数据中心内使用的ACL令牌。阅读ACL [auth方法文档](https://www.consul.io/docs/acl/auth-methods)，了解更多关于它们的工作方式以及为什么您可能想要使用它们。

### ACL旧版系统

Consul 1.3.1及更老版本的ACL系统现在被称为遗留系统。关于遗留系统的引导、ACL规则和ACL系统总体概述的信息，请阅读遗留[系统文档](https://www.consul.io/docs/acl/acl-legacy)。

### ACL 迁移

[迁移文档](https://www.consul.io/docs/acl/acl-migrate-tokens)详细介绍了升级到1.4.0后如何升级现有的遗留令牌。它将简要描述发生了什么变化，然后逐步介绍高级迁移过程选项，最后给出一些特定的迁移策略示例。新的ACL系统对ACL令牌和策略的安全性和管理有改进。

## 学习ACL引导

注意：以下指南位于HashiCorp Learn。通过选择它，您将被跳转到一个新站点。

### 通过ACL安全访问

在本指南中，您将学习如何使用ACL保护UI，API，CLI，服务通信和代理通信。 保护群集时，应首先配置ACL。 ACL文档介绍了ACL系统的基本概念和语法，建议您在开始本[教程](https://learn.hashicorp.com/tutorials/consul/access-control-setup-production)之前先阅读ACL系统。

