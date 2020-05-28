# Istio的流量管理(实操三)(istio 系列五)

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

1. 创建一个`ServiceEntry`，允许访问外部HTTP服务

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



































