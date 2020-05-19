## Istio 安全(概念)(istio 系列四)

[TOC]

通过将一个单一应用划分为多个原子服务的方式，提供了更好的灵活性，可扩展性以及重用服务的能力。然而微服务对安全有特殊的要求：

- 抵御中间人攻击，需要用到流量加密
- 提供灵活的服务访问控制，需要用到TLS和细粒度访问策略
- 决定哪些人在哪些时间可以做哪些事，需要用到审计工具

为了解决这些问题，istio提供了完整的安全方案。本章节全面介绍了如何使用istio进行服务的安全加固(无论该服务运行在哪里)。特别地，Istio安全性可减轻来自内部和外部的(对数据，终端，通信和平台的)威胁。

![](./images/istio security.png)

Istio安全特性提供了强大的身份、策略、透明TLS加密，以及认证，授权和审计(AAA)等功能来对服务和数据进行防护。istio安全的目标是：

- 默认安全：不需要对应用代码和基础架构进行任何改变
- 深度防护：与现有的安全系统结合，来提供多个层面的防护
- 0-信任网络：在不信任的网络上构建安全解决方案

查看[multual TLS迁移文档](https://istio.io/docs/tasks/security/authentication/mtls-migration/)，在已部署的服务上使用istio安全特性。通过[安全任务](https://istio.io/docs/tasks/security/)来连接安全特性的细节。

### 高层架构

istio的安全涉及多个组件：

- 用于密钥和证书管理的证书颁发机构（CA）
- 分发给代理的API server配置：
  - [认证策略](https://istio.io/docs/concepts/security/#authentication-policies)
  - [授权策略](https://istio.io/docs/concepts/security/#authorization-policies)
  - [安全命名信息](https://istio.io/docs/concepts/security/#secure-naming)
- Sidecar和外围代理充当策略执行点（[pep](https://www.jerichosystems.com/technology/glossaryterms/policy_enforcement_point.html)），确保客户端和服务器之间的通信安全。
- 通过一系列Envoy扩展来管理遥测和审计

控制面负责接收来自API server的配置信息，并在数据面配置PEP。PEP使用Envoy实现。架构如下：

![](./images/istio security2.png)

### Istio身份

身份是任何安全设施的最基本概念。在工作负载到工作负载的通信开始时，双方必须交换携带各自身份信息的凭据，来进行双向认证。客户端会将服务的身份与安全命名信息进行比对，来查看该服务是否是授权的工作负载运行器；服务端会根据[授权策略](https://istio.io/docs/concepts/security/#authorization-policies)来决定客户端可以访问的内容，审计记录谁在什么时间访问了什么内容，根据负载控制客户端的行为，以及拒绝没有支付访问的负载的客户端。

istio身份模型使用一级服务标识(`service identity` )来确定请求源的身份。该模型使用更大的灵活性和颗粒度来标识一个用户，单独的负载，或一组负载。在没有服务标识的平台上，isito可以使用其他标识来对负载实例进行分组，如服务名称。

下面展示了不同平台上可以使用的服务标识：

- Kubernetes: Kubernetes service account
- GKE/GCE: GCP service account
- GCP: GCP service account
- AWS: AWS IAM user/role account
- On-premises (non-Kubernetes): user account, custom service account, service name, Istio service account, or GCP service account. The custom service account refers to the existing service account just like the identities that the customer’s Identity Directory manages.

### 身份和证书管理

istio安全使用X.509证书为每个负载提高了强身份信息。每个Envoy代理旁都会运行一个istio agent，istio agent与`istiod`配合，可以在扩展时实现证书的自动滚动。下面展示了证书配置流程：

![](./images/istio security3.png)

isito通过secert发现(SDS)机制来处理身份认证，处理过程为：

- istiod提供了一个gRPC服务来处理CSR(证书签名请求)
- Envoy通过Envoy SDS API发送证书和密钥请求
- 在接收到SDS请求后，istio agent会创建私钥和(在将携带凭据的CSR发送`istiod`前)CSR
- CA会校验CSR携带的凭据，并签发CSR来生成证书
- istio agent通过Envoy SDS API将来自isitod的证书和密钥发送给Envoy
- 周期性地执行如上CSR处理流程来滚动证书和密钥

### 认证

isito提供两种类型的认证：

- 对等体认证：用于服务到服务的身份验证，验证建立连接的客户端。Istio提供了mutual TLS作为传输身份验证的全栈解决方案，这种方式无需修改服务代码，该方案：
  - 为每个服务提供了强标识作为角色(role)，来支持跨集群和云的互操作性。
  - 确保服务到服务的通信安全。
  - 提供了密钥管理系统来自动生成，分发，滚动密钥和证书。
- 请求认证：用于终端用户认证，校验请求中的凭据。Istio请求级认证使用了JSON Web Token(JWT)验证，以及基于自定义身份验证或OpenID Connect开发的程序，如：
  - [ORY Hydra](https://www.ory.sh/)
  - [Keycloak](https://www.keycloak.org/)
  - [Auth0](https://auth0.com/)
  - [Firebase Auth](https://firebase.google.com/docs/auth/)
  - [Google Auth](https://developers.google.com/identity/protocols/OpenIDConnect)

在所有场景中，istio通过一个用户自定义的Kubernetes API将认证策略保存在`Istio config store` 中。istiod会将这些策略更新到每个代理中，并提供合适的密钥。此外，istio支持宽容模式下的身份验证，可以帮助理解一个策略在强制执行前如何影响安全状态。

#### Mutial TLS认证

Istio通过客户端和服务器端的PEP实现服务到服务的通信，PEP作为[Envoy代理](https://www.envoyproxy.io/docs/envoy/latest/)来实现。当一个负载使用mutual TLS认证向另一个负载发送请求时，该请求的处理流程如下：

1. isito将出站流量从客户端重路由到客户端的本地sidecar Envoy中
2. 客户端侧的Envoy与服务端侧的Envoy开始双向TLS握手。在握手期间，客户端侧的Envoy也会进行[安全命名](https://istio.io/docs/concepts/security/#secure-naming)校验，确保可以授权服务端证书中的service account来执行目标服务。
3. 客户端侧的Envoy和服务端侧的Envoy建立双向TLS连接，istio会将流量从客户端的Envoy转发的服务端侧的Envoy
4. 在授权后，服务端测的Envoy会通过本地TCP连接将流量转发的服务端的服务中。

#### 宽容模式

isitl mutual TLS有一个宽容模式，它允许一个服务同时接收明文流量和TLS加密的流量。该特性极大提升了mutual TLS的使用体验。

在很多非istio的客户端和非istio的服务端架构中，当计划将服务端迁移到启用mutual TLS的istio上时都会遇到问题。通常，操作人员不能同时给所有的客户端安装一个sidecar，或没有权限这么做。即使在所有的服务端安装istio sidecar后，操作人员仍然无法在不中断现有连接的情况下启用mutual TLS。

使用宽容模式时，服务端可以同时接收明文和mutual TLS的流量。该模式极大提升了使用istio的灵活性。服务端在安装istio sidecar后，也可以在不中断现有明文流量的情况下接收mutual TLS流量。这样，就可以通过逐步安装并配置客户端的istio sidecar来发送mutual TLS流量。一旦完成客户端的配置，操作人员就可以将服务端配置为仅mutual TLS模式。更多信息，参见[mutual TLS迁移指南](https://istio.io/docs/tasks/security/authentication/mtls-migration/)。

#### 安全命名













