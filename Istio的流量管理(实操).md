# Istio的流量管理(实操)(istio 系列三)

使用官方的[Bookinfo](https://istio.io/docs/examples/bookinfo/)应用进行测试。涵盖官方文档[Traffic Management](https://istio.io/docs/tasks/traffic-management/)章节中的请求路由，故障注入，流量迁移，TCP流量迁移，请求超时，熔断处理和流量镜像。不含ingress和Egree，后续再补充。

[TOC]

## 部署Bookinfo应用

### Bookinfo应用说明

官方提供的测试应用如下，包含如下4个组件：

- `productpage`： `productpage` 服务会调用`details` 和 `reviews`来填充web页面.
- `details`：`details` 服务包含book信息.
- `reviews`： `reviews` 服务包含书评，它会调用 `ratings` 服务.
- `ratings`：`ratings` 服务包与书评相关的含排名信息

`reviews` 包含3个版本：

- v1版本不会调用 `ratings` 服务.
- v2版本会调用 `ratings` 服务,并按照1到5的黑色星展示排名
- v2版本会调用 `ratings` 服务,并按照1到5的红色星展示排名

![](./images/bookinfo_application.png)

### 部署

Bookinfo应用部署在`default`命名空间下，使用自动注入sidecar的方式：

- 通过如下命令在`default`命名空间(当然也可以部署在其他命名空间下面，Bookinfo配置文件中并没有指定部署的命名空间)中启用自动注入sidecar:

  ```shell
  $ cat <<EOF | oc -n <target-namespace> create -f -
  apiVersion: "k8s.cni.cncf.io/v1"
  kind: NetworkAttachmentDefinition
  metadata:
    name: istio-cni
  EOF
  ```

  ```shell
  $ kubectl label namespace default istio-injection=enabled
  ```

- 切换在`default`命名空间下，部署Bookinfo应用：

  ```shell
  $ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
  ```

  等待一段时间，Bookinfo的所有pod就可以成功启动，查看pod和service：

  ```shell
  $ oc get pod
  NAME                              READY   STATUS    RESTARTS   AGE
  details-v1-78d78fbddf-5mfv9       2/2     Running   0          2m27s
  productpage-v1-85b9bf9cd7-mfn47   2/2     Running   0          2m27s
  ratings-v1-6c9dbf6b45-nm6cs       2/2     Running   0          2m27s
  reviews-v1-564b97f875-ns9vz       2/2     Running   0          2m27s
  reviews-v2-568c7c9d8f-6r6rq       2/2     Running   0          2m27s
  reviews-v3-67b4988599-ddknm       2/2     Running   0          2m27s
  ```

  ```shell
  $ oc get svc                                              
  NAME          TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
  details       ClusterIP      10.84.97.183   <none>        9080/TCP   3m33s
  kubernetes    ClusterIP      10.84.0.1      <none>        443/TCP    14d
  productpage   ClusterIP      10.84.98.111   <none>        9080/TCP   3m33s
  ratings       ClusterIP      10.84.237.68   <none>        9080/TCP   3m33s
  reviews       ClusterIP      10.84.39.249   <none>        9080/TCP   3m33s
  ```

  使用如下命令判断Bookinfo应用是否正确安装：

  ```shell
  $ kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
  
  <title>Simple Bookstore App</title> #返回的结果
  ```

  也可以直接通过svc的endpoint进行访问

  ```shell
  $ oc describe svc productpage|grep Endpoint
  Endpoints:         10.83.1.85:9080
  ```

  ```shell
  $ curl -s 10.83.1.85:9080/productpage | grep -o "<title>.*</title>"
  ```

  可在openshift中创建`router`(属于kuberenetes的ingress gateway)进行访问(将${HOST_NAME}替换为实际的主机名)

  ```yaml
  kind: Route
  apiVersion: route.openshift.io/v1
  metadata:
    name: productpage
    namespace: default
    labels:
      app: productpage
      service: productpage
    annotations:
      openshift.io/host.generated: 'true'
  spec:
    host: ${HOST_NAME}
    to:
      kind: Service
      name: productpage
      weight: 100
    port:
      targetPort: http
    wildcardPolicy: None
  ```

  > 此处先不根据官方文档配置ingress，后续再配置

- 配置默认的destination rules

  配置带mutual TLS(一开始学习istio时不建议配置)

  ```shell
  $ kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
  ```

  配置不带mutual TLS

  ```shell
  $ kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
  ```

  获取配置的destination rules

  ```shell
  $ kubectl get destinationrules -o yaml
  ```

  获取到的destination rules如下，注意默认安装下，除了`reviews`外的service只有`v1`版本

  ```yaml
  - apiVersion: networking.istio.io/v1beta1
    kind: DestinationRule
    metadata:
      annotations:
        ...
      name: details
      namespace: default
    spec:
      host: details    #对应kubernetes service "details"
      subsets:
      - labels:        #实际的details的deployment只有一个标签"version: v1"
          version: v1
        name: v1
      - labels:
          version: v2
        name: v2
  	  
  - apiVersion: networking.istio.io/v1beta1
    kind: DestinationRule
    metadata:
      annotations:
        ...
      name: productpage
      namespace: default
    spec:
      host: productpage
      subsets:
      - labels:
          version: v1
        name: v1
  	  
  - apiVersion: networking.istio.io/v1beta1
    kind: DestinationRule
    metadata:
      annotations:
        ...
      name: ratings
      namespace: default
    spec:
      host: ratings
      subsets:
      - labels:
          version: v1
        name: v1
      - labels:
          version: v2
        name: v2
      - labels:
          version: v2-mysql
        name: v2-mysql
      - labels:
          version: v2-mysql-vm
        name: v2-mysql-vm
  	  
  - apiVersion: networking.istio.io/v1beta1
    kind: DestinationRule
    metadata:
      annotations:
        ...
      name: reviews     # kubernetes service "reviews"实际中有3个版本
      namespace: default
    spec:
      host: reviews
      subsets:
      - labels:
          version: v1
        name: v1
      - labels:
          version: v2
        name: v2
      - labels:
          version: v3
        name: v3
  ```

### 卸载

使用如下命令可以卸载Bookinfo

```shell
$ samples/bookinfo/platform/kube/cleanup.sh
```

## 流量管理

### [请求路由](https://istio.io/docs/tasks/traffic-management/request-routing/)

下面展示如何根据官方提供的[Bookinfo](https://istio.io/docs/examples/bookinfo/)微服务的多个版本动态地路由请求。在上面部署BookInfo应用之后，该应用有3个`reviews`服务，分别提供：无排名，有黑星排名，有红星排名三种显示。由于默认情况下istio会使用轮询模式将请求一次分发到3个`reviews`服务上，因此在刷新`/productpage`的页面时，可以看到如下变化：

- V1版本：

  ![](./images/Request Routing1.png)

- V2版本：

  ![](./images/Request Routing2.png)

- V3版本：

  ![](./images/Request Routing3.png)

本次展示如何将请求仅分发到某一个`reviews`服务上。

首先创建如下virtual service：

```shell
$ kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

查看路由信息

```shell
$ kubectl get virtualservices -o yaml
```

```yaml
- apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  metadata:
    annotations:
      ...
    name: details
    namespace: default
  spec:
    hosts:
    - details
    http:
    - route:
      - destination:
          host: details
          subset: v1
		  
- apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  metadata:
    annotations:
      ...
    name: productpage
    namespace: default
  spec:
    hosts:
    - productpage
    http:
    - route:
      - destination:
          host: productpage
          subset: v1
		  
- apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  metadata:
    annotations:
      ...
    name: ratings
    namespace: default
  spec:
    hosts:
    - ratings
    http:
    - route:
      - destination:
          host: ratings
          subset: v1
		  
- apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  metadata:
    annotations:
      ...
    name: reviews
    namespace: default
  spec:
    hosts:
    - reviews
    http:
    - route:
      - destination: #可以看到流量都分发到`reviews`服务的v1版本上
          host: reviews #kubernetes的服务，解析为reviews.default.svc.cluster.local
          subset: v1 #将v1修改为v2就可以将请求分只发到v2版本上
```

此时再刷新`/productpage`的页面时，发现只显示无排名的页面

卸载：

```shell
$ kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

#### 基于用户ID的路由

下面展示基于HTTP首部字段的路由，首先在`/productpage`页面中使用名为`jason`的用户登陆(密码随便写)。

部署启用基于用户的路由：

```shell
$ kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

创建的`VirtualService`如下

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  annotations:
    ...
  name: reviews
  namespace: default
spec:
  hosts:
  - reviews
  http:
  - match: #将HTTP请求首部中有end-user:jason字段的请求路由到v2
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route: #HTTP请求首部中不带end-user:jason字段的请求会被路由到v1
    - destination:
        host: reviews
        subset: v1
```

刷新`/productpage`页面，可以看到只会显示v2版本(带黑星排名)页面，退出`jason`登陆，可以看到只显示v1版本(不带排名)页面。

卸载：

```shell
$ kubectl delete -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

## [故障注入](https://istio.io/docs/tasks/traffic-management/fault-injection/)

本节使用故障注入来测试应用的可靠性。

首先使用如下配置固定请求路径：

```shell
$ kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
$ kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

执行后，请求路径变为：

- `productpage` → `reviews:v2` → `ratings` (仅适用于用户 `jason`)
- `productpage` → `reviews:v1` (适用于除`jason`外的其他用户)

### 注入HTTP延时故障

为了测试Bookinfo应用的弹性，为用户`jason`在`reviews:v2` 和`ratings` 的微服务间注入7s的延时，用来模拟Bookinfo的内部bug。

注意`reviews:v2`在调用`ratings`服务时，有一个10s的硬编码超时时间，因此即使引入了7s的延时，端到端流程上也不会看到任何错误。

注入故障，来延缓来自测试用户jason的流量：

```shell
$ kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

查看部署的virtual service信息：

```shell
$ kubectl get virtualservice ratings -o yaml
```

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  annotations:
    ...
  name: ratings
  namespace: default
spec:
  hosts:
  - ratings
  http:
  - fault: #将来自jason的全部流量注入5s的延迟，流量目的地为v1版本的ratings服务
      delay:
        fixedDelay: 7s
        percentage:
          value: 100
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route: #非来自jason的流量不受影响
    - destination:
        host: ratings
        subset: v1
```

打开 `/productpage` 页面，使用jason用户登陆并刷新浏览器页面，可以看到7s内不会加载页面，且页面上可以看到如下错误信息：

![](./images/Fault Injection1.png)

*相同服务的virtualservice的配置会被覆盖，因此此处没必要清理*

### 注入HTTP中断故障

在`ratings`微服务上模拟为测试用户`jason`引入HTTP中断故障，这种场景下，在加载页面时会看到错误信息`Ratings service is currently unavailable`.

使用如下命令为用户`jason`注入HTTP中断

```shell
$ kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml
```

获取部署的`ratings`的virtual service信息

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  annotations:
    ...
  name: ratings
  namespace: default
spec:
  hosts:
  - ratings
  http:
  - fault: #对来自用户jason的请求直接响应500错误码
      abort:
        httpStatus: 500
        percentage:
          value: 100
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

打开 `/productpage` 页面，使用`jason`用户登陆，可以看到如下错误。退出用户`jason`后该错误消失。

![](./images/Fault Injection2.png)

删除注入的中断故障

```shell
$ kubectl delete -f samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml
```

### 卸载

环境清理

```shell
$ kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

## [流量迁移](https://istio.io/docs/tasks/traffic-management/traffic-shifting/)

本章展示如何将流量从一个版本的微服务上迁移到另一个版本的微服务，如将流量从老版本切换到新版本。通常情况下会逐步进行流量切换，istio下可以基于百分比进行流量切换。注意各个版本的权重之和必须等于100，否则会报`total destination weight ${weight-total}= 100`的错误，${weight-total}为当前配置的权重之和。

### 基于权重的路由

- 首先将所有微服务的流量都分发到v1版本的微服务，打开`/productpage`页面可以看到该页面上没有任何排名信息。

  ```shell
  $ kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
  ```

- 使用如下命令将50%的流量从reviews:v1迁移到review:v3

  ```shell
  $ kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
  ```

- 获取virtual service信息

  ```shell
  $ kubectl get virtualservice reviews -o yaml
  ```

  ```yaml
  apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  metadata:
    annotations:
      ...
    name: reviews
    namespace: default
  spec:
    hosts:
    - reviews
    http:
    - route: #50%的流量到v1，50%的流量到v3。
      - destination:
          host: reviews
          subset: v1
        weight: 50
      - destination:
          host: reviews
          subset: v3
        weight: 50
  ```

- 登陆并刷新`/productpage`，可以看到50%概率会看到v1的页面，50%的概率会看到v2的页面

### 卸载

```shell
$ kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

## [TCP流量迁移](https://istio.io/docs/tasks/traffic-management/tcp-traffic-shifting/)

本节展示如何将TCP流量从一个版本的迁移到另一个版本。例如将TCP的流量从老版本迁移到新版本。

### 基于权重的TCP路由

单独创建一个命名空间部署`tcp-echo`应用

```shell
$ kubectl create namespace istio-io-tcp-traffic-shifting
```

openshift下面需要授权1337的用户进行sidecar注入

```shell
$ oc adm policy add-scc-to-group privileged system:serviceaccounts:istio-io-tcp-traffic-shifting
$ oc adm policy add-scc-to-group anyuid system:serviceaccounts:istio-io-tcp-traffic-shifting
```

创建`NetworkAttachmentDefinition`，使用istio-cni

```shell
$ cat <<EOF | oc -n istio-io-tcp-traffic-shifting create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: istio-cni
EOF
```

对命名空间`istio-io-tcp-traffic-shifting`使用自动注入sidecar的方式

```shell
$ kubectl label namespace istio-io-tcp-traffic-shifting istio-injection=enabled
```

部署`tcp-echo`应用

```shell
$ kubectl apply -f samples/tcp-echo/tcp-echo-services.yaml -n istio-io-tcp-traffic-shifting
```

将`tcp-echo`服务的流量全部分发到v1版本

```shell
$ kubectl apply -f samples/tcp-echo/tcp-echo-all-v1.yaml -n istio-io-tcp-traffic-shifting
```

`tcp-echo`服务的pod如下，包含`v1`和`v2`两个版本

```shell
$ oc get pod
NAME                           READY   STATUS    RESTARTS   AGE
tcp-echo-v1-5cb688897c-hk277   2/2     Running   0          16m
tcp-echo-v2-64b7c58f68-hk9sr   2/2     Running   0          16m
```

默认部署的gateway如下，可以看到它使用了istio默认安装的ingress gateway，通过端口`31400`进行访问

```yaml
$ oc get gateways.networking.istio.io tcp-echo-gateway -oyaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  annotations:
    ...
  name: tcp-echo-gateway
  namespace: istio-io-tcp-traffic-shifting
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: tcp
      number: 31400
      protocol: TCP
```

对应绑定的virtual service为`tcp-echo`。此处host为"*"，表示只要访问到gateway  `tcp-echo-gateway` `31400`端口上的流量都会被分发到该virtual service中。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: tcp-echo
spec:
  hosts:
  - "*"
  gateways:
  - tcp-echo-gateway
  tcp:
  - match:
    - port: 31400
    route:
    - destination: #转发到的后端服务的信息
        host: tcp-echo
        port:
          number: 9000
        subset: v1
```

由于没有安装ingress gateway(没有生效)，按照gateway的原理，可以通过istio默认安装的ingress gateway模拟ingress的访问方式。可以看到默认的ingress gateway pod中打开了`31400`端口：

```shell
$ oc exec -it  istio-ingressgateway-64f6f9d5c6-qrnw2 /bin/sh -n istio-system
$ ss -ntl                                                          
State          Recv-Q          Send-Q      Local Address:Port       Peer Address:Port     
LISTEN         0               0                 0.0.0.0:15090           0.0.0.0:*       
LISTEN         0               0               127.0.0.1:15000           0.0.0.0:*       
LISTEN         0               0                 0.0.0.0:31400           0.0.0.0:*       
LISTEN         0               0                 0.0.0.0:80              0.0.0.0:*       
LISTEN         0               0                       *:15020                 *:* 
```

通过ingress gateway pod的kubernetes service进行访问：

```shell
$ oc get svc |grep ingress
istio-ingressgateway   LoadBalancer   10.84.93.45  ...
```

```shell
$ for i in {1..10}; do (date; sleep 1) | nc 10.84.93.45 31400; done
one Wed May 13 11:17:44 UTC 2020
one Wed May 13 11:17:45 UTC 2020
one Wed May 13 11:17:46 UTC 2020
one Wed May 13 11:17:47 UTC 2020
```

可以看到所有的流量都分发到了v1版本(打印"one")的`tcp-echo`服务

> 直接使用tcp-echo对应的kubernetes service进行访问是不受istio管控的，需要通过virtual service进行访问

下面将20%的流量从`tcp-echo:v1` 迁移到`tcp-echo:v2`

```shell
$ kubectl apply -f samples/tcp-echo/tcp-echo-20-v2.yaml -n istio-io-tcp-traffic-shifting
```

查看部署的路由规则

```yaml
$ kubectl get virtualservice tcp-echo -o yaml -n istio-io-tcp-traffic-shifting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  annotations:
    ...
  name: tcp-echo
  namespace: istio-io-tcp-traffic-shifting
spec:
  gateways:
  - tcp-echo-gateway
  hosts:
  - '*'
  tcp:
  - match:
    - port: 31400
    route:
    - destination:
        host: tcp-echo
        port:
          number: 9000
        subset: v1
      weight: 80
    - destination:
        host: tcp-echo
        port:
          number: 9000
        subset: v2
      weight: 20
```

再次进行测试，结果如下：

```shell
$ for i in {1..10}; do (date; sleep 1) | nc 10.84.93.45 31400; done
one Wed May 13 13:17:44 UTC 2020
two Wed May 13 13:17:45 UTC 2020
one Wed May 13 13:17:46 UTC 2020
one Wed May 13 13:17:47 UTC 2020
one Wed May 13 13:17:48 UTC 2020
one Wed May 13 13:17:49 UTC 2020
one Wed May 13 13:17:50 UTC 2020
one Wed May 13 13:17:51 UTC 2020
one Wed May 13 13:17:52 UTC 2020
two Wed May 13 13:17:53 UTC 2020
```

### 卸载

执行如下命令卸载`tcp-echo`应用

```shell
$ kubectl delete -f samples/tcp-echo/tcp-echo-all-v1.yaml -n istio-io-tcp-traffic-shifting
$ kubectl delete -f samples/tcp-echo/tcp-echo-services.yaml -n istio-io-tcp-traffic-shifting
$ kubectl delete namespace istio-io-tcp-traffic-shifting
```

## [请求超时](https://istio.io/docs/tasks/traffic-management/request-timeouts/)

本节介绍如何使用istio在Envoy上配置请求超时时间。用到了官方的例子Bookinfo

部署路由

```shell
$ kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

HTTP请求的超时时间在路由规则的`timeout`字段中指定。默认情况下禁用HTTP的超时，下面会将`review`服务的超时时间设置为1s，为了校验效果，将`ratings` 服务延时2s。

- 将请求路由到v2版本的`review`服务，即调用`ratings`服务的版本，此时`review`服务没有设置超时

  ```yaml
  $ kubectl apply -f - <<EOF
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: reviews
  spec:
    hosts:
      - reviews
    http:
    - route:
      - destination:
          host: reviews
          subset: v2
  EOF
  ```

- 为`rating`服务增加2s延时

  ```yaml
  $ kubectl apply -f - <<EOF
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
          percent: 100
          fixedDelay: 2s
      route:
      - destination:
          host: ratings
          subset: v1
  EOF
  ```

- 打开`/productpage`页面，可以看到Bookinfo应用正在，但刷新页面后会有2s的延时

- 为review服务设置0.5s的请求超时

  ```yaml
  $ kubectl apply -f - <<EOF
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: reviews
  spec:
    hosts:
    - reviews
    http:
    - route:
      - destination:
          host: reviews
          subset: v2
      timeout: 0.5s
  EOF
  ```

- 此时刷新页面，大概1s返回结果，`reviews`不可用

  > 响应花了1s，而不是0.5s的原因是`productpage` 服务硬编码了一次重试，因此`reviews` 服务在返回前会超时2次。Bookinfo应用是有自己内部的超时机制的，具体参见[fault-injection](https://istio.io/docs/tasks/traffic-management/fault-injection/)

### 卸载

```shell
$ kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

## 熔断

本节将显示如何为连接、请求和异常值检测配置熔断。熔断是创建弹性微服务应用程序的重要模式，允许编写的程序能够限制错误，延迟峰值以及不期望的网络的影响

在`default`命名空间(已经开启自动注入sidecar)下部署`httpbin` 