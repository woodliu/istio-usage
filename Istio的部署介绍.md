# Istio的部署介绍

[TOC]

## 部署模型

当配置一个生产级别的Istio时，需要解决一些问题：如网格是单集群使用，还是跨集群使用？所有的服务会放到一个完全可达的网络中，还是需要网关来连接跨网络的服务？使用一个控制面(可能会跨集群共享一个控制面)，还是使用多个控制面来实现高可用(HA)？所有的集群都连接到一个多集群服务网格，还是联合成一个多网格形态。

所有这些问题，以及他外因素，都代表了Istio部署的各个配置维度。

1. 单集群或多集群
2. 单网络或多网络
3. 单控制面或多控制面
4. 单网格或多网格

上述条件可以任意组合(虽然有些组合相比其他更加普遍，有些则不那么受欢迎，如单集群中的多网格场景)。

在一个涉及多集群的生产环境中，可以混合使用部署模型。例如，可以使用多个控制面来做到HA。在一个3集群环境中，可以将两个集群共享一个控制面，然后给第三个集群在不同的网络中添加另外一个控制面。然后配置三个集群共享各自的控制面，这样所有的集群就可以使用2个控制面来做到HA。

实际使用中，需要根据隔离性，性能，以及HA要求来选择合适的部署模型。本章将描述部署Istio时的各种选择和考量。

### 集群模式

该模式下，应用的负载会运行在多个集群中。为了隔离，性能或高可用，可以限定集群的可用zone和region。

取决于具体要求，生产系统可以跨多个zone或region的多个集群运行，利用云负载均衡器来处理诸如本地性，zonal 或regional故障转移之类的事情。

大多数场景下，集群表示配置和终端发现的边界。例如，每个kubernetes集群都有一个API Server来管理集群的配置，以及提供服务终端的信息，如pod的启停等。由于kubernetes的这种配置行为是基于单个集群的，因此会将潜在的错误(如配置错误)限制在其所在的集群中。

使用Istio可以在任意个集群上配置一个服务网格。

#### 单集群

在最简单的场景中，可以在单个集群中部署单Istio网格。一个集群通常会运行在一个独立的网络中，但具体取决于基础设施提供商。包含一个网络的单集群模型会包含一个控制面，这就是istio最简单的部署模型：

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903164523228-666196152.png)

单集群的部署比较简单，但同时也缺少一些特性，如故障隔离和转移。如果需要高可用，则应该使用多集群模式。

#### 多集群

一个网格可以包含多个集群。使用多集群部署可以在一个网格中提供如下功能。

- 故障隔离和转移：当`cluster-1`宕机后，使用`cluster-2`
- 位置感知路由和故障转移：发送请求到最近的服务
- 多种控制面模型：支持不同级别的可用性。
- 团队和项目隔离：每个团队运行各自的集群

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903165132776-1823623448.png)

多集群部署提供了更大的隔离性和可靠性，但同时也增加了复杂性。如果系统需要高可用，则集群可能会跨多个zone和region。

可以通过金丝雀配置来在单个集群中修改或使用新的二进制版本，这样配置变更仅会影响一小部分的用户流量。如果一个集群出现问题，可以临时把流量路由到临近的集群(直到问题解决)。

此外，可以根据[网络](https://preliminary.istio.io/latest/docs/ops/deployment/deployment-models/#network-models)和云提供商支持的选项配置集群间的通信。例如，如果两个集群使用了相同的底层网络，那么就可以通过简单的防火墙规则实现集群间的通信。

### 网络模型

很多生产系统需要多个网络或子网来实现隔离和高可用。Istio支持将一个服务网格部署到多种类型的网络拓扑中。通过这种方式选择符合现有网络拓扑的网络模型。

#### 单网络

在最简单场景下，服务网格会运行在一个完全连接的网络上，在单网络模型下，所有的负载示例能够在没有Istio网格的情况下实现互联。

单个网络使Istio能够在网格中以统一的方式配置服务使用者，并具有直接处理工作负载实例的能力。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903213949647-1416187096.png)

#### 多网络

多网络提供了如下新功能：

