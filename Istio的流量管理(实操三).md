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

1. 创建一个`ServiceEntry`注册外部服务，这样就可以直接访问外部HTTP服务，可以看到此处并没有用到virtual service和destination rule

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
  - match: #将edition.cnn.com:443的流量分发到edition.cnn.com:443
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
        host: edition.cnn.com #此时定义了DestinationRule，会优先经过DestinationRule处理，再转发到serviceEntry
        subset: tls-origination
        port:
          number: 443
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: edition-cnn-com
spec:
  host: edition.cnn.com #此处与serviceEntry注册的hosts相同，配置发往serviceEntry的流量规则
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

### HTTP流量的egress网关

1. 上面例子中，当网格内的客户端可以直接访问外部服务，此处将会创建一个egress网关，内部流量访问外部服务时会经过该网关。创建一个`ServiceEntry`允许流量访问外部服务`edition.cnn.com`:

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
     - port: #接收来自edition.cnn.com:80的流量
         number: 80
         name: http
         protocol: HTTP
       hosts:
       - edition.cnn.com
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule #该DestinationRule没有定义任何规则，实际可用删除该DestinationRule，并删除下面VirtualService的"subset: cnn"一行
   metadata:
     name: egressgateway-for-cnn
   spec:
     host: istio-egressgateway.istio-system.svc.cluster.local
     subsets:
     - name: cnn #下面VirtualService中会用到
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
     gateways: #罗列应用路由规则的网关
     - istio-egressgateway
     - mesh #istio保留字段，表示网格中的所有sidecar，当忽略gateways字段时，默认会使用mesh，此处表示将所有sidecar到edition.cnn.com的请求
     http:
     - match: #各个match是OR关系
       - gateways: #处理mesh网关，将来自mesh的edition.cnn.com:80请求发往istio-egressgateway.istio-system.svc.cluster.local:80
         - mesh
         port: 80
       route:
       - destination:
           host: istio-egressgateway.istio-system.svc.cluster.local
           subset: cnn #对应DestinationRule中的subset名，由于使用了subset，因此必须使用DestinationRule。删除改行后就可以不定义上面的DestinationRule
           port:
             number: 80
         weight: 100
     - match:
       - gateways:
         - istio-egressgateway #处理istio-egressgateway网关，将来自gateway的edition.cnn.com:80请求发往edition.cnn.com:80
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

