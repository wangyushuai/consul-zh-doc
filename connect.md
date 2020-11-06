# 服务网格

Consul Connect使用相互传输层安全\(TLS\)提供服务到服务的连接授权和加密。应用程序可以在服务网状结构中使用sidecar代理，为入站和出站连接建立TLS连接，而完全不需要知道Connect。应用程序还可以与Connect原生集成，以获得最佳性能和安全性。Connect可以帮助您保护服务的安全，并提供有关服务到服务通信的数据。

观看下面的视频，向HashiCorp的联合创始人Armon了解更多关于Consul Connect的信息。

请点击此链接查看视频：  [Consul Connect](https://www.youtube.com/watch?v=8T8t4-hQY74)

![](.gitbook/assets/image%20%281%29.png)

## 应用安全

_PS:_  [_intentions_]() _可以理解为consul控制面板配置得路由规则_

Connect通过自动服务到服务的加密和基于身份的授权实现了安全部署的最佳实践。Connect 使用注册的服务身份（而不是 IP 地址）来执行 [intentions]()的访问控制。这使得访问控制更容易推理，并使服务能够被包括Kubernetes和Nomad在内的协调器重新安排。 [intentions]()执行与网络无关，因此Connect可与物理网络、云网络、软件定义网络、跨云等协同工作。

## 可观测性

Consul Connect的主要优势之一是它可以为您网络上的所有服务提供统一和一致的视图，无论它们的编程语言和框架如何。当您将Consul Connect配置为使用sidecar代理时，这些代理可以 "看到 "所有服务到服务的流量，并收集相关数据。Consul Connect可以配置Envoy代理去收集7层Metrics，并将其导出到Prometheus等工具。经过正确检测的应用程序也可以通过Envoy发送OpenTracing数据。

## 快速Connect

有几种方法可以在不同的环境中尝试Connect。

* [快速入门Consul服务网格](https://learn.hashicorp.com/tutorials/consul/service-mesh?utm_source=WEBSITE&utm_medium=WEB_IO&utm_offer=ARTICLE_PAGE&utm_content=DOCS)指导您使用Helm Chart 在Kubernetes中安装Consul 服务网格，在服务网格中部署服务，以及使用 [intentions]()来确保服务通信。 
* [服务间安全通信教程](https://learn.hashicorp.com/tutorials/consul/service-mesh-with-envoy-proxy?utm_source=WEBSITE&utm_medium=WEB_IO&utm_offer=ARTICLE_PAGE&utm_content=DOCS)同时新将帮助我们， 使用Consul Connect的内置代理连接本地机器上的两个服务，并配置你的第一个 [intentions]()。该指南还包括使用Envoy作为Connect sidecar代理的介绍。 
* [Kubernetes教程](https://learn.hashicorp.com/tutorials/consul/kubernetes-minikube?utm_source=WEBSITE&utm_medium=WEB_IO&utm_offer=ARTICLE_PAGE&utm_content=DOCS)引导您使用Helm Chart在Kubernetes中配置Consul Connect，并使用[intention]()。您可以在Minikube或现有的Kubernetes集群上运行该指南。 
* [可观察性教程](https://learn.hashicorp.com/tutorials/consul/kubernetes-layer7-observability?utm_source=WEBSITE&utm_medium=WEB_IO&utm_offer=ARTICLE_PAGE&utm_content=DOCS)展示了如何在Kubernetes集群或者Minikube中部署一个收集metric并可视化流水线，通过用Consul，Prometheus, Grafana 这些软件得官方Helm charts。 