- 为**service endpoints**提供了可交叉的IP或VIP
- 跨边界
- 容错功能
- 扩展网络地址
- 符合网络分段的标准

这种模式下，不同网络中的负载实例只能通过一个或多个Istio网关进行互联。Istio使用分区服务发现来为使用者提供service endpoints的不同视图。视图取决于使用者的网络。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903214421317-27025543.png)

### 控制面模型

Istio网格使用控制面来配置网格内负载实例之间的通信。通过复制控制面，工作负载可以连接到任何控制面来获取配置。

最简单的场景下，可在单集群中运行网格的控制面：

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903214543902-750944395.png)

像这样具有自己的本地控制平面的集群被称为主集群。

多集群部署可以共享相同的控制面实例。这种情况下，控制面实例可以位于一个或多个主集群中。没有自己的控制面的集群称为远端集群。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903215105479-1899115394.png)



一个完全由远程集群组成的服务网格可以通过外部控制面进行控制，而不需要通过运行在主集群中的控制面进行控制。通过这种方式将istio在管理上进行了隔离，即将控制面和数据面服务进行了完全分割。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200904101212802-1315864133.png)

> 云提供商的`Managed Control Plane`实际上就是一个外部控制面。

为了高可用，控制面可能需要跨多集群，zone和region来部署。下图中的服务网格在每个region中都使用了一个控制面。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200904101412059-1243762490.png)

使用上述模型具有如下优势：

- 提升可用性：如果一个控制面不可用，停机的范围将仅限于该控制平面
- 配置隔离：可以修改一个集群，zone或region中的配置，而不会影响另外一个控制面的配置。

可以通过故障转移来提升控制面的可靠性。当一个控制面实例变为不可用时，负载实例可以连接到其他可用的控制面实例上。集群，zone和region中都可能发生故障迁移。下图可以看到，当左侧的控制面出问题后，由右侧的控制面托管了Envoy代理(虚线)。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200904101852071-1028807398.png)

以下列表按可用性对控制平面的部署进行了排序：

- 每个region一个集群(低可用性)
- 每个region多集群
- 一个zone一个集群
- 一个zone多个集群
- 每个集群一个控制面(高可用性)

### 身份和信任模型

当在一个服务网格中创建负载实例时，istio会赋予该负载一个身份。

CA( *certificate authority*)会为网格中使用的身份创建和颁发证书，后续就可以使用CA为该身份创建和颁发证书时使用的公钥对消息发送端进行校验。**trust bundle**是Istio网格使用的所有CA公钥的**集合**。任何人都可以使用网格的trust bundle校验来自网格的服务。

#### 网格中的信任

在单istio网格中，istio保证每个负载实例都有一个标识其身份的证书，使用trust bundle来识别网格和联邦网格中的所有身份。CA仅负责为这些身份创建和签发证书。该模型允许网格中的负载实例互联。下图展示了一个具有 *certificate authority*的服务网格。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200904110515914-1587188188.png)

#### 网格之间的信任

如果一个网格中的服务需要调用另外一个网格中的服务，此时需要在两个网格之间使用联邦身份。为了实现联邦身份的信任，需要交换网格的trust buncles(可以通过手动或自动方式，如[SPIFFE Trust Domain Federation](https://docs.google.com/document/d/1OC9nI2W04oghhbEDJpKdIUIw-G23YzWeHZxwGLIkB8k/edit)来交换trust bundles)

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200904110859157-2032885826.png)

### 网格模型

Istio支持在一个网格或联邦网格(也被称为多网格)中部署所有的应用。

#### 单网格

单网格是部署Istio的最简单的方式。在一个网格中，服务名是唯一的。例如，一个`foo`命名空间中，只能存在一个名为`mysvc`的服务。此外，负载实例共享同一个身份service account，因为该资源在命名空间内是唯一的。

单网格可以部署在一个或多个集群中，以及一个或多个网络上。在一个网格中，命名空间用于tenancy(见下)。

#### 多网格

多网格是网格联邦的结果。

多网格提供了如下功能：

- 组织边界：业务线
- 重用服务名称或命名空间：例如`default`命名空间可以用于多种用途
- 强隔离：将测试负载和生产负载进行隔离。

可以使用中间网格来连接网格联邦。当使用联邦时，每个网格都可以暴露一组所有参与的网格都能识别的服务和身份。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200904112820376-963354772.png)