6. 校验egress日志（需要[启用](https://istio.io/docs/tasks/observability/logs/access-log/#enable-envoy-s-access-logging)Envoy日志）

   ```shell
   $ kubectl logs -l istio=egressgateway -c istio-proxy -n istio-system | tail
   [2020-06-02T08:53:08.928Z] "GET /politics HTTP/2" 200 - "-" "-" 0 1297548 5590 3641 "10.80.3.25" "curl/7.64.0" "8b131a2a-eb16-4f99-98e3-dbbdacae710f" "edition.cnn.com" "151.101.129.67:443" outbound|443||edition.cnn.com 10.83.1.219:54718 10.83.1.219:80 10.80.3.25:41204 - -
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

1. 创建到`edition.cnn.com`的`ServiceEntry`，定义外部服务https://edition.cnn.com

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

istio不能保证所有通过egress网关出去的流量的安全性，仅能保证通过sidecar代理的流量的安全性。如果攻击者绕过了sidecar代理，既可以不经过egress网关直接访问外部服务。此时，攻击者的行为不受istio的控制和监控。集群管理员或云供应商必须保证所有的流量都要经过egress网关。例如，集群管理员可以配置一个防火墙，拒绝所有非egress网关的流量。[Kubernetes network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)也可以禁止所有非egress网关的流量。此外，集群管理员或云供应商可以配置网络来保证应用节点只能通过网关访问因特网，为了实现这种效果，需要阻止将公共IP分配给网关以外的pod，并配置NAT设备丢弃非egress网关的报文。

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

## 带TLS源的Egress网关

本节展示如何通过配置一个egress网关来为到外部服务的流量发起TLS。

### 部署

部署sleep应用，并获取其Pod名

```shell
$ kubectl apply -f samples/sleep/sleep.yaml
$ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```

通过`meshConfig.accessLogFile`启用Envoy访问日志

```shell
$ istioctl manifest apply -f cni-annotations.yaml --set values.global.istioNamespace=istio-system --set values.gateways.istio-ingressgateway.enabled=true --set values.gateways.istio-egressgateway.enabled=true --set meshConfig.accessLogFile="/dev/stdout"
```

### 使用Egress网关发起TLS

本节描述使用Egress网关作为TLS源向外部服务发起HTTPS请求。

1. 为`edition.cnn.com`定义一个`ServiceEntry`:

   ```shell
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: cnn
   spec:
     hosts:
     - edition.cnn.com
     ports: #可以通过80和443端口访问edition.cnn.com
     - number: 80
       name: http
       protocol: HTTP
     - number: 443
       name: https
       protocol: HTTPS
     resolution: DNS
   EOF
   ```

2. 校验可以通过`ServiceEntry`直接访问 [http://edition.cnn.com/politics](https://edition.cnn.com/politics)，此时没有经过任何网关

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics
   HTTP/1.1 301 Moved Permanently
   ...
   
   HTTP/2 200
   ...
   ```

   当然也可以访问[https://edition.cnn.com/politics](https://edition.cnn.com/politics)

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - https://edition.cnn.com/politics
   HTTP/1.1 200 OK
   ...
   ```

3. 为 *edition.cnn.com*创建一个`Gateway`，端口为80，以及一个destination rule，将流量定向到egress网关

   ```shell
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3 #定义Gateway
   kind: Gateway
   metadata:
     name: istio-egressgateway
   spec:
     selector:
       istio: egressgateway
     servers:
     - port:
         number: 80
         name: http-port-for-tls-origination
         protocol: HTTP
       hosts:
       - edition.cnn.com
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: egressgateway-for-cnn
   spec:
     host: istio-egressgateway.istio-system.svc.cluster.local #将流量定向到上面定义的Gateway,istio-egressgateway为网关的service名称
     subsets:
     - name: cnn
   EOF
   ```

4. 定义一个`VirtualService`，使流量通过egress网关，并定义一个`DestinationRule`为到 `edition.cnn.com`的请求发起TLS协商

   ```shell
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
       - gateways:
         - mesh #使用istio-egressgateway规则处理到edition.cnn.com:80的流量
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
         - istio-egressgateway #使用originate-tls-for-edition-cnn-com规则处理istio-egressgateway:80流量
         port: 80
       route:
       - destination:
           host: edition.cnn.com
           port:
             number: 443
         weight: 100
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule #配置tls
   metadata:
     name: originate-tls-for-edition-cnn-com
   spec:
     host: edition.cnn.com
     trafficPolicy:
       loadBalancer:
         simple: ROUND_ROBIN
       portLevelSettings: #选择443端口的流量
       - port:
           number: 443
         tls:
           mode: SIMPLE # initiates HTTPS for connections to edition.cnn.com
   EOF
   ```

5. 向 [http://edition.cnn.com/politics](https://edition.cnn.com/politics)发起HTTP请求，请求成功且没有重定向。

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics
   HTTP/1.1 200 OK
   ...
   ```

6. 校验 `istio-egressgateway`  pod的日志

   ```shell
   $ kubectl logs -l istio=egressgateway -c istio-proxy -n istio-system | tail
   ...
   [2020-06-02T08:53:08.928Z] "GET /politics HTTP/2" 200 - "-" "-" 0 1297548 5590 3641 "10.80.3.25" "curl/7.64.0" "8b131a2a-eb16-4f99-98e3-dbbdacae710f" "edition.cnn.com" "151.101.129.67:443" outbound|443||edition.cnn.com 10.83.1.219:54718 10.83.1.219:80 10.80.3.25:41204 - -
   ```

#### 卸载

```shell
$ kubectl delete gateway istio-egressgateway
$ kubectl delete serviceentry cnn
$ kubectl delete virtualservice direct-cnn-through-egress-gateway
$ kubectl delete destinationrule originate-tls-for-edition-cnn-com
$ kubectl delete destinationrule egressgateway-for-cnn
```

### 使用Egress网关执行双向TLS认证

与前一节类似，本节描述了如何配置egress网关来实现双向TLS认证。涉及如下步骤：

1. 生成客户端和服务端证书。
2. 部署外部服务来支持双向TLS协议
3. 重新部署egress网关，使用双向TLS证书

#### 生成客户端和服务端证书

1. clone  https://github.com/nicholasjackson/mtls-go-example库

   ```shell
   $ git clone https://github.com/nicholasjackson/mtls-go-example
   ```

2. 使用上面库中的脚本为`nginx.example.com`生成证书，传入自定义的密码，按`y`确认

   ```shell
   $ ./generate.sh nginx.example.com <password>
   ```

3. 将证书转移到目录中

   ```shell
   $ mkdir ../nginx.example.com && mv 1_root 2_intermediate 3_application 4_client ../nginx.example.com
   ```

4. 回到上一级目录，可以看到有一个`nginx.example.com`目录，保存了所有的证书

   ```shell
   $ cd ..
   ```

#### 部署双向TLS服务

为了模拟支持双向TLS协议的实际外部服务，在kubernetes集群中部署一个NGINX服务，在该服务运行在istio服务网格之外，即运行在不会注入istio sidecar的命名空间中。

1. 创建一个命名空间来表示istio网格外的服务，名为 `mesh-external`。注意不会对该命名空间中的pod自动注入sidecar

   ```shell
   $ kubectl create namespace mesh-external
   ```

2. 创建kubernetes secret来保存服务的CA证书

   ```shell
   $ kubectl create -n mesh-external secret tls nginx-server-certs --key nginx.example.com/3_application/private/nginx.example.com.key.pem --cert nginx.example.com/3_application/certs/nginx.example.com.cert.pem #app的密钥和证书
   
   $ kubectl create -n mesh-external secret generic nginx-ca-certs --from-file=nginx.example.com/2_intermediate/certs/ca-chain.cert.pem  #签署app证书的ca证书
   ```

3. 为NGINX服务创建配置文件：

   ```shell
   cat <<EOF > ./nginx.conf
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
   
       server_name nginx.example.com;
       ssl_certificate /etc/nginx-server-certs/tls.crt; #服务端证书
       ssl_certificate_key /etc/nginx-server-certs/tls.key; #服务端密钥
       ssl_client_certificate /etc/nginx-ca-certs/ca-chain.cert.pem; #签署服务端证书的CA证书
       ssl_verify_client on; #nginx启用双向认证
     }
   }
   EOF
   ```

4. 创建一个kubernetes configmap来保存NGINX服务的配置文件

   ```shell
   $ kubectl create configmap nginx-configmap -n mesh-external --from-file=nginx.conf=./nginx.conf
   ```

5. 部署NGINX服务

   openshift下首先需要赋予Pod privileged权限

   ```shell
   $ oc adm policy add-scc-to-user anyuid system:serviceaccount:mesh-external:default
   ```

   ```shell
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
           securityContext:
             privileged: true
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
         - name: nginx-server-certs #nginx的密钥和证书
           secret:
             secretName: nginx-server-certs
         - name: nginx-ca-certs #签署nginx证书的CA证书
           secret:
             secretName: nginx-ca-certs
   EOF
   ```

6. 为`nginx.example.com`部署一个`ServiceEntry`和一个`VirtualService`将到`nginx.example.com`的流量定向到NGINX服务

   ```shell
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: ServiceEntry
   metadata:
     name: nginx
   spec:
     hosts:
     - nginx.example.com #将流量从nginx.example.com:443分发到my-nginx.mesh-external.svc.cluster.local:443
     ports:
     - number: 80
       name: http
       protocol: HTTP
     - number: 443
       name: https
       protocol: HTTPS
     resolution: DNS
     endpoints: #指定nginx.example.com的后端，如果没有该字段，则endpoints与hosts字段相同
     - address: my-nginx.mesh-external.svc.cluster.local
       ports:
         https: 443
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: nginx
   spec:
     hosts:
     - nginx.example.com
     tls:
     - match:
       - port: 443
         sniHosts:
         - nginx.example.comm:
       route: #将网格内部的nginx.example.com:443流量定向到nginx.example.com:443(即上面serviceEntry注册的服务)
       - destination:
           host: nginx.example.com
           port:
             number: 443
         weight: 100
   EOF
   ```

##### 部署一个容器来测试NGINX服务

1. 由于nginx开启了双向认证，因此客户端需要提供由服务端CA证书签署的客户端证书。创建kubernetes secret来保存客户端的CA证书，注意，客户端不能部署在`mesh-external`命名空间中，否则会有证书问题。

   ```shell
   $ kubectl create secret tls nginx-client-certs --key nginx.example.com/4_client/private/nginx.example.com.key.pem --cert nginx.example.com/4_client/certs/nginx.example.com.cert.pem #客户端的密钥和证书
   
   $ kubectl create secret generic nginx-ca-certs --from-file=nginx.example.com/2_intermediate/certs/ca-chain.cert.pem #签署客户端证书的CA，与服务端CA相同
   ```

2. 部署`sleep`应用，挂载了客户端证书和CA证书，测试发往NGINX发往的请求

   ```shell
   $ kubectl apply -f - <<EOF
   # Copyright 2017 Istio Authors
   #
   #   Licensed under the Apache License, Version 2.0 (the "License");
   #   you may not use this file except in compliance with the License.
   #   You may obtain a copy of the License at
   #
   #       http://www.apache.org/licenses/LICENSE-2.0
   #
   #   Unless required by applicable law or agreed to in writing, software
   #   distributed under the License is distributed on an "AS IS" BASIS,
   #   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   #   See the License for the specific language governing permissions and
   #   limitations under the License.
   
   ##################################################################################################
   # Sleep service
   ##################################################################################################
   apiVersion: v1
   kind: Service
   metadata:
     name: sleep
     labels:
       app: sleep
   spec:
     ports:
     - port: 80
       name: http
     selector:
       app: sleep
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: sleep
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: sleep
     template:
       metadata:
         labels:
           app: sleep
       spec:
         containers:
         - name: sleep
           image: tutum/curl
           command: ["/bin/sleep","infinity"]
           imagePullPolicy: IfNotPresent
           volumeMounts:
           - name: nginx-client-certs
             mountPath: /etc/nginx-client-certs
             readOnly: true
           - name: nginx-ca-certs
             mountPath: /etc/nginx-ca-certs
             readOnly: true
         volumes:
         - name: nginx-client-certs #客户端证书和密钥
           secret:
             secretName: nginx-client-certs
         - name: nginx-ca-certs #签署客户端证书的CA证书
           secret:
             secretName: nginx-ca-certs
   EOF
   ```

3. 定义环境变量来保存`sleep` Pod名字

   ```shell
   $ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
   ```

4. 使用部署的`sleep` pod发送请求到NGINX服务。由于`nginx.example.com`实际上并不存在，且DNS无法解析，因此在`curl`命令中使用`--resolve`选项手动解析主机名。下面的`--resolve`选项中的IP值(1.1.1.1)并不重要，可以使用任何非127.0.0.1的地址。如果目的主机名存在DNS表项中，则无需在curl中使用`--resolve`选项。

   ```shell
   $ kubectl exec -it $SOURCE_POD -c sleep -- curl -v --resolve nginx.example.com:443:1.1.1.1 --cacert /etc/nginx-ca-certs/ca-chain.cert.pem --cert /etc/nginx-client-certs/tls.crt --key /etc/nginx-client-certs/tls.key https://nginx.example.com
   
   * Added nginx.example.com:443:1.1.1.1 to DNS cache
   * Rebuilt URL to: https://nginx.example.com/
   ...
   * Server certificate:
   *        subject: C=US; ST=Denial; L=Springfield; O=Dis; CN=nginx.example.com
   *        start date: 2020-06-02 09:39:23 GMT
   *        expire date: 2021-06-12 09:39:23 GMT
   *        common name: nginx.example.com (matched)
   *        issuer: C=US; ST=Denial; O=Dis; CN=nginx.example.com
   *        SSL certificate verify ok.
   > GET / HTTP/1.1
   > User-Agent: curl/7.35.0
   > Host: nginx.example.com
   > Accept: */*
   >
   < HTTP/1.1 200 OK
   * Server nginx/1.19.0 is not blacklisted
   < Server: nginx/1.19.0
   ...
   ```

5. 校验服务端是需要客户端证书的

   ```shell
   $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name}) -c sleep -- curl -k --resolve nginx.example.com:443:1.1.1.1 https://nginx.example.com
   <html>
   <head><title>400 No required SSL certificate was sent</title></head>
   <body>
   <center><h1>400 Bad Request</h1></center>
   <center>No required SSL certificate was sent</center>
   <hr><center>nginx/1.19.0</center>
   </body>
   </html>
   ```

#### 重部署带客户端证书的Egress网关

1. 在`istio-system`命名空间中创建一个保存客户端证书和CA证书的kubernetes secret。下面的证书与前面的客户端证书相同。

   ```shell
   $ kubectl create -n istio-system secret tls nginx-client-certs --key nginx.example.com/4_client/private/nginx.example.com.key.pem --cert nginx.example.com/4_client/certs/nginx.example.com.cert.pem
   
   $ kubectl create -n istio-system secret generic nginx-ca-certs --from-file=nginx.example.com/2_intermediate/certs/ca-chain.cert.pem
   ```

2. 更新 `istio-egressgateway` deployment，挂载创建的secret。创建如下gateway-patch.json更新 `istio-egressgateway` deployment。

   ```shell
   $ cat > gateway-patch.json <<EOF
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

3. 执行如下命令更新`istio-egressgateway` deployment 

   ```shell
   $ kubectl -n istio-system patch --type=json deploy istio-egressgateway -p "$(cat gateway-patch.json)"
   ```

4. 校验`istio-egressgateway` pod成功加载了密钥和证书，可以看到挂载成功。`/etc/istio/nginx-client-certs`目录中存在`tls.crt`和`tls.key`，`/etc/istio/nginx-ca-certs`中存在`ca-chain.cert.pem`。

   ```shell
   $ kubectl exec -it -n istio-system $(kubectl -n istio-system get pods -l istio=egressgateway -o jsonpath='{.items[0].metadata.name}') -- ls -al /etc/istio/nginx-client-certs /etc/istio/nginx-ca-certs
   /etc/istio/nginx-ca-certs:
   total 8
   drwxrwxrwt. 3 root root  100 Jun  2 15:08 .
   drwxr-xr-x. 1 root root 4096 Jun  2 15:08 ..
   drwxr-xr-x. 2 root root   60 Jun  2 15:08 ..2020_06_02_15_08_05.854591693
   lrwxrwxrwx. 1 root root   31 Jun  2 15:08 ..data -> ..2020_06_02_15_08_05.854591693
   lrwxrwxrwx. 1 root root   24 Jun  2 15:08 ca-chain.cert.pem -> ..data/ca-chain.cert.pem
   
   /etc/istio/nginx-client-certs:
   total 8
   drwxrwxrwt. 3 root root  120 Jun  2 15:08 .
   drwxr-xr-x. 1 root root 4096 Jun  2 15:08 ..
   drwxr-xr-x. 2 root root   80 Jun  2 15:08 ..2020_06_02_15_08_05.976323235
   lrwxrwxrwx. 1 root root   31 Jun  2 15:08 ..data -> ..2020_06_02_15_08_05.976323235
   lrwxrwxrwx. 1 root root   14 Jun  2 15:08 tls.crt -> ..data/tls.crt
   lrwxrwxrwx. 1 root root   14 Jun  2 15:08 tls.key -> ..data/tls.key
   ```

   #### 为Egress网关配置双向TLS源

   1. 为`nginx.example.com`创建`Gateway`，端口443，以及destination rule和virtual service将流量定向到egress网关，并将流量从egress网关定向到外部服务

      ```shell
      $ kubectl apply -f - <<EOF
      apiVersion: networking.istio.io/v1alpha3
      kind: Gateway
      metadata:
        name: istio-egressgateway
      spec:
        selector:
          istio: egressgateway #选择特定的egress
        servers:
        - port:
            number: 443
            name: https
            protocol: HTTPS
          hosts:
          - nginx.example.com
          tls:
            mode: MUTUAL #下面是服务端证书配置，参见https://istio.io/docs/reference/config/networking/gateway/#Server
            serverCertificate: /etc/certs/cert-chain.pem
            privateKey: /etc/certs/key.pem
            caCertificates: /etc/certs/root-cert.pem
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
                number: 443 #指定特定端口的tls策略
              tls:
                mode: ISTIO_MUTUAL
                sni: nginx.example.com
      EOF
      ```

   2. 定义一个`VirtualService`使流量经过到egress网关，以及一个`DestinationRule`执行双向TLS源

      ```shell
      $ kubectl apply -f - <<EOF
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: direct-nginx-through-egress-gateway
      spec:
        hosts:
        - nginx.example.com
        gateways:
        - istio-egressgateway
        - mesh
        http:
        - match:
          - gateways:
            - mesh
            port: 80 #将内部的nginx.example.com:80(可以看出原始流量为HTTP)定向到istio-egressgateway.istio-system.svc.cluster.local:443
          route:
          - destination:
              host: istio-egressgateway.istio-system.svc.cluster.local
              subset: nginx #使用上面定义的DestinationRule egressgateway-for-nginx规则处理
              port:
                number: 443
            weight: 100
        - match:
          - gateways:
            - istio-egressgateway #将istio-egressgateway:443定向到nginx.example.com:443
            port: 443
          route:
          - destination:
              host: nginx.example.com #会使用下面DestinationRule originate-mtls-for-nginx的规则处理，将HTTP流量转变为HTTPS
              port:
                number: 443
            weight: 100
      ---
      apiVersion: networking.istio.io/v1alpha3
      kind: DestinationRule
      metadata:
        name: originate-mtls-for-nginx
      spec:
        host: nginx.example.com
        trafficPolicy:
          loadBalancer:
            simple: ROUND_ROBIN
          portLevelSettings: #针对特定端口443的流量策略
          - port:
              number: 443
            tls:
              mode: MUTUAL #下面是egress网关部署的客户端的证书(参见https://istio.io/docs/reference/config/networking/destination-rule/#ClientTLSSettings),为nginx.example.com:443配置证书。
              clientCertificate: /etc/istio/nginx-client-certs/tls.crt
              privateKey: /etc/istio/nginx-client-certs/tls.key
              caCertificates: /etc/istio/nginx-ca-certs/ca-chain.cert.pem
              sni: nginx.example.com
      EOF
      ```

   3. 发送HTTPS请求到`http://nginx.example.com`:

      ```shell
      $ kubectl exec -it $SOURCE_POD -c sleep -- curl -s --resolve nginx.example.com:80:1.1.1.1 http://nginx.example.com
      
      ```

   4. 校验`istio-egressgateway` pod的日志

      ```shell
      $ kubectl logs -l istio=egressgateway -n istio-system | grep 'nginx.example.com' | grep HTTP
      ```

   ### 卸载

   删除kubernetes资源

   ```shell
   $ kubectl delete secret nginx-server-certs nginx-ca-certs -n mesh-external
   $ kubectl delete secret nginx-client-certs nginx-ca-certs
   $ kubectl delete secret nginx-client-certs nginx-ca-certs -n istio-system
   $ kubectl delete configmap nginx-configmap -n mesh-external
   $ kubectl delete service my-nginx -n mesh-external
   $ kubectl delete deployment my-nginx -n mesh-external
   $ kubectl delete namespace mesh-external
   $ kubectl delete gateway istio-egressgateway
   $ kubectl delete serviceentry nginx
   $ kubectl delete virtualservice direct-nginx-through-egress-gateway
   $ kubectl delete destinationrule originate-mtls-for-nginx
   $ kubectl delete destinationrule egressgateway-for-nginx
   ```

   删除证书

   ```shell
   $ rm -rf nginx.example.com mtls-go-example
   ```

   删除生成的文件

   ```shell
   $ rm -f ./nginx.conf ./istio-egressgateway.yaml
   ```

   上面有问题，要调试











