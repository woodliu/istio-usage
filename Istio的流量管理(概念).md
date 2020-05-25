## Istio的流量管理(概念)

[TOC]

涵盖istio官方文章的[Traffic Management](https://istio.io/docs/concepts/traffic-management/)章节

### 概述

istio的流量路由规则可以简单地控制不同服务间的流量以及API调用。Istio在服务层面提供了断路器，超时，重试等功能，通过这些功能可以简单地实现A/B测试，金丝雀发布，基于百分比的流量分割等，此外还提供了开箱即用的故障恢复功能，用于增加应用的健壮性，以应对服务故障或网络故障。

istio的流量管理依赖Envoy代理，该代理作为sidecar与服务容器部署在同一个pod内，服务发送或接收的流量都会经过Envoy，这样就可以在不改变服务的情况下实现网格中的流量管理。

为了直接管理网格中的流量，istio需要了解所有的endpoints，以及哪些service对应哪些endpoints，为了将这些信息推送到它的服务注册表(service reistry)中，istio需要连接到一个服务发现系统，例如，当istio安装到一个kubernetes集群时，istio会自动探测集群的services和endpoints。

> service reistry：istio内部的服务注册表，包含了service的集合以及service对应的endpoint信息。istio使用服务注册表生成Envoy的配置信息。Pilot可以通过底层平台提供的服务发现机制实现自动注册service，也可以通过[ServiceEntry](https://istio.io/docs/concepts/traffic-management/#service-entries)手动注册service。

使用服务注册表，Envoy代理可以将流量定向到相关的服务中。大多数基于微服务的应用会有多个实例，且每个实例处理一部分服务流量，这些实例有时也被称为负载均衡池。默认情况下，Envoy代理会使用轮询模式，通过服务的负载均衡池分发流量，此时会按照顺序将请求发送到负载均衡池中的各个成员。

虽然istio的基本服务发现和负载均衡提供了一个可运行的服务网格，但istio的功能远非如此。在大多数场景下，用户可能想更好地控制网格的流量，如在A/B测试中按照百分比将流量导入一个新版本的服务，或对某些服务实例应用不同的负载均衡策略，对进出网格的流量应用特殊的规则，或将网格的外部依赖项添加到服务注册表中等。这些功能都可以通过istio的流量管理API，在istio中添加流量配置来实现。

跟其他istio配置一样，流量管理API也使用CRD指定。下面介绍各个流量管理API资源，以及这些API的功能。API资源包括：

- [Virtual services](https://istio.io/docs/concepts/traffic-management/#virtual-services)
- [Destination rules](https://istio.io/docs/concepts/traffic-management/#destination-rules)
- [Gateways](https://istio.io/docs/concepts/traffic-management/#gateways)
- [Service entries](https://istio.io/docs/concepts/traffic-management/#service-entries)
- [Sidecars](https://istio.io/docs/concepts/traffic-management/#sidecars)

### [Virtual services](https://istio.io/docs/reference/config/networking/virtual-service/#VirtualService)

[Virtual services](https://istio.io/docs/reference/config/networking/virtual-service/#VirtualService)和[destination rules](https://istio.io/docs/concepts/traffic-management/#destination-rules)是istio流量路由功能的核心部分。virtual service规定了(在用户平台提供的基本连接和服务发现的基础上)如何将一个请求路由到一个istio的服务网格中。每个virtual service包含一个按顺序处理的路由规则集，istio会通过这些规则将每个匹配virtual service的请求分发到网格中一个特定的目的地。根据需求可以使用多个virtual service或不使用virtual service。

#### 为什么使用virtual service

virtual service将客户端请求与目标负载进行解耦，通过这种方式，大大提升了istio流量管理的灵活性，并增强了流量管理的功能。virtual service还提供了大量方式来指定不同的流量路由规则，用于将流量发送到目标负载中。

如果没有virtual service，Envoy会像前面介绍的那样，在所有服务实例中采用轮询负载均衡方式进行流量分发。用户需要了解具体的负载才能做进一步动作。例如，某些负载可能代表不同版本的服务，这样就可以用于A/B测试，将不同百分比的流量分发到不同版本的服务中，或直接将流量分发到特定的服务实例。

使用virtual service后，就可以为一个或多个主机名指定流量行为，使用virtual service中的路由规则告诉Envoy如何将virtual service的流量发送到合适的目的地。路由目的地可能对应相同版本的服务或完全不同版本的服务。

一个典型的场景是将流量发送到不同版本的服务。客户端将virtual service视为一个独立的实体，并将请求发送到virtual service，Envoy会根据virtual service中的规则将流量路由到不同的版本：例如，"20%的调用分发到新版本"，或"将来自某些用户的调用分发到版本2"。通过这种方式可以实现金丝雀发布。流量路由完全独立于实例的deployment，这意味着新版本的服务实例的数量可以根据流量负载扩大或缩小，而不必参考流量路由。相比之下，像Kubernetes这样的容器编排平台只支持基于实例缩放的流量分布，这样很快就会变得复杂。

Virtual services允许：

- 通过一个virtual service处理多个应用服务。如果网格使用了kubernetes，可以通过一个virtual service处理特定命名空间下的所有服务。通过将一个virtual service映射到多个"真实的"服务，可以方便将单一应用程序转换为由不同的微服务组成的复合服务，而无需服务的使用者去适应这种转变(*可以看作是kubernetes service之上的service*)。
- 将配置的流量规则与[gataways](https://istio.io/docs/concepts/traffic-management/#gateways)进行结合来控制ingress和egress的流量。

在一些场景下还需要配置destination rule来使用这些特性，此时需要指定服务子集(`subset`)。在一个独立的对象中指定服务子集和其他destination指定的策略(*如负载均衡*)，可以在不同的virtual service中重用这些配置。

#### Virtual services举例

下面是virtual service的一个例子，它根据路由规则将请求分发到不同版本的服务。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews #虚拟service
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews #对应kubernetes的service "reviews"
        subset: v2    #对应destination rule中的服务子集
  - route:
    - destination:
        host: reviews
        subset: v3
```

##### hosts字段

`hosts`字段列出了virtual service的主机，即用户可寻址的目标或路由规则应用的目标。它是客户端向服务发送请求时使用的一个或多个地址。通过该字段提供的地址访问virtual service，进而访问后端服务。在集群内部(网格内)使用时通常与kubernetes的Service同命；当需要在集群外部(网格外)访问时，该字段为gateway请求的请求的地址，即与gateway的`hosts`字段相同，也可采用DNS的模糊匹配。

```yaml
hosts:
- reviews
```

virtual service的主机名可以是IP地址、DNS名称，也可以是短名称(例如Kubernetes服务短名称)，该名称会被隐式或显式解析为全限定域名（FQDN），具体取决于istio依赖的平台。可以使用前缀通配符（“*”）为所有匹配的服务创建一组路由规则。virtual service的`hosts`不一定作为Istio服务注册表的一部分，它们只是**虚拟目的地**，允许用户为网格无法路由到的虚拟主机建立流量模型。

> virtual service的hosts短域名在解析为完整的域名时，补齐的namespace是VirtualService所在的命名空间，而非Service所在的命名空间。如上例的hosts会被解析为：reviews.default.svc.cluster.local。

#### 路由规则

`http`部分包含了virtual service的路由规则，描述了匹配条件，以及将HTTP/1.1，HTTP2和gRPC流量发送到`hosts`字段指定的目的地的相关动作。一个路由规则包含流量的目的地以及0个或多个条件，具体取决于实际的场景。

##### match

第一个路由规则的例子如下，它有一个以`match`字段开头的条件，这种情况下希望路由来自用户"jason"的所有请求，使用`headers`, `end-user`和`exact` 字段来挑选合适的请求。

```yaml
- match:
   - headers:
       end-user:
         exact: jason
```

##### destination

路由部分的`destination`字段指定了匹配条件的流量的实际地址。与virtual service的主机不同，该`host`必须是存在于istio的服务注册表中的真实目的地，否则Envoy不知道应该将流量发送到哪里。它可以是一个带代理的网格服务或使用`service entry`添加的非网格服务。在kubernetes作为平台的情况下，`host`表示名为kubernetes的service名称：

```yaml
route:
- destination:
    host: reviews
    subset: v2
```

注意在本章的其他例子中使用kubernetes的短名称作为`destination`的主机。当使用这种规则时，istio会根据包含路由规则的virtual service的命名空间添加域后缀，在这些例子使用短名称意味着可以在任意命名空间中应用这些规则。

Istio根据包含路由规则的虚拟服务的命名空间添加域后缀来获取主机的完整的名称。

> 只有当destination主机和virtual service在相同的kubernetes命名空间下面才会正常工作。由于kubernetes的短名称可能会导致误解，因此建议在生成环境中使用完整的主机名。

`destination` 部分也指定了期望请求的kubernetes的服务子集，上面例子中定义了名为v2的子集。

##### 路由规则优先级

路由规则按从上到下的顺序运行， virtual service定义的第一个规则具有最高的优先级。这种情况下当不匹配第一条规则时，会进行第二条规则的匹配，当第二条规则不匹配时，会进行第三条规则的匹配。

```shell
- route:
  - destination:
      host: reviews
      subset: v3
```

建议提供一个无需匹配的默认规则作为每个virtual service的最后一条规则，确保virtual service的流量能够至少匹配到一条路由。

##### 路由规则的更多介绍

正如上面描述的，可以通过设置路由规则将特定的流量分发到特定的目的地。例如，以下virtual service可以让用户将流量分发到两个独立的service(reviews和ratings)上，就像这两个service是virtual service http://bookinfo.com/的一部分(*即virtual service作为kubernetes的service的service，可以方便前者进行扩展*)。virtual service的路由匹配规则基于请求的URLs，然后将请求分发到正确的service上。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo   #virtual service的名称
spec:
  hosts:
    - bookinfo.com #virtual service的地址
  http:
  - match:         #匹配条件一
    - uri:
        prefix: /reviews
    route:         ##如果匹配条件一，则路由到真实的reviews服务上
    - destination:
        host: reviews
  - match:         #匹配条件二 
    - uri:
        prefix: /ratings
    route:         #如果匹配条件二，则路由到真实的ratings服务上
    - destination:
        host: ratings
```

对于一些匹配条件，用户可以使用确切的值，前缀或正则表达式。

可以在相同的`match`块中添加多个匹配条件，多个匹配条件之间的关系为AND；或在相同的规则中添加多个`match`块，多个`match`之间的关系为OR。对于一个给定的virtual service，可以配置多个路由规则，者使得一个virtual serivce中的路由条件变得更加复杂或更加简单。可以在[HTTPMatchRequest reference](https://istio.io/docs/reference/config/networking/virtual-service/#HTTPMatchRequest)中找到完整的`match`字段以及字段值。

除了使用匹配条件，还可以使用百分比的"权重"来分发流量，通常用于A/B测试和金丝雀发布。

```yaml
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 75
    - destination:
        host: reviews
        subset: v2
      weight: 25
```

此外还可以使用路由规则对流量采取某些动作，如：

- 追加或移除首部
- 重写URL
- 对本目的地的调用设置重试策略

更多参见[HTTPRoute reference](https://istio.io/docs/reference/config/networking/virtual-service/#HTTPRoute)

### [Destination Rule](https://istio.io/docs/reference/config/networking/destination-rule/)

destination rule是istio流量路由功能的重要组成部分。一个virtual service可以看作是如何将流量分发的给定的目的地，然后调用destination rule来配置分发到该目的地的流量。destination rule在virtual service的路由规则之后起作用(*即在virtual service的math->route-destination之后起作用，此时流量已经分发到真实的service上*)，应用于真实的目的地。

特别地，可以使用destination rule来指定命名的服务子集，例如根据版本对服务的实例进行分组，然后通过virtual service的路由规则中的服务子集将控制流量分发到不同服务的实例中。

destination rule允许在调用完整的目标服务或特定的服务子集(如倾向使用的负载均衡模型，TLS安全模型或断路器)时自定义Envoy流量策略。更多参见[Destination Rule reference](https://istio.io/docs/reference/config/networking/destination-rule/)。

#### 负载均衡选项

istio默认会使用轮询策略，此外istio也支持如下负载均衡模型，可以在destination rule中使用这些模型，将请求分发的特定的服务或服务子集。

- Random：将请求转发到一个随机的实例上
- Weighted：按照指定的百分比将请求转发到实例上
- Least requests：将请求转发到具有最少请求数目的实例上

更多参见[Envoy load balancing documentation](https://www.envoyproxy.io/docs/envoy/v1.5.0/intro/arch_overview/load_balancing)

#### Destination rule例子

下面的Destination rule使用不同的负载均衡策略为my-svc目的服务配置了3个不同的子集(`subset`)。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:     #默认的负载均衡策略模型为随机
    loadBalancer:
      simple: RANDOM
  subsets:        
  - name: v1  #subset1，将流量转发到具有标签version:v1的deployment对应的服务上
    labels:
      version: v1
  - name: v2  #subset2，将流量转发到具有标签version:v2的deployment对应的服务上,指定负载均衡为轮询
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3   #subset3，将流量转发到具有标签version:v3的deployment对应的服务上
    labels:
      version: v3
```

每个子集由一个或多个`labels`定义，对应kubernetes中的对象(如pod)的key/value对。这些标签定义在kubernetes服务的deployment的`metadata` 中，用于标识不同的版本。

除了定义子集外，destination rule还定义了该目的地中所有子集的默认流量策略，以及仅覆盖该子集的特定策略。默认的策略定义在`subset`字段之上，为`v1`和`v3`子集设置了随机负载均衡策略，在`v2`策略中使用了轮询负载均衡。

### [Gateway](https://istio.io/docs/reference/config/networking/gateway/#Gateway) 

gateway用于管理进出网格的流量，指定可以进入或离开网格的流量。gateway配置应用于网格边缘的独立的Envoy代理上，而不是服务负载的Envoy代理上。

与其他控制进入系统的流量的机制(如kubernetes ingress API)不同，istio gateway允许利用istio的流量路由的强大功能和灵活性。istio的gateway资源仅允许配置4-6层的负载属性，如暴露的端口，TLS配置等等，但结合istio的virtual service，就可以像管理istio网格中的其他数据面流量一样管理gateway的流量。

gateway主要用于管理ingress流量，但也可以配置egress gateway。通过egress gateway可以配置流量离开网格的特定节点，限制哪些服务可以访问外部网络，或通过[egress安全控制](https://istio.io/blog/2019/egress-traffic-control-in-istio-part-1/)来提高网格的安全性。gateway可以用于配置为一个纯粹的内部代理。

istio(通过`istio-ingressgateway`和`istio-egressgateway`参数)提供了一些预配置的gateway代理，`default` profile下仅会部署ingress gateway。gateway可以通过部署文件进行部署，也可以单独部署。

下面是`default` profile默认安装的ingress

```shell
$ oc get gateways.networking.istio.io -n istio-system
NAME                   AGE
istio-ingressgateway   4d20h
```

可以看到该ingress就是一个普通的pod，该pod仅包含一个`istio-proxy`容器

```shell
$ oc get pod -n istio-system |grep ingress
istio-ingressgateway-64f6f9d5c6-qrnw2   1/1     Running   0          4d20h
```

下面是一个gateway的例子，用于配置外部HTTPS的ingress流量：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:              #指定gateway配置下发的代理，如具有标签app: my-gateway-controller的pod
    app: my-gateway-controller
  servers:
  - port:                #gateway pod暴露的端口信息
      number: 443
      name: https
      protocol: HTTPS
    hosts:                #外部流量
    - ext-host.example.com  
    tls:
      mode: SIMPLE
      serverCertificate: /tmp/tls.crt
      privateKey: /tmp/tls.key
```

上述gateway配置允许来自`ext-host.example.com` 流量进入网格的443端口，但没有指定该流量的路由。(*此时流量只能进入网格，但没有指定处理该流量的服务，因此需要与virtual service进行绑定*)

为了为gateway指定路由，需要通过virtual service的`gateway`字段，将gateway绑定到一个virtual service上，将来自`ext-host.example.com`流量引入一个`VirtualService`，`hosts`可以是通配符，表示引入匹配到的流量。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-svc
spec:
  hosts:
  - ext-host.example.com
  gateways:        #将gateway "ext-host-gwy"绑定到virtual service "virtual-svc"上
  - ext-host-gwy
```

### [Service entries](https://istio.io/docs/reference/config/networking/service-entry/#ServiceEntry)

*可以看作是一个网格外部的virtual service*

使用service entry可以在istio内部维护的service registry中注册一个表项。在添加service entry后，Envoy代理就可以将流量转发到该服务上(就像服务是网格中的服务一样)。配置service entry允许管理网格外的服务的流量，包括：

- 重定向和转发到达外部目的地的流量，例如web使用的api，或者使用遗留基础设施提供的服务。
- 为外部目的地定义重试，超时和故障注入策略
- 提供将[vm添加到网格](https://istio.io/docs/examples/virtual-machines/)中，在VM中运行网格服务
- 在逻辑上将一个不同的集群添加到网格中，来在kubernetes上配置[多集群istio网格](https://istio.io/docs/setup/install/multicluster/gateways/#configure-the-example-services)。

无需为每个网格服务使用的外部服务添加service entry。默认下，istio仅会配置Envoy代理来转发请求到无法识别的服务。如果一个目的地没有注册到网格中，则不能利用istio的特性来控制到该目的地的流量。

下面是一个通过service entry将外部服务`ext-svc.example.com` 添加到istio的service registry的例子：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: svc-entry
spec:
  hosts:
  - ext-svc.example.com   #外部服务
  ports:                  #外部服务对应的端口
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL #flag，表示该服务是网格外的服务
  resolution: DNS         #主机服务的发现模型
```

使用`hosts`字段指定外部资源，该字段可以是一个完整的域名，也可以是使用前缀通配符的域名。

可以使用virtual service和destination rule来控制到达一个service entry的流量。例如，下面destination rule配置了流量路由使用mutual TLS来加密到达外部服务`ext-svc.example.com`间的连接。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ext-res-dr
spec:
  host: ext-svc.example.com
  trafficPolicy:
    tls:
      mode: MUTUAL
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
```

### [Sidecars](https://istio.io/docs/reference/config/networking/sidecar/#Sidecar)

默认情况下，istio会配置每个Envory代理来接收与其关联的负载的(所有端口上的)流量，以及在转发流量时将流量分发到网格中的每个负载。使用sidecar可以实现如下功能：

- 对Envoy代理接受的端口和协议集进行调优
- 限制Envoy代理可以访问的服务集

在大型应用中，如果在每个服务都经过sidecar代理，可能会因为内存过高而影响网格的性能，这种情况下，可能需要限制sidecar的可达性。

可以将sidecar配置到某个特定的命名空间中，或通过`workloadSelector`选择特定的负载。例如，下面的sidecar配置将bookinfo命名空间中的所有服务配置为，仅访问在同一命名空间和Istio控制平面中的服务。

*sidecar配置仅能影响到该配置所在的命名空间*

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: bookinfo #sidecar所在的命名空间
spec:
  egress:             #限制bookinfo能够访问的egress 
  - hosts:            #使用namespace/dnsName格式的监听器，参见https://istio.io/docs/reference/config/networking/sidecar/#IstioEgressListener
    - "./*"
    - "istio-system/*"
```

### 网络弹性和测试

除了可以帮助引导网格周围的流量，Istio还提供可选的故障恢复和故障注入功能。可以在运行时动态配置这些功能。使用这些特性可以帮助增强应用的可靠性，保证服务网格能够容忍节点失败，以及防止本地化故障级联到其他节点。

#### 超时

timeout是Envoy代理在等待给定服务响应前的时间，保证不会无限等待服务的响应，最后返回成功会超时后返回失败。默认的HTTP请求超时时间为15s，即如果服务无法在15秒内响应，则调用失败。

对于一些应用和服务，istio的默认超时可能不大合适。例如，超时过长可能会在有失败的服务下出现大量延时，而超时过短可能会导致不必要的调用失败(如调用链比较长)。为了优化超时设置，istio允许使用virtual service动态调整每个服务的超时，而无需修改服务代码。下面virtual service将`ratings` 服务的`v1`服务子集的超时设置为10s。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    timeout: 10s
```

#### 重试

与timeout类似，可以在virtual service中设置单个服务的重试配置，默认情况下HTTP请求在返回错误前会重试2次。下面例子中配置的最大重试次数为3，每次超时时间为2s。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
```

#### [断路器](https://istio.io/docs/tasks/traffic-management/circuit-breaking/)

断路器是istio提供的另一种构建弹性微服务的机制。在断路器中，可以设置对服务中单个主机的呼叫限制，如限制到一台主机的并发连接数，或限制到一台主机的调用失败的次数，一旦达到限制值，断路器或发出告警并停止连接这台主机。

由于断路器会作用到负载均衡池中的真实网格目标，可以在`destination rules`中配置断路器阈值，该设置作用到服务中的每个主机。下面例子限制了reviews服务负载的v1服务子集的最大并发连接数为100。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 100
```

#### [故障注入](https://istio.io/docs/tasks/traffic-management/fault-injection/)

在配置完网络，包括故障恢复策略后，可以使用istio的故障注入机制来测试应用的故障恢复能力。故障注入是一个测试机制，将错误引入到一个系统来保证系统能够承受该错误支持错误恢复。使用故障注入可以特别有助于确保不会因为故障恢复策略的兼容性和限制性而导致关键服务不可用。

与在网络层面引入延迟报文或杀死pod不同，istio允许在应用层注入故障，如通过注入HTTP错误码来获取相关的结果。

使用virtual service可以注入两种类型的故障：

- 延迟：时间故障。用于模拟增加的网络延迟或超载的上游服务
- 中断：崩溃故障。用于模拟上游服务故障，通常以HTTP错误代码或TCP连接失败的形式出现。

下面例子中，使用virtual service注入故障，对`ratings`服务的访问请求中，每1000个请求中就有一个5s的延迟。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 0.1 #对1/1000的流量注入5s的延时
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
```

### 与应用的配合

istio的故障恢复功能对应用来说是完全透明的。一个应用无法知道一个Envoy sidecar代理是否为一个被调用的服务配置了失败处理功能。这意味着如果在应用代理中设置了故障恢复策略，则需要注意这两个策略是相互独立的，有可能发送冲突。例如，假设有两个超时设置，一个配置在virtual service中，一个配置在应用中。应用为调用某个服务的API设置的超时为2s；而virtual service中设置的超时为3s，重试次数为1。这种情况下，应用首先会发送超时，Envoy的超时和重试将失效。

虽然istio的故障恢复功能提升了网格中服务的可靠性和可用性，但应用仍然需要处理故障或错误，并做出相应动作。例如，当一个负载均衡池中的所有实例都失败后，Envoy会返回`HTTP 503`错误，应用必须实现对HTTP 503错误码做出反馈。