为了避免服务名冲突，可以给每个网格分配一个全局唯一的网格ID，来保证每个服务的fully qualified domain name (FQDN)是有效的。

当联邦中两个网格没有使用相同的信任域时，必须对这两个网格的身份和**trust bundles**进行联邦。查看[Multiple Trust Domains](https://preliminary.istio.io/latest/docs/ops/deployment/deployment-models/#trust-between-meshes)。

### 租户模式

在Istio中，租户是一个共享用户组，共享一组已部署的工作负载的访问权限和特权。通常需要从网络配置和策略层面来为不同的租户隔离负载实例。

可以配置租户模式来满足组织的隔离需求：

- 安全
- 策略
- 容量
- 成本
- 性能

Istio支持两种类型的租户：

- [Namespace tenancy](https://preliminary.istio.io/latest/docs/ops/deployment/deployment-models/#namespace-tenancy)
- [Cluster tenancy](https://preliminary.istio.io/latest/docs/ops/deployment/deployment-models/#cluster-tenancy)

#### Namespace tenancy

在一个网格中，Istio使用命名空间作为租户的单位。Istio也可以运行在没有实现命名空间租户的环境中。在实现命名空间租户的环境中，可以保证仅允许一个团队将负载部署在一个给定的命名空间或一组命名空间中。默认情况下，多个租户命名空间中的服务都可以互联。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200904131957636-354941125.png)

为了提升隔离性，可以选择暴露到其他命名空间中的服务。通过授权策略来暴露服务或限制访问。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200904132152788-496424085.png)

当使用多集群时，每个集群中的相同名称的命名空间被看作是相同的命名空间。例如，`cluster-1`中的`foo`命名空间中的`Service B`，以及`cluster-2`中的`foo`命名空间中的`Service B`被认为是相同的服务，且Istio会在服务发现时合并endpoints，并在这些endpoints间执行负载均衡。下图展示了具有相同命名空间的两个集群

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200904132604677-1058821779.png)

#### Cluster tenancy

Istio支持以集群作为租户的单位。这种情况下，可以给每个团队指定特定的集群或一组集群来部署负载，并授权团队成员。可以给成员分配不同的角色，如：

- 集群管理员
- 开发人员

为了在Istio中使用集群租户，需要将每个集群作为独立的网格。此外，可以使用Istio将一组集群作为单租户。这样每个团队就可以拥有一个或多个集群，而不是将所有的集群配置为单网格。为了连接不同团队的网格，可以将这些将这些网格联邦为多网格。下图展示了使用两个集群和命名空间来隔离服务网格。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200904132904974-1657499887.png)

由于网格由不同的团队或组织进行操作，因此服务命名很少会不同。例如，`cluster-1`的`foo`命名空间中的`mysvc`和`cluster-2`的`foo`命名空间中的`mysvc`服务并不是相同的服务。最常见的示例是Kubernetes中的场景，许多团队将工作负载部署到`default`命名空间。

当每个团队都有各自的网格后，就可以使用多网格模型跨网格进行通信。

## 性能和可靠性

Istio使用丰富的路由，负载均衡，服务到服务的认证，监控等简化了部署服务的网络，且不需要修改应用代码。Istio力争使用最少的资源开销来提供这些便利，以及在增加最小的延迟下支撑更大规模的网格和更高的请求率。

Istio数据面组件，Envoy代理会处理流经系统的数据。Istio的控制面组件，Pilot，Galley和Citadel来配置数据面。数据面和控制面都有明显的性能问题。

### 1.7的性能摘要

