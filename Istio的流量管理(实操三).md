# shellIstio的流量管理(实操三)(istio 系列五)

涵盖官方文档[Traffic Management](https://istio.io/docs/tasks/traffic-management/)章节中的egress部分。

[TOC]

## 访问外部服务

由于启用了istio的pod的出站流量默认都会被重定向到代理上，因此对集群外部URL的访问取决于代理的配置。默认情况下，Envoy代理透传对未知服务的访问，虽然这种方式为新手提供了便利，但最好配置更严格的访问控制。

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

### Envoy转发流量到外部服务

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

   使用这种方式可以访问外部服务，但没有对该流量进行监控和控制，下面介绍如何监控和控制网格到外部服务的流量。

### 控制访问外部服务

使用ServiceEntry配置可以从istio集群内部访问公共服务。本节展示如何配置访问外部HTTP服务，httpbin.org以及www.baidu.com，同时会监控和控制istio流量。

#### 修改默认的阻塞策略

为了展示如何控制访问外部服务的方式，需要将`meshConfig.outboundTrafficPolicy.mode`设置为`REGISTRY_ONLY`

1. 执行如下命令将`meshConfig.outboundTrafficPolicy.mode`选项设置为`REGISTRY_ONLY`

   ```shell
   $ kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' | kubectl replace -n istio-system -f -
   ```

2. 从SOURCE_POD访问外部HTTPS服务，此时请求会被阻塞

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://www.baidu.com | grep  "HTTP/"; kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://edition.cnn.com | grep "HTTP/"
   command terminated with exit code 35
   command terminated with exit code 35
   ```

#### 访问外部HTTP服务

1. 创建一个`ServiceEntry`，允许访问外部HTTP服务，可以看到此处并没有用到virtual service和destination rule

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
       protocol: HTTPS
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
       - httpbin.org
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

可以配置Envoy sidecar，使其不拦截特定IP段的请求。为了实现该功能，可以修改`global.proxy.includeIPRanges`或`global.proxy.excludeIPRanges`[配置选项](https://archive.istio.io/v1.4/docs/reference/config/installation-options/)(*类似白名单和黑名单*)，并使用`kubectl apply`命令更新`istio-sidecar-injector`配置。也可以修改annotations  `traffic.sidecar.istio.io/includeOutboundIPRanges`达到相同的效果。在更新`istio-sidecar-injector`配置后，相应的变动会影响到所有的应用pod。

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

```shell
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: edition-cnn-com
spec:
  hosts:
  - edition.cnn.com
  ports:
  - number: 80     # HTTP访问
    name: http-port
    protocol: HTTP
  - number: 443    # HTTPS访问
    name: https-port
    protocol: HTTPS
  resolution: DNS
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: edition-cnn-com
spec:
  hosts:
  - edition.cnn.com #客户端访问的外部地址
  tls:
  - match:
    - port: 443
      sniHosts:
      - edition.cnn.com
    route:
    - destination:
        host: edition.cnn.com #与gateway不同，此处并没有将virtualservice与serviceentry进行绑定
        port:
          number: 443
      weight: 100
EOF
```

访问外部服务，下面使用了`-L`选项使请求端依照返回的重定向信息重新发起请求。第一个请求会发往`http://edition.cnn.com/politics`，服务端会返回重定向信息，第二个请求会按照重定向信息发往`https://edition.cnn.com/politics`。可以看到第一次是`HTTP`访问，第二次是`HTTPS`访问。

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

重新定义 `ServiceEntry` 和`VirtualService` ，并增加`DestinationRule`来执行TLS源。此时`VirtualService`会将HTTP请求流量从80端口重定向到`DestinationRule`的443端口，作为TLS源。与前面不同，此时客户端只能通过HTTP进行访问。

```shell
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
    - port: 80
    route:
    - destination:
        host: edition.cnn.com #
        subset: tls-origination
        port:
          number: 443
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: edition-cnn-com
spec:
  host: edition.cnn.com #此处与serviceEntry注册的hosts相同，将流量发往serviceEntry
  subsets:
  - name: tls-origination
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
      portLevelSettings:
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

假设在一个安全要求比较高的组织中，所有离开服务网格的流量都要经过一个指定的节点，这些节点会运行在指定的机器上，与运行应用的集群的节点分开。这些特定的节点会在出站流量上应用策略，且对这些节点的监控将更加严格。

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
$ istioctl manifest apply -f cni-annotations.yaml --set values.global.istioNamespace=istio-system \
    --set values.gateways.istio-ingressgateway.enabled=false \
    --set values.gateways.istio-egressgateway.enabled=true
```

> 注意：apply的时候使用自己定制化的文件，否则系统会使用默认的profile，导致配置丢失！
>
> 下面操作关于在`default`命名空间中为egress网关创建destination rule，因此要求`sleep`应用也部署在`default`命名空间中。如果应用不在default命名空间中，将无法在[destination rule查找路径](https://istio.io/docs/ops/best-practices/traffic-management/#cross-namespace-configuration)找到destination rule，客户端请求将会失败。

### HTTP流量的egress gateway

1. 创建一个`ServiceEntry`允许流量访问外部服务`edition.cnn.com`:

   ```shell
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

3. 为*edition.cnn.com*创建一个`Gateway`，端口80，并定义一个destination rule将流量定向到egress网关。

   ```shell
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
         number: 80
         name: http
         protocol: HTTP
       hosts:
       - edition.cnn.com
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: egressgateway-for-cnn
   spec:
     host: istio-egressgateway.istio-system.svc.cluster.local
     subsets:
     - name: cnn
   EOF
   ```

4. 定义`VirtualService`，将流量从sidecar定向到egress网关，然后将流量从egress网关定向到外部服务。

   ```shell
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: direct-cnn-through-egress-gateway
   spec:
     hosts:
     - edition.cnn.com
     gateways: #定义应用路由规则的网关
     - istio-egressgateway
     - mesh #istio保留字段，表示网格中的所有sidecar，当忽略gateways字段时，默认会使用mesh
     http:
     - match: #各个match是OR关系
       - gateways: #将来自gateway mesh且端口为80的请求发往istio-egressgateway.istio-system.svc.cluster.local:80
         - mesh
         port: 80
       route:
       - destination:
           host: istio-egressgateway.istio-system.svc.cluster.local
           subset: cnn #对于DestinationRule中的subset名
           port:
             number: 80
         weight: 100
     - match:
       - gateways:
         - istio-egressgateway #将来自gateway istio-egressgateway且端口为80的请求发往edition.cnn.com:80
         port: 80
       route:
       - destination:
           host: edition.cnn.com
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

6. 校验egress日志(本环境并没有官方所示的打印。)

   ```shell
   $ kubectl logs -l istio=egressgateway -c istio-proxy -n istio-system | tail
   ```

#### 卸载

```shell
$ kubectl delete gateway istio-egressgateway
$ kubectl delete serviceentry cnn
$ kubectl delete virtualservice direct-cnn-through-egress-gateway
$ kubectl delete destinationrule egressgateway-for-cnn
```

### HTTPS流量的egress gateway

本节展示通过egress网关定向HTTPS流量。会使用到`ServiceEntry`，一个egress `Gateway`和一个`VirtualService`。

1. 创建到`edition.cnn.com`的`ServiceEntry`

   ```shell
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
       protocol: TLS
     resolution: DNS
   EOF
   ```

2. 校验可以通过`ServiceEntry`访问https://edition.cnn.com/politics

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - https://edition.cnn.com/politics
   HTTP/2 200
   ...
   ```

3. 为*edition.cnn.com*创建egress `Gateway`，一个destination rule和一个virtual service。与上面支持HTTP egress 网关的不同点参见下面标注的内容，其他字段相同。

4. 访问https://edition.cnn.com/politics

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - https://edition.cnn.com/politics
   HTTP/2 200
   ...
   ```

5. 校验log

   ```shell
   $ kubectl logs -l istio=egressgateway -n istio-system
   ```

#### 卸载

```shell
$ kubectl delete serviceentry cnn
$ kubectl delete gateway istio-egressgateway
$ kubectl delete virtualservice direct-cnn-through-egress-gateway
$ kubectl delete destinationrule egressgateway-for-cnn
```

### 安全考量

istio不能保证所有通过egress网关出去的流量的安全性，仅能保证通过sidecar代理的流量的安全性。如果攻击者绕过了sidecar代理，既可以不经过egress网关直接访问外部服务。此时，攻击者的行为不受istio的控制和监控。集群管理员或云供应商必须保证所有的流量都要经过egress网关。例如，集群管理员可以配置一个防火墙，拒绝所有非egress网关的流量。[Kubernetes network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)也可以禁止所有非egress网关的流量。此外，集群管理员或云供应商可以配置网络来保证应用节点只能通过网关访问因特网，为了实现这种效果，需要阻止将公共IP分配给网关以外的pod，并配置NAT设备丢弃非egress网关的报文。

### 使用Kubernetes network policies

本节展示如何创建一个[Kubernetes network policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)来防止绕过egress网关。为了测试网络策略，需要创建一个命名空间`test-egress`，部署`sleep`应用，并尝试向网关安全的外部服务发送请求。

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

   ```shell
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

    ```shell
    $ kubectl apply -n test-egress -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: egressgateway-for-cnn
    spec:
      host: istio-egressgateway.istio-system.svc.cluster.local
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



















