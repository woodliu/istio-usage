# Istio的流量管理(实操三)

涵盖官方文档[Traffic Management](https://istio.io/docs/tasks/traffic-management/)章节中的egress部分。其中有一小部分问题(已在下文标注)待官方解决。

[TOC]

## 访问外部服务

由于启用了istio的pod的出站流量默认都会被重定向到代理上，因此对集群外部URL的访问取决于代理的配置。默认情况下，Envoy代理会透传对未知服务的访问，虽然这种方式为新手提供了便利，但最好配置更严格的访问控制。

本节展示使用如下三种方式访问外部服务:

1. 允许Envoy代理透传到网格外部的服务
2. 配置[service entries](https://istio.io/docs/reference/config/networking/service-entry/)来访问外部访问
3. 透传某一个IP端的请求

### 部署

- 部署sleep app，用于发送请求。

  ```shell
  $ kubectl apply -f samples/sleep/sleep.yaml
  ```

- 设置`SOURCE_POD`为请求源pod名

  ```shell
  $ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
  ```

### Envoy透传流量到外部服务

istio有一个[安装选项](https://istio.io/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-OutboundTrafficPolicy-Mode)，`meshConfig.outboundTrafficPolicy.mode`，用于配置sidecar处理外部服务(即没有定义到istio内部服务注册中心的服务)。如果该选项设置为`ALLOW_ANY`，则istio代理会放行到未知服务的请求；如果选项设置为`REGISTRY_ONLY`，则istio代理会阻塞没有在网格中定义HTTP服务或服务表项的主机。默认值为`ALLOW_ANY`，允许快速对istio进行评估。

1. 首先将`meshConfig.outboundTrafficPolicy.mode`选项设置为`ALLOW_ANY`。默认应该就是`ALLOW_ANY`，使用如下方式获取当前的模式：

   ```shell
   $kubectl get configmap istio -n istio-system -o yaml |grep -o "mode: ALLOW_ANY" |uniq
   mode: ALLOW_ANY
   ```

   如果没有配置模式，可以手动添加：

   ```
   outboundTrafficPolicy: 
     mode: ALLOW_ANY
   ```

2. 从网格内向外部服务发送两个请求，可以看到请求成功，返回200

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://www.baidu.com | grep  "HTTP/"; kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://edition.cnn.com | grep "HTTP/"
   HTTP/1.1 200 OK
   HTTP/2 200
   ```

   使用这种方式可以访问外部服务，但无法对该流量进行监控和控制，下面介绍如何监控和控制网格到外部服务的流量。

### 控制访问外部服务

使用ServiceEntry配置可以从istio集群内部访问公共服务。本节展示如何配置访问外部HTTP服务，httpbin.org以及www.baidu.com，同时会监控和控制istio流量。

#### 修改默认的阻塞策略

为了展示如何控制访问外部服务的方式，需要将`meshConfig.outboundTrafficPolicy.mode`设置为`REGISTRY_ONLY`

1. 执行如下命令将`meshConfig.outboundTrafficPolicy.mode`选项设置为`REGISTRY_ONLY`

   ```shell
   $ kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' | kubectl replace -n istio-system -f -
   ```

2. 从SOURCE_POD访问外部HTTPS服务，此时请求会被阻塞(可能需要等一段时间来使配置生效)

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://www.baidu.com | grep  "HTTP/"; kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://edition.cnn.com | grep "HTTP/"
   command terminated with exit code 35
   command terminated with exit code 35
   ```

#### 访问外部HTTP服务

1. 创建一个`ServiceEntry`注册外部服务，这样就可以直接访问外部HTTP服务，可以看到此处并没有用到virtual service和destination rule

   > 下面serviceEntry使用`DNS` 作为resolution是一种比较安全的方式，将resolution设置为`NONE`将可能导致攻击。例如，恶意客户可能会再HOST首部中设置`httpbin.org`，但实际上访问的不同的IP地址。istio sidecar代理会信任HOST首部，并错误地允许此次访问(即使会将流量传递到不同于主机的IP地址)，该主机可能是一个恶意网站，或是一个被网格安全策略屏蔽的合法网站。
   >
   > 使用`DNS` resolution时，sidecar代理会忽略原始目的地址，并将流量传递给`hosts`字段的主机。在转发流量前会使用DNS请求`hosts`字段的IP地址。
   >
   > serviceEntry包括如下三种[resolution](https://istio.io/latest/docs/reference/config/networking/service-entry/#ServiceEntry-Resolution)：
   >
   > | Name     | Description                                                  |
   > | -------- | ------------------------------------------------------------ |
   > | `NONE`   | Assume that incoming connections have already been resolved (to a specific destination IP address). Such connections are typically routed via the proxy using mechanisms such as IP table REDIRECT/ eBPF. After performing any routing related transformations, the proxy will forward the connection to the IP address to which the connection was bound. |
   > | `STATIC` | Use the static IP addresses specified in endpoints (see below) as the backing instances associated with the service. |
   > | `DNS`    | Attempt to resolve the IP address by querying the ambient DNS, during request processing. If no endpoints are specified, the proxy will resolve the DNS address specified in the hosts field, if wildcards are not used. If endpoints are specified, the DNS addresses specified in the endpoints will be resolved to determine the destination IP address. DNS resolution cannot be used with Unix domain socket endpoints. |

   ```shell
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: httpbin-ext
   spec:
     hosts:
     - httpbin.org #外部服务URI
     ports:
     - number: 80 #外部服务HTTP端口信息
       name: http
       protocol: HTTP
     resolution: DNS
     location: MESH_EXTERNAL # 表示一个外部服务，即httpbin.org是网格外部的服务
   EOF
   ```

2. 从SOURCE_POD请求外部HTTP服务

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl http://httpbin.org/headers
   {
     "headers": {
       "Accept": "*/*",
       "Content-Length": "0",
       "Host": "httpbin.org",
       "User-Agent": "curl/7.64.0",
       ...
       "X-Envoy-Decorator-Operation": "httpbin.org:80/*",
     }
   }
   ```

   注意HTTP添加了istio sidecar代理首部`X-Envoy-Decorator-Operation`。

3. 校验`SOURCE_POD` sidecar代理的日志(实际并没有如同官方文档中的打印，[issue](https://github.com/istio/istio.io/issues/7419)跟踪)

   ```shell
   $ kubectl logs $SOURCE_POD -c istio-proxy | tail
   ```

#### 访问外部HTTPS服务

1. 创建ServiceEntry允许访问外部HTTPS服务

   ```shell
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: baidu
   spec:
     hosts:
     - www.baidu.com
     ports:
     - number: 443 # 外部服务HTTPS端口
       name: https
       protocol: HTTPS #指定外部服务为HTTPS协议
     resolution: DNS
     location: MESH_EXTERNAL
   EOF
   ```

2. 从`SOURCE_POD`访问外部服务

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://www.baidu.com | grep  "HTTP/"
   HTTP/1.1 200 OK
   ```

#### 管理到外部的流量

与管理集群内部的流量类似，istio 的路由规则也可以管理使用`ServiceEntry`配置的外部服务。本例将会为`httpbin.org`服务设置一个超时规则.

1. 从测试的pod向外部服务`httpbin.org`的/delay地址发送一个请求，大概5s后返回200

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
   200
   real    0m 5.43s
   user    0m 0.00s
   sys     0m 0.00s
   ```

2. 对外部服务`httpbin.org`设置一个3s的超时时间

   ```shell
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: httpbin-ext
   spec:
     hosts:
       - httpbin.org #此处的hosts与serviceEntry的hosts字段内容对应
     http:
     - timeout: 3s
       route:
         - destination:
             host: httpbin.org 
           weight: 100
   EOF
   ```

3. 几秒后，重新访问该服务，可以看到访问超时

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- time curl -o /dev/null -s -w "%{http_code}\n" http://httpbin.org/delay/5
   504
   real    0m 3.02s
   user    0m 0.00s
   sys     0m 0.00s
   ```

#### 卸载

```shell
$ kubectl delete serviceentry httpbin-ext google
$ kubectl delete virtualservice httpbin-ext --ignore-not-found=true
```

#### 直接访问外部服务

可以配置Envoy sidecar，使其不拦截特定IP段的请求。为了实现该功能，可以修改`global.proxy.includeIPRanges`或`global.proxy.excludeIPRanges`[配置选项](https://archive.istio.io/v1.4/docs/reference/config/installation-options/)(*类似白名单和黑名单*)，并使用`kubectl apply`命令更新`istio-sidecar-injector`配置。也可以修改annotations  `traffic.sidecar.istio.io/includeOutboundIPRanges`来达到相同的效果。在更新`istio-sidecar-injector`配置后，相应的变动会影响到所有的应用pod。

> 与使用ALLOW_ANY流量策略配置sidecar放行所有到未知服务的流量不同，上述方式会绕过sidecar的处理，即在特定IP段上不启用istio功能。使用这种方式不能增量地为特定目的地添加service entry，但使用`ALLOW_ANY`方式是可以的，因此这种方式仅仅建议用于性能测试或其他特殊场景中。

一种不把到外部IP的流量重定向到sidecar代理的方式是将`global.proxy.includeIPRanges`设置为集群内部服务使用的一个IP段或多个IP段。

[找到](https://istio.io/docs/tasks/traffic-management/egress/egress-control/#determine-the-internal-ip-ranges-for-your-platform)平台使用的内部IP段后，就可以使用如下方式配置`includeIPRanges`，这样目的地非`10.0.0.1/24`的流量会绕过sidecar的处理。

```shell
$ istioctl manifest apply <the flags you used to install Istio> --set values.global.proxy.includeIPRanges="10.0.0.1/24"
```

### 总结

本节介绍了三种访问外部服务的方式:

1. 配置Envoy 允许访问外部服务
2. 在网格内部使用service entry注册可访问的外部服务，推荐使用这种方式
3. 配置istio sidecar排除处理某些IP段的流量

第一种方式的流量会经过istio sidecar代理，当使用这种方式时，无法监控访问外部服务的流量，无法使用istio的流量控制功能。第二种方法可以在调用集群内部或集群外部的服务时充分使用istio服务网格特性，本章的例子中，在访问外部服务时设置了超时时间。第三种方式会绕过istio sidecar代理，直接访问外部服务。然而这种方式需要指定集群的配置，与第一种方式类似，这种方式也无法监控到外部服务的流量，且无法使用istio的功能。

### 卸载

```shell
$ kubectl delete -f samples/sleep/sleep.yaml
```

#### 环境恢复

检查当前的模式

```shell
$ kubectl get configmap istio -n istio-system -o yaml | grep -o "mode: ALLOW_ANY" | uniq
$ kubectl get configmap istio -n istio-system -o yaml | grep -o "mode: REGISTRY_ONLY" | uniq
```

将模式从`ALLOW_ANY`切换到`REGISTRY_ONLY`

```shell
$ kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' | kubectl replace -n istio-system -f -
```

将模式从`REGISTRY_ONLY`切换到`ALLOW_ANY`

```shell
$ kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: REGISTRY_ONLY/mode: ALLOW_ANY/g' | kubectl replace -n istio-system -f -
```

## Egress TLS Origination(Egress TLS源)

本节展示如何通过配置istio来(对到外部服务的流量)初始化TLS。当原始流量为HTTP时，Istio会与外部服务建立HTTPS连接，即istio会加密到外部服务的请求。

创建`sleep`应用

```shell
$ kubectl apply -f samples/sleep/sleep.yaml
```

获取`sleep`的pod名

```shell
$ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```

创建`ServiceEntry` 和`VirtualService`访问 `edition.cnn.com`

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: edition-cnn-com
spec:
  hosts:
  - edition.cnn.com #外部服务URI
  ports:
  - number: 80     # HTTP访问
    name: http-port
    protocol: HTTP
  - number: 443    # HTTPS访问
    name: https-port
    protocol: HTTPS #指定外部服务为HTTPS协议
  resolution: DNS
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: edition-cnn-com
spec:
  hosts:
  - edition.cnn.com #外部服务URI
  tls: #非终结的TLS&HTTPS流量
  - match: #将edition.cnn.com:443的流量分发到edition.cnn.com:443
    - port: 443
      sniHosts:
      - edition.cnn.com
    route:
    - destination:
        host: edition.cnn.com
        port:
          number: 443
      weight: 100
EOF
```

访问外部服务，下面使用了`-L`选项使请求端依照返回的重定向信息重新发起请求。第一个请求会发往`http://edition.cnn.com/politics`，服务端会返回重定向信息，第二个请求会按照重定向信息发往`https://edition.cnn.com/politics`。可以看到第一次是`HTTP`访问，第二次是`HTTPS`访问。

> 如果没有上述VirtualService，也可以通过下面命令进行访问。此处应该是为了与下面例子结合。

```shell
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics

HTTP/1.1 301 Moved Permanently
...
location: https://edition.cnn.com/politics
...

HTTP/2 200
...
```

上述过程会有两个弊端：上面的第一个HTTP访问显然是冗余的；如果在应用和`edition.cnn.com` 之间存在攻击者，这样该攻击者就可以通过嗅探链路获取请求端执行的操作，存在安全风险。

使用istio的TLS源可以解决如上问题。

### 为Egress流量配置TLS源

重新定义 `ServiceEntry` 和`VirtualService` ，并增加`DestinationRule`来发起TLS。此时`VirtualService`会将HTTP请求流量从80端口重定向到`DestinationRule`的443端口，然后由`DestinationRule`来发起TLS。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry # serviceEntry跟前面配置一样
metadata:
  name: edition-cnn-com
spec:
  hosts:
  - edition.cnn.com #注册到注册中心的host。用于选择virtualService和DestinationRule
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
  - number: 443
    name: https-port-for-tls-origination
    protocol: HTTPS
  resolution: DNS
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: edition-cnn-com #请求的hosts字段
spec:
  hosts:
  - edition.cnn.com #请求中的hosts字段内容
  http:
  - match:
    - port: 80 #后续将http流量通过destinationrule转换为https流量
    route:
    - destination:
        host: edition.cnn.com #此时定义了DestinationRule，会经过DestinationRule处理
        subset: tls-origination
        port:
          number: 443
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: edition-cnn-com
spec:
  host: edition.cnn.com #istio注册表中的服务
  subsets:
  - name: tls-origination
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
      portLevelSettings: #配置与上游服务edition.cnn.com的连接。即在443端口上使用tls SIMPLE进行连接
      - port:
          number: 443
        tls:
          mode: SIMPLE # initiates HTTPS when accessing edition.cnn.com
EOF
```

向`http://edition.cnn.com/politics`发送请求，可以看到此时会返回200，且不会经过重定向，相当于做了一个代理。

```shell
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics
HTTP/1.1 200 OK
...
```

当然直接使用https进行访问也是可以的，与上面使用http进行访问的结果相同`kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - https://edition.cnn.com/politics`

### 需要考虑的安全性问题

由于应用和sidecar代理之间是没有加密。因此渗透到应用所在的node节点的攻击者仍然能够看到该节点上未加密的本地通信内容。对于安全性较高的场景，建议应用直接使用HTTPS。

### 卸载

```shell
$ kubectl delete serviceentry edition-cnn-com
$ kubectl delete virtualservice edition-cnn-com
$ kubectl delete destinationrule edition-cnn-com
$ kubectl delete -f samples/sleep/sleep.yaml
```

## Egress 网关

本节描述如何通过一个指定的egress网关访问外部服务。istio使用ingress和egress网关在服务网格边界配置负载均衡。一个ingress网关允许定义网格的入站点，egress网关的用法类似，定义了网格内流量的出站点。

### 使用场景

假设在一个安全要求比较高的组织中，所有离开服务网格的流量都要经过一个**指定的节点**(*前面的egress访问都是在离开pod之后按照k8s方式访问，并没有指定必须经过某个节点*)，这些节点会运行在指定的机器上，与运行应用的集群的节点分开。这些特定的节点会在出站流量上应用策略，且对这些节点的监控将更加严格。

另外一个场景是集群中的应用所在的节点没有公网IP，因此网格内部的服务无法访问因特网。定义一个egress网关并为该网关所在的节点分配公网IP，这样流量就可以通过该节点访问公网服务。

### 环境配置

创建sleep应用并获取Pod名

```shell
$ kubectl apply -f samples/sleep/sleep.yaml
$ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```

### 部署Istio egress网关

校验是否已经部署istio egress网关

```shell
$ kubectl get pod -l istio=egressgateway -n istio-system
```

如果没有部署，执行如下步骤部署egress网关

```shell
$ istioctl manifest apply -f cni-annotations.yaml --set values.global.istioNamespace=istio-system --set values.gateways.istio-ingressgateway.enabled=true --set values.gateways.istio-egressgateway.enabled=true
```

> 注意：apply的时候使用自己定制化的文件，否则系统会使用默认的profile，导致配置丢失！
>
> 下面操作关于在`default`命名空间中为egress网关创建destination rule，因此要求`sleep`应用也部署在`default`命名空间中。如果应用不在default命名空间中，将无法在[destination rule查找路径](https://istio.io/docs/ops/best-practices/traffic-management/#cross-namespace-configuration)找到destination rule，客户端请求将会失败。

### <span id="jump">HTTP流量的egress网关</span>

1. 上面例子中，当网格内的客户端可以直接访问外部服务，此处将会创建一个egress网关，内部流量访问外部服务时会经过该网关。创建一个`ServiceEntry`允许流量访问外部服务`edition.cnn.com`:

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: cnn
   spec:
     hosts:
     - edition.cnn.com
     ports: #可以通过HTTP和HTTPS服务外部服务
     - number: 80
       name: http-port
       protocol: HTTP
     - number: 443
       name: https
       protocol: HTTPS
     resolution: DNS
   EOF
   ```

2. 校验请求能够发往http://edition.cnn.com/politics，此处的操作与上一节相同。

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics
   HTTP/1.1 301 Moved Permanently
   ...
   
   HTTP/2 200
   ...
   ```

3. 为`edition.cnn.com`创建一个`Gateway`，端口80，监听来自`edition.cnn.com:80`的流量。

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: istio-egressgateway
   spec:
     selector:
       istio: egressgateway
     servers:
     - port: #监听来自edition.cnn.com:80的流量，
         number: 80
         name: http
         protocol: HTTP
       hosts:
       - edition.cnn.com
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule #该DestinationRule没有定义任何规则，实际可以删除该DestinationRule，并删除下面VirtualService的"subset: cnn"一行
   metadata:
     name: egressgateway-for-cnn
   spec:
     host: istio-egressgateway.istio-system.svc.cluster.local
     subsets:
     - name: cnn #下面VirtualService中会用到
   EOF
   ```

4. 定义`VirtualService`，将流量从sidecar定向到egress网关，然后将流量从egress网关定向到外部服务。

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: direct-cnn-through-egress-gateway
   spec:
     hosts:
     - edition.cnn.com
     gateways: #列出应用路由规则的网关
     - istio-egressgateway
     - mesh #istio保留字段，表示网格中的所有sidecar，当忽略gateways字段时，默认会使用mesh，此处表示所有sidecar到edition.cnn.com的请求
     http: #采用http路由规则
     - match: #各个match是OR关系
       - gateways: #处理mesh网关，将来自mesh的edition.cnn.com:80请求发往istio-egressgateway.istio-system.svc.cluster.local:80
         - mesh
         port: 80
       route:
       - destination:
           host: istio-egressgateway.istio-system.svc.cluster.local
           subset: cnn #对应DestinationRule中的subset名，由于使用了subset，因此必须使用DestinationRule。删除该行后就可以不使用上面的DestinationRule
           port:
             number: 80
         weight: 100
     - match:
       - gateways:
         - istio-egressgateway #处理istio-egressgateway网关，将来自gateway edition.cnn.com:80的请求发往edition.cnn.com:80
         port: 80
       route:
       - destination:
           host: edition.cnn.com #该host就对应serviceEntry注册的服务地址
           port:
             number: 80
         weight: 100
   EOF
   ```

5. 发送HTTP请求[http://edition.cnn.com/politics](https://edition.cnn.com/politics)

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics
   HTTP/1.1 301 Moved Permanently
   ...
   
   HTTP/2 200
   ...
   ```

6. 校验egress日志（需要[启用](https://istio.io/docs/tasks/observability/logs/access-log/#enable-envoy-s-access-logging)Envoy日志）

   ```shell
   $ kubectl logs -l istio=egressgateway -c istio-proxy -n istio-system | tail
   [2020-08-25T14:55:49.810Z] "GET /politics HTTP/2" 301 - "-" "-" 0 0 1445 1444 "10.80.3.231" "curl/7.64.0" "2151bde2-4382-4e2f-b088-e464943c2a9b" "edition.cnn.com" "151.101.1.67:80" outbound|80||edition.cnn.com 10.80.3.232:51516 10.80.3.232:8080 10.80.3.231:38072 - -
   ```

本例中仍然实在sleep pod中执行HTTP请求，通过301重定向重新发送HTTPS请求，而上面规则中并没有将HTTPs流程转发给网关，因此从上面网关上看不到到443端口的流量，但可以在sleep的istio-proxy sidecar的日志中可以看到完整的流量信息，如下：

```shell
[2020-08-25T14:55:33.114Z] "GET /politics HTTP/1.1" 301 - "-" "-" 0 0 310 310 "-" "curl/7.64.0" "d57ddf5f-985b-431a-8766-7481b75dc486" "edition.cnn.com" "151.101.1.67:80" outbound|80||edition.cnn.com 10.80.3.231:48390 151.101.65.67:80 10.80.3.231:44674 - default
[2020-08-25T14:55:33.439Z] "- - -" 0 - "-" "-" 906 1326852 5490 - "-" "-" "-" "-" "151.101.129.67:443" outbound|443||edition.cnn.com 10.80.3.231:47044 151.101.65.67:443 10.80.3.231:42990 edition.cnn.com -
```

#### 卸载

```shell
$ kubectl delete gateway istio-egressgateway
$ kubectl delete serviceentry cnn
$ kubectl delete virtualservice direct-cnn-through-egress-gateway
$ kubectl delete destinationrule egressgateway-for-cnn
```

### <span id="jump">HTTPS流量的egress gateway</span>

本节展示通过egress网关定向`HTTPS`流量，。会使用到`ServiceEntry`，一个egress `Gateway`和一个`VirtualService`。

1. 创建到`edition.cnn.com`的`ServiceEntry`，定义外部服务https://edition.cnn.com

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: cnn
   spec:
     hosts:
     - edition.cnn.com
     ports:
     - number: 443
       name: tls
       protocol: TLS #protocol为TLS，用于非终结的流量
     resolution: DNS
   EOF
   ```

   >protocol字段可以为`HTTP|HTTPS|GRPC|HTTP2|MONGO|TCP|TLS`其中之一，其中TLS 表示不会终止TLS连接，且连接会基于SNI首部进行路由。

2. 校验可以通过`ServiceEntry`访问https://edition.cnn.com/politics

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - https://edition.cnn.com/politics
   HTTP/2 200
   ...
   ```

3. 为`edition.cnn.com`创建egress `Gateway`，一个destination rule和一个virtual service。

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: istio-egressgateway
   spec:
     selector:
       istio: egressgateway
     servers:
     - port:
         number: 443
         name: tls
         protocol: TLS #该字段与serviceEntry的字段相同
       hosts:
       - edition.cnn.com
       tls:
         mode: PASSTHROUGH #透传模式，不在网关上终止TLS，由sidecar发起TLS
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: egressgateway-for-cnn
   spec:
     host: istio-egressgateway.istio-system.svc.cluster.local
     subsets:
     - name: cnn
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: direct-cnn-through-egress-gateway
   spec:
     hosts:
     - edition.cnn.com
     gateways:
     - mesh
     - istio-egressgateway
     tls: #此处由http变为了tls
     - match:
       - gateways:
         - mesh
         port: 443
         sniHosts:
         - edition.cnn.com #基于SNI的路由
       route:
       - destination:
           host: istio-egressgateway.istio-system.svc.cluster.local
           subset: cnn
           port:
             number: 443
     - match:
       - gateways:
         - istio-egressgateway
         port: 443
         sniHosts:
         - edition.cnn.com #指定tls的SNI
       route:
       - destination:
           host: edition.cnn.com
           port:
             number: 443
         weight: 100
   EOF
   ```

   > 由于TLS本身是加密的，无法像HTTP一样根据host首部字段进行路由管理，因此采用了SNI扩展。SNI位于TLS协商的client-hello阶段，作为client-hello的扩展字段存在，基于TLS SNI的路由与基于HTTP host首部字段的路由管理，在逻辑上是相同的。SNI也支持通配符模式。
   >
   > ![](https://img2020.cnblogs.com/blog/1334952/202008/1334952-20200825110446619-299137861.png)

4. 访问https://edition.cnn.com/politics

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - https://edition.cnn.com/politics
   HTTP/2 200
   ...
   ```

5. 校验log

   ```shell
   $ kubectl logs -l istio=egressgateway -n istio-system
   ...
   [2020-06-02T09:06:43.152Z] "- - -" 0 - "-" "-" 906 1309129 1282 - "-" "-" "-" "-" "151.101.193.67:443" outbound|443||edition.cnn.com 10.83.1.219:39574 10.83.1.219:443 10.80.3.25:35492 edition.cnn.com -
   ```

#### 卸载

```shell
$ kubectl delete serviceentry cnn
$ kubectl delete gateway istio-egressgateway
$ kubectl delete virtualservice direct-cnn-through-egress-gateway
$ kubectl delete destinationrule egressgateway-for-cnn
```

### 安全考量

istio不能保证所有通过egress网关出去的流量的安全性，仅能保证通过sidecar代理的流量的安全性。如果攻击者绕过了sidecar代理，就可以不经过egress网关直接访问外部服务。此时，攻击者的行为不受istio的控制和监控。集群管理员或云供应商必须保证所有的流量都要经过egress网关。例如，集群管理员可以配置一个防火墙，拒绝所有非egress网关的流量。[Kubernetes network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)也可以禁止所有非egress网关的流量。此外，集群管理员或云供应商可以配置网络来保证应用节点只能通过网关访问因特网，为了实现这种效果，需要阻止将公共IP分配给网关以外的pod，并配置NAT设备丢弃非egress网关的报文。

### 使用Kubernetes network policies

本节展示如何创建一个[Kubernetes network policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)来防止绕过egress网关。为了测试网络策略，需要创建一个命名空间`test-egress`，部署`sleep`应用，并尝试向网关安全的外部服务发送请求。

首先完成中的[egress-gateway-for-https-traffic](https://istio.io/docs/tasks/traffic-management/egress/egress-gateway/#egress-gateway-for-https-traffic)步骤，然后执行如下操作

1. 创建`test-egress`命名空间

   ```shell
   $ kubectl create namespace test-egress
   ```

2. 将`sleep`部署到`test-egress`命名空间中

   ```shell
   $ kubectl apply -n test-egress -f samples/sleep/sleep.yaml
   ```

3. 校验部署的pod不存在istio sidecar

   ```shell
   $ kubectl get pod $(kubectl get pod -n test-egress -l app=sleep -o jsonpath={.items..metadata.name}) -n test-egress
   NAME                    READY   STATUS    RESTARTS   AGE
   sleep-f8cbf5b76-g2t2l   1/1     Running   0          27s
   ```

4. 从 `test-egress` 命名空间中的`sleep` pod向https://edition.cnn.com/politics 发送HTTPS请求，返回200成功

   ```shell
   $ kubectl exec -it $(kubectl get pod -n test-egress -l app=sleep -o jsonpath={.items..metadata.name}) -n test-egress -c sleep -- curl -s -o /dev/null -w "%{http_code}\n"  https://edition.cnn.com/politics
   200
   ```

5. 在istio组件所在的命名空间创建标签，如果istio组件部署在`istio-system`命名空间中，则操作方式如下：

   ```shell
   $ kubectl label namespace istio-system istio=system
   ```

6. 给`kube-system`命名空间打标签

   ```shell
   $ kubectl label ns kube-system kube-system=true
   ```

7. 部署一个`NetworkPolicy`限制从`test-egress`命名空间到`istio-system`命名空间和`kube-system` DNS服务的egress流量：

   ```yaml
   $ cat <<EOF | kubectl apply -n test-egress -f -
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-egress-to-istio-system-and-kube-dns
   spec:
     podSelector: {}
     policyTypes:
     - Egress
     egress:
     - to:
       - namespaceSelector:
           matchLabels:
             kube-system: "true"
       ports:
       - protocol: UDP
         port: 53
     - to:
       - namespaceSelector:
           matchLabels:
             istio: system
   EOF
   ```

8. 重新发送HTTPS请求到https://edition.cnn.com/politics，此时由于network policy阻止了流量，请求会失败。由于`sleep` Pod无法绕过`istio-egressgateway`(*需要环境保证，如果环境上即使没有`istio-egressgateway`也能访问外部服务，则此处可能会与预期不一样，本人使用的openshift环境无法测试这种场景*)访问外部服务，唯一的方式是将流量定向到`istio-egressgateway`上。

   ```shell
   $ kubectl exec -it $(kubectl get pod -n test-egress -l app=sleep -o jsonpath={.items..metadata.name}) -n test-egress -c sleep -- curl -v https://edition.cnn.com/politics
   Hostname was NOT found in DNS cache
     Trying 151.101.65.67...
     Trying 2a04:4e42:200::323...
   Immediate connect fail for 2a04:4e42:200::323: Cannot assign requested address
     Trying 2a04:4e42:400::323...
   Immediate connect fail for 2a04:4e42:400::323: Cannot assign requested address
     Trying 2a04:4e42:600::323...
   Immediate connect fail for 2a04:4e42:600::323: Cannot assign requested address
     Trying 2a04:4e42::323...
   Immediate connect fail for 2a04:4e42::323: Cannot assign requested address
   connect to 151.101.65.67 port 443 failed: Connection timed out
   ```

9. 现在给`test-egress`命名空间的`sleep` pod注入istio sidecar代理

   ```shell
   $ kubectl label namespace test-egress istio-injection=enabled
   ```

10. 重新在`test-egress`命名空间中部署`sleep` deployment

    openshift环境需要首先执行如下步骤：

    ```shell
    $ cat <<EOF | oc -n test-egress create -f -
    apiVersion: "k8s.cni.cncf.io/v1"
    kind: NetworkAttachmentDefinition
    metadata:
      name: istio-cni
    EOF
    
    $ oc adm policy add-scc-to-group privileged system:serviceaccounts:test-egress
    $ oc adm policy add-scc-to-group anyuid system:serviceaccounts:test-egress
    ```

    部署`sleep`应用

    ```shell
    $ kubectl delete deployment sleep -n test-egress
    $ kubectl apply -f samples/sleep/sleep.yaml -n test-egress
    ```

11. 校验`test-egress`命名空间的sleep注入了istio sidecar

    ```shell
    $ kubectl get pod $(kubectl get pod -n test-egress -l app=sleep -o jsonpath={.items..metadata.name}) -n test-egress -o jsonpath='{.spec.containers[*].name}'
    sleep istio-proxy
    ```

12. 创建与`default`命名空间中相同的destination rule，将流量定向到egress网关：

    ```yaml
    $ kubectl apply -n test-egress -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: egressgateway-for-cnn
    spec:
      host: istio-egressgateway.istio-system.svc.cluster.local #内部服务地址，不需要用serviceEntry
      subsets:
      - name: cnn
    EOF
    ```

13. 发送HTTPS请求到https://edition.cnn.com/politics：

    ```shell
    $ kubectl exec -it $(kubectl get pod -n test-egress -l app=sleep -o jsonpath={.items..metadata.name}) -n test-egress -c sleep -- curl -s -o /dev/null -w "%{http_code}\n" https://edition.cnn.com/politics
    ```

14. 校验egress网关代理的日志

    ```shell
    $ kubectl logs -l istio=egressgateway -n istio-system
    ...
    [2020-06-02T09:04:11.239Z] "- - -" 0 - "-" "-" 906 1309030 1606 - "-" "-" "-" "-" "151.101.1.67:443" outbound|443||edition.cnn.com 10.83.1.219:56116 10.83.1.219:443 10.80.3.25:59032 edition.cnn.com -
    ```

#### 卸载网络策略

```shell
$ kubectl delete -f samples/sleep/sleep.yaml -n test-egress
$ kubectl delete destinationrule egressgateway-for-cnn -n test-egress
$ kubectl delete networkpolicy allow-egress-to-istio-system-and-kube-dns -n test-egress
$ kubectl label namespace kube-system kube-system-
$ kubectl label namespace istio-system istio-
$ kubectl delete namespace test-egress
```

## 带TLS源的Egress网关(文件挂载)

在上一节的[HTTPS流量的egress gateway](#jump)中展示了如何配置istio来实现对外部服务的流量发起TLS。[HTTP流量的egress网关](#jump)中展示例子展示了如何配置istio来通过一个特定的egress网格服务来转发egress流量。本节的例子将结合这两个例子来描述如何配置一个egress网关来为到外部服务的流量发起TLS。

### 部署

在default(已启用sidecar自动注入)命名空间下安装sleep

```shell
$ kubectl apply -f samples/sleep/sleep.yaml
```

获取Pod名称

```shell
$ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```

创建egress网关

```shell
$ istioctl install -f cni-annotations.yaml --set values.global.istioNamespace=istio-system --set values.gateways.istio-egressgateway.enabled=true  --set meshConfig.accessLogFile="/dev/stdout"
```

### 使用一个egress网关发起TLS

本节描述如何使用于[HTTPS流量的egress gateway](#jump)相同的方式发起TLS，但此处使用了一个egress网关。注意这种情况下，通过egress网关来发起TLS，而前面的例子中使用了sidecar发起TLS(curl时指定的是https://edition.cnn.com/politics)。

1. 为`edition.cnn.com`定义一个`ServiceEntry`

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: cnn
   spec:
     hosts:
     - edition.cnn.com
     ports:
     - number: 80
       name: http
       protocol: HTTP
     - number: 443
       name: https
       protocol: HTTPS
     resolution: DNS
   EOF
   ```

2. 校验可以通过创建的`ServiceEntry`向[http://edition.cnn.com/politics](https://edition.cnn.com/politics)发送请求

   ```shell
   # kubectl exec "${SOURCE_POD}" -c sleep -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics
   HTTP/1.1 301 Moved Permanently
   ...
   location: https://edition.cnn.com/politics
   ...
   ```

3. 为`edition.cnn.com`创建一个`Gateway`，监听`edition.cnn.com:80`，以及一个destination rule来处理sidecar到egress网关的请求

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: istio-egressgateway
   spec:
     selector:
       istio: egressgateway
     servers: #配置网关暴露的主机信息
     - port:
         number: 80
         name: https-port-for-tls-origination
         protocol: HTTPS
       hosts:
       - edition.cnn.com
       tls: 
         mode: ISTIO_MUTUAL #与网关的连接使用ISTIO_MUTUAL模式
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: egressgateway-for-cnn
   spec:
     host: istio-egressgateway.istio-system.svc.cluster.local
     subsets:
     - name: cnn
       trafficPolicy:
         loadBalancer:
           simple: ROUND_ROBIN
         portLevelSettings: #基于单个端口的流量策略
         - port:
             number: 80
           tls: #与上游服务的连接设置，即到网关的tls配置，使用ISTIO_MUTUAL模式
             mode: ISTIO_MUTUAL
             sni: edition.cnn.com #表示TLS连接的服务端
   EOF
   ```

4. 定义一个`VirtualService`将流量转移到egress网关，以及一个`DestinationRule`来为到`edition.cnn.com`的请求发起TLS。

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: direct-cnn-through-egress-gateway
   spec:
     hosts:
     - edition.cnn.com
     gateways:
     - istio-egressgateway
     - mesh
     http:
     - match:
       - gateways: #处理来自网格内部所有到edition.cnn.com的流量，发送到egress网关，并使用subset: cnn进行处理
         - mesh
         port: 80
       route:
       - destination:
           host: istio-egressgateway.istio-system.svc.cluster.local
           subset: cnn
           port:
             number: 80
         weight: 100
     - match:
       - gateways:
         - istio-egressgateway #处理来自网关istio-egressgateway的流量，直接发往edition.cnn.com
         port: 80
       route:
       - destination:
           host: edition.cnn.com
           port:
             number: 443
         weight: 100
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: originate-tls-for-edition-cnn-com
   spec:
     host: edition.cnn.com
     trafficPolicy:
       loadBalancer:
         simple: ROUND_ROBIN
       portLevelSettings:
       - port:
           number: 443
         tls:
           mode: SIMPLE # 网关到edition.cnn.com使用SIMPLE模式，由于edition.cnn.com是网格外部服务，因此不能使用ISTIO_MUTUAL
   EOF
   ```

   整个过程为：网格内部HTTP流量->istio-egressgateway(配置TLS)->发起TLS连接

5. 发送`HTTP`请求到[http://edition.cnn.com/politics](https://edition.cnn.com/politics)

   ```shell
   # kubectl exec "${SOURCE_POD}" -c sleep -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics
   HTTP/1.1 200 OK
   ```

   此时的输出中不包含*301 Moved Permanently* 消息

6. 校验`istio-egressgateway` pod的日志

   ```shell
   $ kubectl logs -l istio=egressgateway -c istio-proxy -n istio-system | tail
   ```

   可以看到如下输出：

   ```shell
   [2020-08-25T15:16:17.840Z] "GET /politics HTTP/2" 200 - "-" "-" 0 1297688 7518 460 "10.80.3.231" "curl/7.64.0" "2c71707e-3304-418c-840e-c37256c1ad41" "edition.cnn.com" "151.101.193.67:443" outbound|443||edition.cnn.com 10.80.3.232:38522 10.80.3.232:8080 10.80.3.231:46064 edition.cnn.com -
   ```

> 各种资源的tls设置：
>
> | 资源            | 描述                                                         |
> | --------------- | ------------------------------------------------------------ |
> | virtualService  | [tls](https://istio.io/latest/docs/reference/config/networking/virtual-service/#TLSRoute)字段：用于非终结TLS&HTTPS流量的路由规则。通常使用ClientHello消息中的SNI值进行路由。TLS路由将会应用于端口名为`https -` `tls-`的平台服务，使用HTTPS/TLS协议的非终结网关端口(使用`passthrough`TLS模式)，以及使用HTTPS/TLS协议的serviceEntry端口。注：不关联virtual service的`https-`或`tls-`端口的流量将被视为不透明的TCP流量。 |
> | DestinationRule | DestinationRule主要对连接上游服务的tls进行配置，包含网格内的网关以及网格外的服务<br />[ClientTLSSettings](https://istio.io/latest/docs/reference/config/networking/destination-rule/#ClientTLSSettings)字段：连接上游的SSL/TLS相关设置<br />[portLevelSettings](https://istio.io/latest/docs/reference/config/networking/destination-rule/#TrafficPolicy-PortTrafficPolicy)字段：按照端口对上游服务进行设置，该字段包含了ClientTLSSettings字段 |
> | Gateway         | Gateway主要暴露的服务的tls进行配置，含ingress和egress，前者通常可以使用SIMPLE/MUTUAL模式，后者可以使用SIMPLE/MUTUAL/ISTIO_MUTUAL模式[<br />ServerTLSSettings](https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings)字段：控制服务端行为的TLS相关选项集。使用这些选项来控制是否应将所有http请求重定向到https，并使用[TLS模式](https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSmode) |

#### 卸载

```shell
$ kubectl delete gateway istio-egressgateway
$ kubectl delete serviceentry cnn
$ kubectl delete virtualservice direct-cnn-through-egress-gateway
$ kubectl delete destinationrule originate-tls-for-edition-cnn-com
$ kubectl delete destinationrule egressgateway-for-cnn
```

### 使用egress网关发起mutual TLS

与前面章节类似，本节描述如何配置egress网关来向外部服务发起TLS，不同的是这次要使用mutual TLS(上面用的是SIMPLE模式)。

在本例中首先需要：

1. 生成client和server证书
2. 部署支持mutual TLS协议的外部服务
3. 重新部署egress网关，使用mutual TLS证书

然后就是通过egress 网关发起TLS。

#### 生成client和server的证书和key

1. 创建根证书和私钥，用于签发服务证书

   ```shell
   $ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
   ```

2. 为 `my-nginx.mesh-external.svc.cluster.local`创建证书和私钥

   ```shell
   $ openssl req -out my-nginx.mesh-external.svc.cluster.local.csr -newkey rsa:2048 -nodes -keyout my-nginx.mesh-external.svc.cluster.local.key -subj "/CN=my-nginx.mesh-external.svc.cluster.local/O=some organization"
   $ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in my-nginx.mesh-external.svc.cluster.local.csr -out my-nginx.mesh-external.svc.cluster.local.crt
   ```

3. 生成client证书和私钥

   ```shell
   $ openssl req -out client.example.com.csr -newkey rsa:2048 -nodes -keyout client.example.com.key -subj "/CN=client.example.com/O=client organization"
   $ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 1 -in client.example.com.csr -out client.example.com.crt
   ```

#### 部署mutual TLS server

为了模拟一个支持mutual TLS协议的外部服务，需要在kubernetes集群中部署一个NGINX服务，但该服务位于istio服务网格外，即位于一个没有启用istio sidecar代理注入的命名空间。

1. 创建一个唯一istio网格外的命名空间，名为`mesh-external`，该命名空间不启用sidecar自动注入。

   ```shell
   $ kubectl create namespace mesh-external
   ```

2. 创建kubernetes secret，包含服务端的证书和CA证书

   ```shell
   $ kubectl create -n mesh-external secret tls nginx-server-certs --key my-nginx.mesh-external.svc.cluster.local.key --cert my-nginx.mesh-external.svc.cluster.local.crt
   $ kubectl create -n mesh-external secret generic nginx-ca-certs --from-file=example.com.crt
   ```

3. 创建NGINX 服务的配置文件

   ```shell
   $ cat <<\EOF > ./nginx.conf
   events {
   }
   
   http {
     log_format main '$remote_addr - $remote_user [$time_local]  $status '
     '"$request" $body_bytes_sent "$http_referer" '
     '"$http_user_agent" "$http_x_forwarded_for"';
     access_log /var/log/nginx/access.log main;
     error_log  /var/log/nginx/error.log;
   
     server {
       listen 443 ssl;
   
       root /usr/share/nginx/html;
       index index.html;
   
       server_name my-nginx.mesh-external.svc.cluster.local;
       ssl_certificate /etc/nginx-server-certs/tls.crt;
       ssl_certificate_key /etc/nginx-server-certs/tls.key;
       ssl_client_certificate /etc/nginx-ca-certs/example.com.crt;
       ssl_verify_client on;
     }
   }
   EOF
   ```

4. 创建一个kubernetes ConfigMap来保存NGINX服务的配置信息

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: v1
   kind: Service
   metadata:
     name: my-nginx
     namespace: mesh-external
     labels:
       run: my-nginx
   spec:
     ports:
     - port: 443
       protocol: TCP
     selector:
       run: my-nginx
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: my-nginx
     namespace: mesh-external
   spec:
     selector:
       matchLabels:
         run: my-nginx
     replicas: 1
     template:
       metadata:
         labels:
           run: my-nginx
       spec:
         containers:
         - name: my-nginx
           image: nginx
           ports:
           - containerPort: 443
           volumeMounts:
           - name: nginx-config
             mountPath: /etc/nginx
             readOnly: true
           - name: nginx-server-certs
             mountPath: /etc/nginx-server-certs
             readOnly: true
           - name: nginx-ca-certs
             mountPath: /etc/nginx-ca-certs
             readOnly: true
         volumes:
         - name: nginx-config
           configMap:
             name: nginx-configmap
         - name: nginx-server-certs
           secret:
             secretName: nginx-server-certs
         - name: nginx-ca-certs
           secret:
             secretName: nginx-ca-certs
   EOF
   ```

#### 使用client证书重新部署egress网关

1. 创建kubernetes secret，包含客户端证书和CA证书

   ```shell
   $ kubectl create -n istio-system secret tls nginx-client-certs --key client.example.com.key --cert client.example.com.crt
   $ kubectl create -n istio-system secret generic nginx-ca-certs --from-file=example.com.crt
   ```

2. 更新`istio-egressgateway` deployment来挂载创建的secret。创建如下`gateway-patch.json`文件来给`istio-egressgateway` deployment打补丁。

   ```json
   cat > gateway-patch.json <<EOF
   [{
     "op": "add",
     "path": "/spec/template/spec/containers/0/volumeMounts/0",
     "value": {
       "mountPath": "/etc/istio/nginx-client-certs",
       "name": "nginx-client-certs",
       "readOnly": true
     }
   },
   {
     "op": "add",
     "path": "/spec/template/spec/volumes/0",
     "value": {
     "name": "nginx-client-certs",
       "secret": {
         "secretName": "nginx-client-certs",
         "optional": true
       }
     }
   },
   {
     "op": "add",
     "path": "/spec/template/spec/containers/0/volumeMounts/1",
     "value": {
       "mountPath": "/etc/istio/nginx-ca-certs",
       "name": "nginx-ca-certs",
       "readOnly": true
     }
   },
   {
     "op": "add",
     "path": "/spec/template/spec/volumes/1",
     "value": {
     "name": "nginx-ca-certs",
       "secret": {
         "secretName": "nginx-ca-certs",
         "optional": true
       }
     }
   }]
   EOF
   ```

3. 使用如下命令使补丁生效

   ```shell
   $ kubectl -n istio-system patch --type=json deploy istio-egressgateway -p "$(cat gateway-patch.json)"
   ```

4. 校验加载到`istio-egressgateway` pod中的密钥和证书

   ```shell
   $ kubectl exec -n istio-system "$(kubectl -n istio-system get pods -l istio=egressgateway -o jsonpath='{.items[0].metadata.name}')" -- ls -al /etc/istio/nginx-client-certs /etc/istio/nginx-ca-certs
   ```

   `tls.crt` 和`tls.key` 应该位于 `/etc/istio/nginx-client-certs`目录中，而 `ca-chain.cert.pem` 位于`/etc/istio/nginx-ca-certs`目录中。

#### 配置egress流量的mutual TLS源

1. 为`my-nginx.mesh-external.svc.cluster.local:443`创建一个egress Gateway，以及destination rules和virtual service来将流量转发到egress网关上，并通过该egress网关转发给外部服务。

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: istio-egressgateway
   spec:
     selector:
       istio: egressgateway
     servers:
     - port:
         number: 443
         name: https
         protocol: HTTPS
       hosts:
       - my-nginx.mesh-external.svc.cluster.local #暴露给网格内部服务地址，使用ISTIO_MUTUAL进行交互
       tls:
         mode: ISTIO_MUTUAL
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule #处理网格内部pod到网关的流量
   metadata:
     name: egressgateway-for-nginx
   spec:
     host: istio-egressgateway.istio-system.svc.cluster.local
     subsets:
     - name: nginx
       trafficPolicy:
         loadBalancer:
           simple: ROUND_ROBIN
         portLevelSettings: #连接的上游服务属性
         - port:
             number: 443
           tls:
             mode: ISTIO_MUTUAL
             sni: my-nginx.mesh-external.svc.cluster.local
   EOF
   ```

2. 定义一个VirtualService将流量转移到egress网关

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: direct-nginx-through-egress-gateway
   spec:
     hosts:
     - my-nginx.mesh-external.svc.cluster.local
     gateways:
     - istio-egressgateway
     - mesh
     http: #内部流量采用http协议，由网关进行mutual tls协商
     - match:
       - gateways:
         - mesh
         port: 80
       route:
       - destination:
           host: istio-egressgateway.istio-system.svc.cluster.local
           subset: nginx
           port:
             number: 443
         weight: 100
     - match:
       - gateways:
         - istio-egressgateway
         port: 443
       route:
       - destination:
           host: my-nginx.mesh-external.svc.cluster.local #外部服务地址
           port:
             number: 443
         weight: 100
   EOF
   ```

3. 添加一个`DestinationRule`来发起TLS

   ```yaml
   $ kubectl apply -n istio-system -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule #处理网关到外部服务的流量
   metadata:
     name: originate-mtls-for-nginx
   spec:
     host: my-nginx.mesh-external.svc.cluster.local
     trafficPolicy:
       loadBalancer:
         simple: ROUND_ROBIN
       portLevelSettings:
       - port:
           number: 443
         tls:
           mode: MUTUAL #使用MUTUAL模式连接外部服务，证书位于网关pod中
           clientCertificate: /etc/istio/nginx-client-certs/tls.crt
           privateKey: /etc/istio/nginx-client-certs/tls.key
           caCertificates: /etc/istio/nginx-ca-certs/example.com.crt
           sni: my-nginx.mesh-external.svc.cluster.local
   EOF
   ```

4. 发送HTTP请求到`http://my-nginx.mesh-external.svc.cluster.local`:

   ```shell
   $ kubectl exec "$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})" -c sleep -- curl -s http://my-nginx.mesh-external.svc.cluster.local
   ```

   ```html
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   ...
   ```

5. 校验`istio-egressgateway` pod的日志

   ```shell
   # kubectl logs -l istio=egressgateway -n istio-system | grep 'my-nginx.mesh-external.svc.cluster.local' | grep HTTP
   [2020-08-26T08:26:15.054Z] "GET / HTTP/1.1" 200 - "-" "-" 0 612 4 4 "10.80.3.231" "curl/7.64.0" "e8bf12bd-9c39-409e-a837-39afc151fc7e" "my-nginx.mesh-external.svc.cluster.local" "10.80.2.14:443" outbound|443||my-nginx.mesh-external.svc.cluster.local 10.80.2.15:56608 10.80.2.15:8443 10.80.3.231:50962 my-nginx.mesh-external.svc.cluster.local -
   ```

#### 卸载

```shell
$ kubectl delete secret nginx-server-certs nginx-ca-certs -n mesh-external
$ kubectl delete secret istio-egressgateway-certs istio-egressgateway-ca-certs -n istio-system
$ kubectl delete configmap nginx-configmap -n mesh-external
$ kubectl delete service my-nginx -n mesh-external
$ kubectl delete deployment my-nginx -n mesh-external
$ kubectl delete namespace mesh-external
$ kubectl delete gateway istio-egressgateway
$ kubectl delete virtualservice direct-nginx-through-egress-gateway
$ kubectl delete destinationrule -n istio-system originate-mtls-for-nginx
$ kubectl delete destinationrule egressgateway-for-nginx

$ rm example.com.crt example.com.key my-nginx.mesh-external.svc.cluster.local.crt my-nginx.mesh-external.svc.cluster.local.key my-nginx.mesh-external.svc.cluster.local.csr client.example.com.crt client.example.com.csr client.example.com.key

$ rm ./nginx.conf
$ rm ./gateway-patch.json
```

```shell
$ kubectl delete service sleep
$ kubectl delete deployment sleep
```

## 带TLS源的Egress网关(SDS)

本节展示如何通过配置一个egress网关来为到外部服务的流量发起TLS。使用[Secret Discovery Service (SDS)](https://www.envoyproxy.io/docs/envoy/latest/configuration/security/secret#secret-discovery-service-sds)来配置私钥，服务证书以及根证书(上一节中使用文件挂载方式来管理证书)。

### 部署

部署sleep应用，并获取其Pod名

```shell
$ kubectl apply -f samples/sleep/sleep.yaml
$ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```

### egress网关使用SDS发起TLS

#### 生成CA和server证书和密钥

1. 创建根证书和私钥来签署服务的证书

   ```shell
   $ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
   ```

2. 为 `my-nginx.mesh-external.svc.cluster.local`创建证书和私钥

   ```shell
   $ openssl req -out my-nginx.mesh-external.svc.cluster.local.csr -newkey rsa:2048 -nodes -keyout my-nginx.mesh-external.svc.cluster.local.key -subj "/CN=my-nginx.mesh-external.svc.cluster.local/O=some organization"
   $ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in my-nginx.mesh-external.svc.cluster.local.csr -out my-nginx.mesh-external.svc.cluster.local.crt
   ```

#### 部署一个simple 模式的TLS服务

下面的操作与使用文件挂载相同，部署一个NGINX服务

1. 创建istio网格外的命名空间`mesh-external`

   ```shell
   $ kubectl create namespace mesh-external
   ```

2. 创建kubernetes secret来保存服务的证书和CA证书

   ```shell
   $ kubectl create -n mesh-external secret tls nginx-server-certs --key my-nginx.mesh-external.svc.cluster.local.key --cert my-nginx.mesh-external.svc.cluster.local.crt
   $ kubectl create -n mesh-external secret generic nginx-ca-certs --from-file=example.com.crt
   ```

3. 创建NGINX服务的配置文件

   ```shell
   $ cat <<\EOF > ./nginx.conf
   events {
   }
   
   http {
     log_format main '$remote_addr - $remote_user [$time_local]  $status '
     '"$request" $body_bytes_sent "$http_referer" '
     '"$http_user_agent" "$http_x_forwarded_for"';
     access_log /var/log/nginx/access.log main;
     error_log  /var/log/nginx/error.log;
   
     server {
       listen 443 ssl;
   
       root /usr/share/nginx/html;
       index index.html;
   
       server_name my-nginx.mesh-external.svc.cluster.local;
       ssl_certificate /etc/nginx-server-certs/tls.crt;
       ssl_certificate_key /etc/nginx-server-certs/tls.key;
       ssl_client_certificate /etc/nginx-ca-certs/example.com.crt;
       ssl_verify_client off; # simple TLS下server不需要校验client的证书
     }
   }
   EOF
   ```

4. 创建一个kubernetes ConfigMap来保存NGINX服务的配置信息

   ```shell
   $ kubectl create configmap nginx-configmap -n mesh-external --from-file=nginx.conf=./nginx.conf
   ```

5. 部署NGINX服务

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: v1
   kind: Service
   metadata:
     name: my-nginx
     namespace: mesh-external
     labels:
       run: my-nginx
   spec:
     ports:
     - port: 443
       protocol: TCP
     selector:
       run: my-nginx
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: my-nginx
     namespace: mesh-external
   spec:
     selector:
       matchLabels:
         run: my-nginx
     replicas: 1
     template:
       metadata:
         labels:
           run: my-nginx
       spec:
         containers:
         - name: my-nginx
           image: nginx
           ports:
           - containerPort: 443
           volumeMounts:
           - name: nginx-config
             mountPath: /etc/nginx
             readOnly: true
           - name: nginx-server-certs
             mountPath: /etc/nginx-server-certs
             readOnly: true
           - name: nginx-ca-certs
             mountPath: /etc/nginx-ca-certs
             readOnly: true
         volumes:
         - name: nginx-config
           configMap:
             name: nginx-configmap
         - name: nginx-server-certs
           secret:
             secretName: nginx-server-certs
         - name: nginx-ca-certs
           secret:
             secretName: nginx-ca-certs
   EOF
   ```

#### 为egress流量发起simple TLS

1. 创建一个kubernetes Secret来保存egress网格发起TLS使用的CA证书，由于使用的是SIMPLE模式，因此无需客户端证书，仅对ca证书实现SDS，后续在网关的destinationRule中使用。

   ```shell
   $ kubectl create secret generic client-credential-cacert --from-file=ca.crt=example.com.crt -n istio-system
   ```

   注意，Istio-only-CA证书的secret名称必须以-cacert结尾，并且必须在与部署的Istio相同的命名空间(默认为`Istio-system`)中创建该secret。

   > secret名称不应该以`istio`或`prometheus`开头，且secret不能包含`token`字段

   下面的配置除最后一个destinationRule外，其余配置都与上一节相同

2. 为`my-nginx.mesh-external.svc.cluster.local:443`创建一个egress Gateway，以及destination rules和virtual service来将流量转发到egress网关上，并通过该egress网关转发给外部服务。

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: istio-egressgateway
   spec:
     selector:
       istio: egressgateway
     servers:
     - port:
         number: 443
         name: https
         protocol: HTTPS
       hosts:
       - my-nginx.mesh-external.svc.cluster.local
       tls:
         mode: ISTIO_MUTUAL
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: egressgateway-for-nginx
   spec:
     host: istio-egressgateway.istio-system.svc.cluster.local
     subsets:
     - name: nginx
       trafficPolicy:
         loadBalancer:
           simple: ROUND_ROBIN
         portLevelSettings:
         - port:
             number: 443
           tls:
             mode: ISTIO_MUTUAL
             sni: my-nginx.mesh-external.svc.cluster.local
   EOF
   ```

3. 定义一个VirtualService将流量转移到egress网关

   ```yaml
   kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: direct-nginx-through-egress-gateway
   spec:
     hosts:
     - my-nginx.mesh-external.svc.cluster.local
     gateways:
     - istio-egressgateway
     - mesh
     http:
     - match:
       - gateways:
         - mesh
         port: 80
       route:
       - destination:
           host: istio-egressgateway.istio-system.svc.cluster.local
           subset: nginx
           port:
             number: 443
         weight: 100
     - match:
       - gateways:
         - istio-egressgateway
         port: 443
       route:
       - destination:
           host: my-nginx.mesh-external.svc.cluster.local
           port:
             number: 443
         weight: 100
   EOF
   ```

4. 添加一个`DestinationRule`来发起TLS

   ```yaml
   $ kubectl apply -n istio-system -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: originate-tls-for-nginx
   spec:
     host: my-nginx.mesh-external.svc.cluster.local
     trafficPolicy:
       loadBalancer:
         simple: ROUND_ROBIN
       portLevelSettings:
       - port:
           number: 443
         tls:
           mode: SIMPLE
           credentialName: client-credential # 对应前面创建的包含ca证书的secret client-credential-cacert，但此时不带"-cacert"后缀
           sni: my-nginx.mesh-external.svc.cluster.local #网格外部服务
   EOF
   ```

5. 发送一个HTTP请求到 `http://my-nginx.mesh-external.svc.cluster.local`:

   ```shell
   $ kubectl exec "$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})" -c sleep -- curl -s http://my-nginx.mesh-external.svc.cluster.local
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   ...
   ```

6. 检查`istio-egressgateway`中的访问日志

   ```shell
   # kubectl logs -l istio=egressgateway -n istio-system | grep 'my-nginx.mesh-external.svc.cluster.local' | grep HTTP
   [2020-08-26T12:26:09.316Z] "GET / HTTP/1.1" 200 - "-" "-" 0 612 3 3 "10.80.3.231" "curl/7.64.0" "67803676-5617-4e12-a14a-5cef95ea2e87" "my-nginx.mesh-external.svc.cluster.local" "10.80.2.19:443" outbound|443||my-nginx.mesh-external.svc.cluster.local 10.80.2.15:40754 10.80.2.15:8443 10.80.3.231:57626 my-nginx.mesh-external.svc.cluster.local -
   ```

#### 卸载

```shell
$ kubectl delete destinationrule originate-tls-for-nginx -n istio-system
$ kubectl delete virtualservice direct-nginx-through-egress-gateway
$ kubectl delete destinationrule egressgateway-for-nginx
$ kubectl delete gateway istio-egressgateway
$ kubectl delete secret client-credential-cacert -n istio-system
$ kubectl delete service my-nginx -n mesh-external
$ kubectl delete deployment my-nginx -n mesh-external
$ kubectl delete configmap nginx-configmap -n mesh-external
$ kubectl delete secret nginx-server-certs nginx-ca-certs -n mesh-external
$ kubectl delete namespace mesh-external
```

```shell
$ rm example.com.crt example.com.key my-nginx.mesh-external.svc.cluster.local.crt my-nginx.mesh-external.svc.cluster.local.key my-nginx.mesh-external.svc.cluster.local.csr
```

```shell
$ rm ./nginx.conf
```

### egress网关使用SDS发起mutual TLS

#### 创建客户端和服务端证书和密钥

下面操作跟前面一样，创建CA和客户端，服务端证书

```shell
$ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt

$ openssl req -out my-nginx.mesh-external.svc.cluster.local.csr -newkey rsa:2048 -nodes -keyout my-nginx.mesh-external.svc.cluster.local.key -subj "/CN=my-nginx.mesh-external.svc.cluster.local/O=some organization"
$ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in my-nginx.mesh-external.svc.cluster.local.csr -out my-nginx.mesh-external.svc.cluster.local.crt

$ openssl req -out client.example.com.csr -newkey rsa:2048 -nodes -keyout client.example.com.key -subj "/CN=client.example.com/O=client organization"
$ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 1 -in client.example.com.csr -out client.example.com.crt
```

#### 部署一个mutual TLS服务端

下面的配置也跟之前一样

```shell
$ kubectl create namespace mesh-external
```

```shell
$ kubectl create -n mesh-external secret tls nginx-server-certs --key my-nginx.mesh-external.svc.cluster.local.key --cert my-nginx.mesh-external.svc.cluster.local.crt
$ kubectl create -n mesh-external secret generic nginx-ca-certs --from-file=example.com.crt
```

```shell
$ cat <<\EOF > ./nginx.conf
events {
}

http {
  log_format main '$remote_addr - $remote_user [$time_local]  $status '
  '"$request" $body_bytes_sent "$http_referer" '
  '"$http_user_agent" "$http_x_forwarded_for"';
  access_log /var/log/nginx/access.log main;
  error_log  /var/log/nginx/error.log;

  server {
    listen 443 ssl;

    root /usr/share/nginx/html;
    index index.html;

    server_name my-nginx.mesh-external.svc.cluster.local;
    ssl_certificate /etc/nginx-server-certs/tls.crt;
    ssl_certificate_key /etc/nginx-server-certs/tls.key;
    ssl_client_certificate /etc/nginx-ca-certs/example.com.crt;
    ssl_verify_client on; # mutual TLS下的server会校验client的证书
  }
}
EOF
```

```shell
$ kubectl create configmap nginx-configmap -n mesh-external --from-file=nginx.conf=./nginx.conf
```

```yaml
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  namespace: mesh-external
  labels:
    run: my-nginx
spec:
  ports:
  - port: 443
    protocol: TCP
  selector:
    run: my-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  namespace: mesh-external
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 443
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx
          readOnly: true
        - name: nginx-server-certs
          mountPath: /etc/nginx-server-certs
          readOnly: true
        - name: nginx-ca-certs
          mountPath: /etc/nginx-ca-certs
          readOnly: true
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-configmap
      - name: nginx-server-certs
        secret:
          secretName: nginx-server-certs
      - name: nginx-ca-certs
        secret:
          secretName: nginx-ca-certs
EOF
```

#### 配置egress流量使用SDS发起mutual TLS

1. 创建一个kubernetes secret来保存客户端证书和ca证书

   ```shell
   $ kubectl create secret -n istio-system generic client-credential --from-file=tls.key=client.example.com.key \
     --from-file=tls.crt=client.example.com.crt --from-file=ca.crt=example.com.crt
   ```

   使用SDS的secret名称跟上一节的要求一样，部署到istio所在的命名空间，且名称不能以`istio`和`prometheus`开头，不能包含`token`字段。

2. 为`my-nginx.mesh-external.svc.cluster.local:443`创建`Gateway`

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: istio-egressgateway
   spec:
     selector:
       istio: egressgateway
     servers:
     - port:
         number: 443
         name: https
         protocol: HTTPS
       hosts:
       - my-nginx.mesh-external.svc.cluster.local
       tls:
         mode: ISTIO_MUTUAL
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: egressgateway-for-nginx
   spec:
     host: istio-egressgateway.istio-system.svc.cluster.local
     subsets:
     - name: nginx
       trafficPolicy:
         loadBalancer:
           simple: ROUND_ROBIN
         portLevelSettings:
         - port:
             number: 443
           tls:
             mode: ISTIO_MUTUAL
             sni: my-nginx.mesh-external.svc.cluster.local
   EOF
   ```

3. 创建`VirtualService`

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: direct-nginx-through-egress-gateway
   spec:
     hosts:
     - my-nginx.mesh-external.svc.cluster.local
     gateways:
     - istio-egressgateway
     - mesh
     http:
     - match:
       - gateways:
         - mesh
         port: 80
       route:
       - destination:
           host: istio-egressgateway.istio-system.svc.cluster.local
           subset: nginx
           port:
             number: 443
         weight: 100
     - match:
       - gateways:
         - istio-egressgateway
         port: 443
       route:
       - destination:
           host: my-nginx.mesh-external.svc.cluster.local
           port:
             number: 443
         weight: 100
   EOF
   ```

4. 与前面不同点就在该`DestinationRule`中的`credentialName`字段，包含了前面创建的证书`client-credential`

   ```yaml
   $ kubectl apply -n istio-system -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: originate-mtls-for-nginx
   spec:
     host: my-nginx.mesh-external.svc.cluster.local
     trafficPolicy:
       loadBalancer:
         simple: ROUND_ROBIN
       portLevelSettings:
       - port:
           number: 443
         tls:
           mode: MUTUAL
           credentialName: client-credential # this must match the secret created earlier to hold client certs
           sni: my-nginx.mesh-external.svc.cluster.local
   EOF
   ```

5. 发送请求并校验egressgateway pod的日志

   ```shell
   $ kubectl exec "$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})" -c sleep -- curl -s http://my-nginx.mesh-external.svc.cluster.local
   $ kubectl logs -l istio=egressgateway -n istio-system | grep 'my-nginx.mesh-external.svc.cluster.local' | grep HTTP
   ```

#### 卸载

```shell
$ kubectl delete secret nginx-server-certs nginx-ca-certs -n mesh-external
$ kubectl delete secret client-credential -n istio-system
$ kubectl delete configmap nginx-configmap -n mesh-external
$ kubectl delete service my-nginx -n mesh-external
$ kubectl delete deployment my-nginx -n mesh-external
$ kubectl delete namespace mesh-external
$ kubectl delete gateway istio-egressgateway
$ kubectl delete virtualservice direct-nginx-through-egress-gateway
$ kubectl delete destinationrule -n istio-system originate-mtls-for-nginx
$ kubectl delete destinationrule egressgateway-for-nginx
```

```shell
$ rm example.com.crt example.com.key my-nginx.mesh-external.svc.cluster.local.crt my-nginx.mesh-external.svc.cluster.local.key my-nginx.mesh-external.svc.cluster.local.csr client.example.com.crt client.example.com.csr client.example.com.key
```

```shell
$ rm ./nginx.conf
$ rm ./gateway-patch.json
```

```shell
$ kubectl delete service sleep
$ kubectl delete deployment sleep
```

## 使用通配符主机的egress

上两节中为网关配置了特定的主机名，如 `edition.cnn.com`。本节将展示如何为egress流量配置位于同域的一组主机，如`*.wikipedia.org`。

### 背景说明

假设要在istio上为所有语言的`wikipedia.org`站点启用egress流量，每个特定语言的`wikipedia.org`站点都有其各自的主机名，如`en.wikipedia.org` 和`de.wikipedia.org`分别表示英文和德文。此时可能会希望为所有的Wikipedia egress流量配置相同的参数，而不需要为每种语言的站点单独指定。

> 原文中用于展示的站点为`*.wikipedia.org`，但鉴于这类站点在国内无法访问，故修改为`*.baidu.com`

### 部署

部署sleep应用并获取POD名称

```shell
$ kubectl apply -f samples/sleep/sleep.yaml
$ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```

### 配置到通配符主机的直接流量

首先，为了简化场景，创建一个带通配符主机的`ServiceEntry`，并直接访问服务。当直接调用服务时(不经过egress网关)，通配符主机的配置与其他主机并没有什么不同(只是对同一域中的主机的服务更加方便)。

为`*.baidu.com`定义一个`ServiceEntry`和相应的`VirtualSevice`：

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: baidu
spec:
  hosts:
  - "*.baidu.com" #通配符主机
  ports:
  - number: 443
    name: tls
    protocol: TLS
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: baidu
spec:
  hosts:
  - "*.baidu.com"
  tls:
  - match:
    - port: 443
      sniHosts:
      - "*.baidu.com"
    route:
    - destination:
        host: "*.baidu.com"
        port:
          number: 443
EOF
```

发送请求给https://map.baidu.com/和https://fanyi.baidu.com/:

```shell
# kubectl exec -it $SOURCE_POD -c sleep -- sh -c 'curl -s https://map.baidu.com/ | grep -o "<title>.*</title>"; curl -s https://fanyi.baidu.com/ | grep -o "<title>.*</title>"'
<title>百度地图</title>
<title>百度翻译-200种语言互译、沟通全世界！</title>
```

> 可以不使用VirtualService

#### 卸载

```shell
$ kubectl delete serviceentry baidu
$ kubectl delete virtualservice baidu
```

### 配置到通配符主机的egress网关流量

通过egress网关访问通配符主机的配置取决于通配符域集是否由一个公共主机来提供服务。例如*\*.wikipedia.org*，所有指定语言的站点都由*\*wikipedia.org*的某一个服务端提供服务，这样就可以将流量路由到任何*\*.wikipedia.org*站点对应的IP(包括*www.wikipedia.org*)。

> 由于map.baidu.com和fanyi.baidu.com的服务并不是由www.baidu.com对应的某个IP服务的(可以使用nslookup或dig命令查看)，因此无法用于测试本场景，下面为官网内容

一般情况下，如果一个通配符的所有域名不是由一个托管服务器提供服务的，则需要更复杂的配置。

#### 单个主机服务器的通配符配置

当一个服务端服务所有的通配符主机时，对使用egress网关访问通配符主机的配置与访问非通配符主机的配置类似。

1. 为`*.wikipedia.org,`创建一个egress `Gateway`，destination rule和一个virtual service，将流量导入egress网关，并通过egress网关访问外部服务

   ```yaml
   $  kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: istio-egressgateway
   spec:
     selector:
       istio: egressgateway
     servers:
     - port:
         number: 443
         name: tls
         protocol: TLS
       hosts:
       - "*.wikipedia.org"
       tls:
         mode: PASSTHROUGH #由网格内部发起https请求，非终结TLS
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: egressgateway-for-wikipedia
   spec:
     host: istio-egressgateway.istio-system.svc.cluster.local
     subsets:
       - name: wikipedia
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: direct-wikipedia-through-egress-gateway
   spec:
     hosts:
     - "*.wikipedia.org"
     gateways:
     - mesh
     - istio-egressgateway
     tls: #网格内部的TLS流量处理
     - match:
       - gateways:
         - mesh
         port: 443
         sniHosts:
         - "*.wikipedia.org"
       route:
       - destination:
           host: istio-egressgateway.istio-system.svc.cluster.local
           subset: wikipedia
           port:
             number: 443
         weight: 100
     - match:
       - gateways:
         - istio-egressgateway
         port: 443
         sniHosts:
         - "*.wikipedia.org"
       route:
       - destination:
           host: www.wikipedia.org #将流量从网格传给外部服务
           port:
             number: 443
         weight: 100
   EOF
   ```

2. 为目的服务www.wikipedia.com创建`ServiceEntry`

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: www-wikipedia
   spec:
     hosts:
     - www.wikipedia.org
     ports:
     - number: 443
       name: tls
       protocol: TLS
     resolution: DNS
   EOF
   ```

3. 发送请求给https://map.baidu.com/和https://fanyi.baidu.com/:

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- sh -c 'curl -s https://en.wikipedia.org/wiki/Main_Page | grep -o "<title>.*</title>"; curl -s https://de.wikipedia.org/wiki/Wikipedia:Hauptseite | grep -o "<title>.*</title>"'
   <title>Wikipedia, the free encyclopedia</title>
   <title>Wikipedia – Die freie Enzyklopädie</title>
   ```

##### 卸载

```shell
$ kubectl delete serviceentry www-wikipedia
$ kubectl delete gateway istio-egressgateway
$ kubectl delete virtualservice direct-wikipedia-through-egress-gateway
$ kubectl delete destinationrule egressgateway-for-wikipedia
```

#### 任意域的通配符配置

上一节中的配置之所以能够生效，是因为任何一个*wikipedia.org*服务端都可以服务所有的*\*.wikipedia.org*站点。然有些情况下不是这样的，例如有可能希望访问更加通用的域，如`.com`或`.org`。

在istio网关上配置到任意通配符的域会带来挑战，上一节中直接将流量传递给了 *www.wikipedia.org*(直接配置到了网关上)。受限于[Envoy](https://www.envoyproxy.io/)(默认的istio egress网关代理)，网关并不知道接收到的请求中的任意主机的IP地址。Envoy会将流量路由到**预定义的主机**，**预定义的IP地址**或**请求中的原始目的IP地址**。在网关场景下，由于请求会首先被路由到egress网关上，因此会丢失请求中的原始目的IP地址，并将目的IP地址替换为网关的IP地址，最终会导致基于Envoy的istio网关无法路由到没有进行预配置的任意主机，进而导致无法为任意通配符域执行流量控制。

为了给HTTPS和TLS启用流量控制，需要额外部署一个SNI转发代理。Envoy会将到通配符域的请求路由到SNI转发代理，然后将请求转发到SNI中指定的目的地。

使用SNI代理和相关组件的egress网关架构如下，由于Envoy无法处理任意通配符的主机，因此需要转发到SNI代理上进行SNI的路由处理。

![](https://img2020.cnblogs.com/blog/1334952/202008/1334952-20200827105838226-1820408330.png)

下面将展示如何重新部署egress网关来使用SNI代理，并配置istio通过网关路由HTTPS流量到任意通配符域。

##### 配置egress网关的SNI代理

本节中将在标准的istio Envoy代理之外部署为egress网关部署一个SNI代理。本例中使用Nginx作为SNI代理，该SNI代理将会监听8443端口，然后将流量转发到443端口。

1. 为Nginx SNI代理创建配置文件。注意`server`下的`listen`指令为8443，`proxy_pass`指令使用`ssl_preread_server_name`，端口443以及将`ssl_preread`设置为`on`来启用`SNI` reading。

   ```shell
   $ cat <<EOF > ./sni-proxy.conf
   user www-data;
   
   events {
   }
   
   stream {
     log_format log_stream '\$remote_addr [\$time_local] \$protocol [\$ssl_preread_server_name]'
     '\$status \$bytes_sent \$bytes_received \$session_time';
   
     access_log /var/log/nginx/access.log log_stream;
     error_log  /var/log/nginx/error.log;
   
     # tcp forward proxy by SNI
     server {
       resolver 8.8.8.8 ipv6=off;
       listen       127.0.0.1:8443;
       proxy_pass   \$ssl_preread_server_name:443;
       ssl_preread  on;
     }
   }
   EOF
   ```

2. 创建一个kubernets ConfigMap来保存Nginx SNI代理的配置

   ```shell
   $ kubectl create configmap egress-sni-proxy-configmap -n istio-system --from-file=nginx.conf=./sni-proxy.conf
   ```

3. 下面命令将生成 `istio-egressgateway-with-sni-proxy.yaml` 

> 本例因为官方isitoOperator格式有变而无法运行，官方可能需要修改代码，参见此[issue](https://github.com/istio/istio/issues/26635)

##### 配置使用SNI代理的egress网关的路由转发

....

##### 卸载

```shell
$ kubectl delete serviceentry wikipedia
$ kubectl delete gateway istio-egressgateway-with-sni-proxy
$ kubectl delete virtualservice direct-wikipedia-through-egress-gateway
$ kubectl delete destinationrule egressgateway-for-wikipedia
$ kubectl delete --ignore-not-found=true envoyfilter forward-downstream-sni egress-gateway-sni-verifier
```

```shell
$ kubectl delete serviceentry sni-proxy
$ kubectl delete destinationrule disable-mtls-for-sni-proxy
$ kubectl delete -f ./istio-egressgateway-with-sni-proxy.yaml
$ kubectl delete configmap egress-sni-proxy-configmap -n istio-system
```

```shell
$ rm ./istio-egressgateway-with-sni-proxy.yaml
$ rm ./sni-proxy.conf
```

### 卸载

```shell
$ kubectl delete -f samples/sleep/sleep.yaml
```

## Egress流量的kubernetes服务

kubernetes [ExternalName](https://kubernetes.io/docs/concepts/services-networking/service/#externalname) services 和带[Endpoints](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors)的kubernetes services 允许为外部服务创建本地DNS别名，该DNS别名的格式与本地服务的DNS表项的格式相同，即 `<service name>.<namespace name>.svc.cluster.local`。DNS别名为工作负载提供了位置透明性：负载可以通过这种方式调用本地和外部服务。如果某个时间需要在**集群中**部署外部服务，就可以通过更新该kubernetes service来引用本地版本。工作负载将继续运行，不需要做任何改变。

本任务将展示istio如何使用这些kubernetes机制来访问外部服务。不过此时必须使用TLS模式来进行访问，而不是istio的mutual TLS。因为外部服务不是istio服务网格的一部分，因此不能使用istio mutual TLS，必须根据外部服务的需要以及负载访问外部服务的方式来设置TLS模式。如果负载发起明文HTTP请求，但外部服务需要TLS，此时可能需要istio发起TLS。如果负载已经使用了TLS，那么流量已经经过加密，此时就可以禁用istio的mutual TLS。

> 本节描述如何将istio集成到现有kubernetes配置中

虽然本例使用了HTTP协议，但使用egress流量的kubernetes Services也可以使用其他协议。

### 部署

- 部署sleep应用并获取POD名称

  ```shell
  $ kubectl apply -f samples/sleep/sleep.yaml
  $ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
  ```

- 创建一个不使用istio的命名空间

  ```shell
  $ kubectl create namespace without-istio
  ```

- 在`without-istio`命名空间下启用`sleep`

  ```shell
  $ kubectl apply -f samples/sleep/sleep.yaml -n without-istio
  ```

- 创建一个环境变量`SOURCE_POD_WITHOUT_ISTIO`来保存`without-istio`命名空间下的pod名称

  ```shell
  $ export SOURCE_POD_WITHOUT_ISTIO="$(kubectl get pod -n without-istio -l app=sleep -o jsonpath={.items..metadata.name})"
  ```

- 校验该pod没有istio sidecar

  ```shell
  # kubectl get pod "$SOURCE_POD_WITHOUT_ISTIO" -n without-istio
  NAME                    READY   STATUS    RESTARTS   AGE
  sleep-f8cbf5b76-tbptz   1/1     Running   0          53s
  ```

### 通过kubernetes ExternalName service访问外部服务

1. 在`default`命名空间中为`httpbin.org`创建一个kubernetes ExternalName service，将外服服务`httpbin.org`映射为kubernetes服务`my-httpbin`，即可以通过访问`my-httpbin.default.svc.cluster.local`来访问`my-httpbin`。

   ```yaml
   $ kubectl apply -f - <<EOF
   kind: Service
   apiVersion: v1
   metadata:
     name: my-httpbin
   spec:
     type: ExternalName
     externalName: httpbin.org
     ports:
     - name: http
       protocol: TCP
       port: 80
   EOF
   ```

2. 观察service，可以看到并没有cluster IP

   ```shell
   # kubectl get svc my-httpbin
   NAME         TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
   my-httpbin   ExternalName   <none>       httpbin.org   80/TCP    3s
   ```

3. 在源pod(不带istio sidecar)中通过kubernetes的service主机名访问 `httpbin.org`。由于在网格中，因此不需要访问时禁用istio 的mutual TLS。

   ```shell
   # kubectl exec "$SOURCE_POD_WITHOUT_ISTIO" -n without-istio -c sleep -- curl my-httpbin.default.svc.cluster.local/headers
   {
     "headers": {
       "Accept": "*/*",
       "Host": "my-httpbin.default.svc.cluster.local",
       "User-Agent": "curl/7.64.0",
       "X-Amzn-Trace-Id": "Root=1-5f485a71-54548d2e5f8b0dc1002e2ce0"
     }
   }
   ```

4. 本例中，使用向 `httpbin.org` 发送了未加密的HTTP请求。为了简单例子，下面禁用了TLS模式，允许向外部服务发送未加密的流量。在实际使用时，建议配置[Egress TLS源](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-tls-origination/)，下面相当于将流量重定向到内部服务`my-httpbin.default.svc.cluster.local`上，而`my-httpbin.default.svc.cluster.local`映射了外部服务 `httpbin.org`，这样就可以通过这种方式通过kubernetes service来访问外部服务。

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: my-httpbin
   spec:
     host: my-httpbin.default.svc.cluster.local
     trafficPolicy:
       tls:
         mode: DISABLE
   EOF
   ```

5. 在带istio sidecar的pod中通过kubernetes service主机名访问`httpbin.org`，注意isito sidecar添加的首部，如 `X-Envoy-Decorator-Operation`。同时注意`Host`首部字段等于自己的service主机名。

   ```shell
   # kubectl exec "$SOURCE_POD" -c sleep -- curl my-httpbin.default.svc.cluster.local/headers
   {
     "headers": {
       "Accept": "*/*",
       "Content-Length": "0",
       "Host": "my-httpbin.default.svc.cluster.local",
       "User-Agent": "curl/7.64.0",
       "X-Amzn-Trace-Id": "Root=1-5f485d05-ac81b19dcee92359b5cae307",
       "X-B3-Sampled": "0",
       "X-B3-Spanid": "bee9babd29c28cec",
       "X-B3-Traceid": "b8260c4ba5390ed0bee9babd29c28cec",
       "X-Envoy-Attempt-Count": "1",
       "X-Envoy-Decorator-Operation": "my-httpbin.default.svc.cluster.local:80/*",
       "X-Envoy-Peer-Metadata": "ChoKCkNMVVNURVJfSUQSDBoKS3ViZXJuZXRlcwo2CgxJTlNUQU5DRV9JUFMSJhokMTAuODAuMi4yNixmZTgwOjozMGI4OmI3ZmY6ZmUxNDpiMjE0Ct4BCgZMQUJFTFMS0wEq0AEKDgoDYXBwEgcaBXNsZWVwChkKDGlzdGlvLmlvL3JldhIJGgdkZWZhdWx0CiAKEXBvZC10ZW1wbGF0ZS1oYXNoEgsaCWY4Y2JmNWI3NgokChlzZWN1cml0eS5pc3Rpby5pby90bHNNb2RlEgcaBWlzdGlvCioKH3NlcnZpY2UuaXN0aW8uaW8vY2Fub25pY2FsLW5hbWUSBxoFc2xlZXAKLwojc2VydmljZS5pc3Rpby5pby9jYW5vbmljYWwtcmV2aXNpb24SCBoGbGF0ZXN0ChoKB01FU0hfSUQSDxoNY2x1c3Rlci5sb2NhbAofCgROQU1FEhcaFXNsZWVwLWY4Y2JmNWI3Ni13bjlyNwoWCglOQU1FU1BBQ0USCRoHZGVmYXVsdApJCgVPV05FUhJAGj5rdWJlcm5ldGVzOi8vYXBpcy9hcHBzL3YxL25hbWVzcGFjZXMvZGVmYXVsdC9kZXBsb3ltZW50cy9zbGVlcAoaCg9TRVJWSUNFX0FDQ09VTlQSBxoFc2xlZXAKGAoNV09SS0xPQURfTkFNRRIHGgVzbGVlcA==",
       "X-Envoy-Peer-Metadata-Id": "sidecar~10.80.2.26~sleep-f8cbf5b76-wn9r7.default~default.svc.cluster.local"
     }
   }
   ```

#### 卸载

```shell
$ kubectl delete destinationrule my-httpbin
$ kubectl delete service my-httpbin
```

### 使用带endpoints的kubernetes service访问一个外部服务

1. 为`map.baidu.com`创建一个kubernetes service，不带selector

   ```yaml
   $ kubectl apply -f - <<EOF
   kind: Service
   apiVersion: v1
   metadata:
     name: my-baidu-map
   spec:
     ports:
     - protocol: TCP
       port: 443
       name: tls
   EOF
   ```

2. 为外部服务手动创建endpoints，IP来自`map.baidu.com`后端地址。此时可以通过kubernetes service直接访问外部服务

   ```shell
   # nslookup map.baidu.com
   Server:         100.100.2.136
   Address:        100.100.2.136#53
   
   Non-authoritative answer:
   map.baidu.com   canonical name = map.n.shifen.com.
   Name:   map.n.shifen.com
   Address: 180.101.49.69
   ```

   ```shell
   $ kubectl apply -f - <<EOF
   kind: Endpoints
   apiVersion: v1
   metadata:
     name: my-baidu-map
   subsets:
     - addresses:
         - ip: 180.101.49.69
       ports:
         - port: 443
           name: tls
   EOF
   ```

3. 观测上述service，可以通过其cluster IP访问`map.baidu.com`

   ```shell
   # oc get svc my-baidu-map
   NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
   my-baidu-map   ClusterIP   10.84.20.176   <none>        443/TCP   116s
   ```

4. 从不带istio sidecar的pod中向`map.baidu.com`发送HTTPS请求。注意下面`curl`在访问`map.baidu.com`时使用了`--resolve`选项

   ```shell
   # kubectl exec "$SOURCE_POD_WITHOUT_ISTIO" -n without-istio -c sleep -- curl -s --resolve map.baidu.com:443:"$(kubectl get service my-baidu-map -o jsonpath='{.spec.clusterIP}')" https://map.baidu.com | grep -o "<title>.*</title>"
   <title>百度地图</title>
   ```

5. 这种情况下，负载会直接向`map.baidu.com`发送HTTPS请求，此时可以安全禁用istio的mutual TLS(当然也可以不禁用，此时不需要部署destinationRule)

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: my-baidu-map
   spec:
     host: my-baidu-map.default.svc.cluster.local
     trafficPolicy:
       tls:
         mode: DISABLE
   EOF
   ```

6. 从带istio sidecar的pod中访问`map.baidu.com`

   ```shell
   # kubectl exec "$SOURCE_POD" -c sleep -- curl -s --resolve map.baidu.com:443:"$(kubectl get service my-baidu-map -o jsonpath='{.spec.clusterIP}')" https://map.baidu.com  | grep -o "<title>.*</title>"
   <title>百度地图</title>
   ```

7. 校验请求确实是通过cluster IP(`10.84.20.176`)进行访问的。

   ```shell
   # kubectl exec "$SOURCE_POD" -c sleep -- curl -v --resolve map.baidu.com:443:"$(kubectl get service my-baidu-map -o jsonpath='{.spec.clusterIP}')" https://map.baidu.com -o /dev/null
   * Expire in 0 ms for 6 (transfer 0x562c95903680)
   * Added map.baidu.com:443:10.84.20.176 to DNS cache
   * Hostname map.baidu.com was found in DNS cache
   *   Trying 10.84.20.176...
   * TCP_NODELAY set
   ```

#### 卸载

```shell
$ kubectl delete destinationrule my-baidu-map
$ kubectl delete endpoints my-baidu-map
$ kubectl delete service my-baidu-map
```

### 卸载

```shell
$ kubectl delete -f samples/sleep/sleep.yaml
$ kubectl delete -f samples/sleep/sleep.yaml -n without-istio
$ kubectl delete namespace without-istio
$ unset SOURCE_POD SOURCE_POD_WITHOUT_ISTIO
```

## 使用外部HTTPS代理

在前面[配置Egress网关](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/)的例子中展示了如何通过istio边界组件*Egress网关*将流量转发到外部服务中。然而，但是，有些情况下需要外部的、遗留的（非Istio）HTTPS代理来访问外部服务。例如，公司可能已经部署了一个代理，所有组织中的应用都必须通过该代理来转发流量。

本例展示如何通过外部代理转发流量。由于所有的由于都会使用HTTP CONNECT方法来与HTTPS代理建立连接，配置流量到一个外部代理不同于配置流量到外部HTTP和HTTPS服务。

### 部署

创建sleep应用并获取POD名称

```shell
$ kubectl apply -f samples/sleep/sleep.yaml
$ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```

### 部署一个HTTPS代理

为了模拟一个遗留的代理，需要在集群中部署HTTPS代理。为了模拟在集群外部运行的更真实的代理，需要通过代理的IP地址而不是Kubernetes服务的域名来定位代理的pod。本例使用[Squid](http://www.squid-cache.org/)，但也可以使用其他HTTPS代理来支持HTTP CONNECT。

1. 为HTTPS代理创建一个命名空间，不启用istio sidecar自动注入。使用这种方式来模拟集群外的代理。

   ```shell
   $ kubectl create namespace external
   ```

2. 创建Squid代理的配置文件

   ```shell
   $ cat <<EOF > ./proxy.conf
   http_port 3128
   
   acl SSL_ports port 443
   acl CONNECT method CONNECT
   
   http_access deny CONNECT !SSL_ports
   http_access allow localhost manager
   http_access deny manager
   http_access allow all
   
   coredump_dir /var/spool/squid
   EOF
   ```

3. 创建一个kubernetes ConfigMap来保存代理的配置

   ```shell
   $ kubectl create configmap proxy-configmap -n external --from-file=squid.conf=./proxy.conf
   ```

4. 部署Squid容器。注：openshift可能会因为scc导致权限错误，为方便测试，将容器设置为privileged权限

   ```powershell
   # oc adm policy add-scc-to-user privileged -z default
   ```

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: squid
     namespace: external
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: squid
     template:
       metadata:
         labels:
           app: squid
       spec:
         serviceAccount: default
         volumes:
         - name: proxy-config
           configMap:
             name: proxy-configmap
         containers:
         - name: squid
           image: sameersbn/squid:3.5.27
           imagePullPolicy: IfNotPresent
           securityContext:
             privileged: true
           volumeMounts:
           - name: proxy-config
             mountPath: /etc/squid
             readOnly: true
   EOF
   ```

5. 在external命名空间中创建sleep应用来测试到代理的流量(不受istio控制)

   ```shell
   $ kubectl apply -n external -f samples/sleep/sleep.yaml
   ```

6. 获取代理pod的地址并定义在`PROXY_IP`环境变量中

   ```shell
   $ export PROXY_IP="$(kubectl get pod -n external -l app=squid -o jsonpath={.items..podIP})"
   ```

7. 定义`PROXY_PORT`环境变量来保存代理的端口，即Squid使用的端口3128

   ```shell
   $ export PROXY_PORT=3128
   ```

8. 从external命名空间中的sleep Pod中通过代理向外部服务发送请求：

   ```shell
   # kubectl exec "$(kubectl get pod -n external -l app=sleep -o jsonpath={.items..metadata.name})" -n external -- sh -c "HTTPS_PROXY=$PROXY_IP:$PROXY_PORT curl https://map.baidu.com" | grep -o "<title>.*</title>"
     % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
     0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<title>百度地图</title>
   100  154k    0  154k    0     0   452k      0 --:--:-- --:--:-- --:--:--  452k
   ```

9. 检查代理的访问日志：

   ```shell
   # kubectl exec "$(kubectl get pod -n external -l app=squid -o jsonpath={.items..metadata.name})" -n external -- tail /var/log/squid/access.log
   1598596320.477    342 10.80.2.81 TCP_TUNNEL/200 165939 CONNECT map.baidu.com:443 - HIER_DIRECT/180.101.49.69 -
   ```

现在，完成了如下两个于istio无关的任务：

- 部署了HTTPS 代理
- 通过代理访问`map.baidu.com`

下面将配置启用istio的pod使用HTTPS代理。

### 配置流量到外部HTTPS代理

1. 为HTTPS代理定义一个TCP(非HTTP) Service Entry。虽然应用会使用HTTP CONNECT方法来与HTTPS代理建立连接，但必须为代理配置TCP流量，而非HTTP。一旦建立连接，代理只是充当一个TCP隧道。

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1beta1
   kind: ServiceEntry
   metadata:
     name: proxy
   spec:
     hosts:
     - my-company-proxy.com # ignored
     addresses:
     - $PROXY_IP/32 #hosts字段的后端IP
     ports:
     - number: $PROXY_PORT
       name: tcp
       protocol: TCP
     location: MESH_EXTERNAL
   EOF
   ```

2. 从default命名空间中的sleep Pod发送请求，由于该pod带有sidecar，istio会对流量进行控制

   ```shell
   # kubectl exec "$SOURCE_POD" -c sleep -- sh -c "HTTPS_PROXY=$PROXY_IP:$PROXY_PORT curl https://map.baidu.com" | grep -o "<title>.*</title>"
     % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
     0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<title>百度地图</title>
   100  154k    0  154k    0     0   439k      0 --:--:-- --:--:-- --:--:--  439k
   ```

3. 检查istio sidecar 代理的日志，可以看到对外访问了my-company-proxy.com

   ```shell
   # kubectl exec "$SOURCE_POD" -c sleep -- sh -c "HTTPS_PROXY=$PROXY_IP:$PROXY_PORT curl https://map.baidu.com" | grep -o "<title>.*</title>"
   [2020-08-28T12:38:10.064Z] "- - -" 0 - "-" "-" 898 166076 354 - "-" "-" "-" "-" "10.80.2.87:3128" outbound|3128||my-company-proxy.com 10.80.2.77:36576 10.80.2.87:3128 10.80.2.77:36574 - -
   ```

4. 检查代理的访问日志，可以看到转发了HTTP CONNECT请求

   ```shell
   # kubectl exec "$(kubectl get pod -n external -l app=squid -o jsonpath={.items..metadata.name})" -n external -- tail /var/log/squid/access.log
   1598618290.412    346 10.80.2.77 TCP_TUNNEL/200 166076 CONNECT map.baidu.com:443 - HIER_DIRECT/180.101.49.69 -
   ```

### 过程理解

在本例中完成了如下步骤：

1. 部署一个HTTPS代理来模拟外部代理
2. 创建TCP service entry来使istio控制的流量转发到外部代理

注意不能为需要经过外部代理的外部服务(如map.baidu.com)创建service entry。这是因为从istio的角度看，这些请求仅会发送到外部代理，Istio并不知道外部代理会进一步转发请求的事实。

### 卸载

```shell
$ kubectl delete -f samples/sleep/sleep.yaml
$ kubectl delete -f samples/sleep/sleep.yaml -n external
```

```shell
$ kubectl delete -n external deployment squid
$ kubectl delete -n external configmap proxy-configmap
$ rm ./proxy.conf
```

```shell
$ kubectl delete namespace external
```

```shell
$ kubectl delete serviceentry proxy
```