[Istio负载测试](https://github.com/istio/tools/tree/release-1.7/perf/load)网格包含1000个服务和2000个sidecar，每秒70000个网格范围的请求。在对Istio1.7测试之后，得出如下结果：

- 在每秒有1000个请求经过代理时，Envoy代理会使用**0.5 vCPU** 和**50 MB**内存
- 如果部署时使用了Mixer，在每秒有1000个网格范围的请求时，`istio-telemetry`服务会使用**0.6 vCPU**
- Pilot使用**1 vCPU** 和1.5 GB内存
- Envoy代理对90％的延迟增加了3.12毫秒。

### 控制面性能

Pilot会根据用户的配置文件和系统的当前状态配置sidecar代理。在Kubernetes环境中，CRD和deployment构成了配置和系统状态。Istio配置对象，如gateway和virtual service等提供了用户可编辑的配置。为了生成代理的配置，Pilot处理来自Kubernetes环境和用户配置的组合配置以及系统状态。

控制平面支持成千上万的服务，这些服务分布在成千上万个Pod中，并使用类似数量的由用户编写的virtual services和其他配置对象。Pilot的CPU和内存会随着配置数据的变化和系统状态而变化。CPU的使用包含如下因素：

- deployment变化率
- 配置变化率
- 连接到Pilot的代理数目

但这部分是支持水平扩展的。

**当启用命名空间租户时，使用1vCPU和1.5GB内存的单个Pilot可以支持1000个服务，2000个sidecar。可以通过增加Pilot的数量来降低配置所有代理的时间。**

### 数据面性能

数据面性能依赖很多因素，如：

- 客户端连接数
- 目标请求率
- 请求大小和响应大小
- 代理worker的线程数
- 协议
- CPU cores
- 代理过滤器的数目和类型，特别是Mixer的过滤器

延迟，吞吐量以及代理的CPU和内存消耗均根据上述因素进行衡量。

#### CPU和内存

由于sidecar代理会在数据路径上做一些额外的工作，因此会消耗CPU和内存。如Istio 1.1中，在每秒1000个请求的情况下，一个代理会消耗0.6 vCPU。

代理的内存消耗取决于代理保存的总配置状态。大量listeners，clusters和routes可能会增加内存消耗。Istio 1.1引入了命名空间隔离来限制发送到一个代理的配置。在一个大的命名空间中，一个代理大概会消耗50 MB的内存。

由于代理通常不会缓存通过的数据，因此请求率不会影响内存消耗。

> 命名空间的隔离是通过[sidecar](https://preliminary.istio.io/latest/docs/reference/config/networking/sidecar/)资源实现的。如何使用可以参见[istio-namespace-isolation-tricks](https://www.funkypenguin.co.nz/note/istio-namespace-isolation-tricks/)。

#### 延迟

由于Istio在数据路径上注入了sidecar代理，因此延迟是一个需要重点考虑的因素。Istio会将身份验证和Mixer过滤器添加到代理，每个额外的过滤器都会增加代理内部的路径长度，并影响到延迟。

Envoy代理会收在客户端接收到响应之后采集原始的遥测数据。对采集请求的遥测数据的时间不会计算在总的完成请求所需要的时间内。但由于worker忙于处理请求，因此worker可能不会立即处理下一个请求。这种处理会增加请求在队列中等待的时间，并影响到平均值和尾部延迟。实际的尾部延迟取决于流量状况。

#### Istio 1.7的延迟

在网格中，请求会通过客户端的代理，然后到达服务端。1.7的默认配置中(即，telemetry V2)，这两个代理在基准数据平面延迟的90百分位和99百分位延迟上分别增加了大约3.12ms 和3.13ms。通过[Istio benchmarks](https://github.com/istio/tools/tree/release-1.7/perf/benchmark)的`http/1.1`协议得出如上结论，使用两个代理worker，并启用mutual TLS，通过16个客户端连接来发送每秒1000个请求，每个请求1KB。

在即将发布的Istio版本中，将把Istio策略和Istio遥测功能作为TelemetryV2添加到代理中。通过这种方式来减少流经系统的数据量，从而减少CPU使用量和延迟。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200905200922839-153015420.png)

​															*P90 latency vs client connections without jitter*

![image-20200905201003085](C:\Users\liuchanglin\AppData\Roaming\Typora\typora-user-images\image-20200905201003085.png)

​														*P99 latency vs client connections without jitter*

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200905201045199-1049608139.png)

​																*P90 latency vs client connections with jitter*

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200905200554764-1910643038.png)

​															*P99 latency vs client connections with jitter*

- `baseline` 客户端Pod直接调用服务端Pod，不经过sidecar
- `none_both` 使用Istio代理，但不配置Istio过滤器
- `v2-stats-wasm_both` 客户端和服务端的sidecar都配置似乎用telemetry v2 `v8` 
- `v2-stats-nullvm_both` 客户端和服务端的sidecar默认都配置似乎用telemetry v2 `nullvm` 
- `v2-sd-full-nullvm_both` 使用配置的 telemetry v2 `nullvm`暴露Stackdriver  metrics，访问日志
- `v2-sd-nologging-nullvm_both` 与上面系统，但不暴露访问日志

#### Benchmarking 工具

Istio使用如下工具来进行benchmarking：

- [`fortio.org`](https://fortio.org/) - 恒定吞吐量的负载测试工具
- [`blueperf`](https://github.com/blueperf/) - 真实的云原生应用程序
- [`isotope`](https://github.com/istio/tools/tree/release-1.7/isotope) - 具有可配置拓扑的合成应用程序

## Pods 和Services

作为网格的一部分，kubernetes的pod和service必须满足如下要求：

- 关联的Service：一个pod必须对应至少一个kubernetes service，即使pod没有暴露任何端口。如果一个pod对应多个kubernetes services，那么这些services无法为不同的协议使用相同的端口号，如HTTP和TCP。
- 应用的UID：确保pods不能以用户ID (UID)值为1337的身份运行应用程序
- **`NET_ADMIN`** 和**`NET_RAW`**  **capabilities**：如果集群强制使用pod安全策略，则必须给pod添加 `NET_ADMIN` 和`NET_RAW` capabilities。如果使用了[Istio CNI 插件](https://istio.io/latest/docs/setup/additional-setup/cni/)，则可以不遵守该要求。
- **Deployments** 使用 **app** 和**version labels**：建议Deployments 使用`app`和`version` labels。`app`和`version` labels会给istio采集的metrics和遥测数据添加上下文信息
  - `app` label：每个deployment应该包含不同的`app` label。`app` label用于在分布式跟踪中添加上下文信息。
  - `version` label：指定特定deployment对应的应用版本。
- 命名service端口：可以选择使用命名服务端口来指定协议。更多细节参见[Protocol Selection](https://istio.io/latest/docs/ops/configuration/traffic-management/protocol-selection/)。

### 要求的pod capabilities

如果集群使用了[pod安全策略](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)，除非使用了Istio CNI插件，否则pod必须允许`NET_ADMIN` 和`NET_RAW` capabilities。Envoy代理的initialization容器会使用这些capabilities。

为了校验pod是否允许`NET_ADMIN` 和`NET_RAW` capabilities，需要校验pod对应的service account是否可以使用pod安全策略来允许`NET_ADMIN` 和`NET_RAW` capabilities。如果没有在pod的deployment中指定service account，那么pod会使用其所在命名空间的`default` service account。

使用如下命令可以列出一个service accout的capabilities，替换`<your namespace>` 和`<your service account>`

```shell
# for psp in $(kubectl get psp -o jsonpath="{range .items[*]}{@.metadata.name}{'\n'}{end}"); do if [ $(kubectl auth can-i use psp/$psp --as=system:serviceaccount:<your namespace>:<your service account>) = yes ]; then kubectl get psp/$psp --no-headers -o=custom-columns=NAME:.metadata.name,CAPS:.spec.allowedCapabilities; fi; done
```

例如可以使用如下命令来校验`default`命名空间中的`default` service account。

```shell
# for psp in $(kubectl get psp -o jsonpath="{range .items[*]}{@.metadata.name}{'\n'}{end}"); do if [ $(kubectl auth can-i use psp/$psp --as=system:serviceaccount:default:default) = yes ]; then kubectl get psp/$psp --no-headers -o=custom-columns=NAME:.metadata.name,CAPS:.spec.allowedCapabilities; fi; done
```

如果在capabilities列表中看到 `NET_ADMIN` 和 `NET_ADMIN` 或`*`，则说明该pod允许运行Isti init容器，否则需要[配置权限](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#authorizing-policies)。