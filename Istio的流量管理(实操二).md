# Istio的流量管理(实操二)

涵盖官方文档[Traffic Management](https://istio.io/latest/docs/tasks/traffic-management/ingress/)章节中的inrgess部分。

[TOC]

## [Ingress网关](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/)

在kubernetes环境中，kubernetes ingress资源用于指定暴露到集群外的服务。在istio服务网格中，使用了一种不同的配置模型，称为[istio网关](https://istio.io/docs/reference/config/networking/gateway/)。一个网关允许将istio的特性，如镜像和路由规则应用到进入集群的流量上。

本节描述了如何使用istio网关将一个服务暴露到服务网格外。

### 环境准备

使用如下命令创建httpbin服务

```shell
$ kubectl apply -f samples/httpbin/httpbin.yaml
```

### 确定ingress的IP和端口

由于本环境中没有配置对外的负载均衡，因此此处的`EXTERNAL-IP`为空，使用`node port`进行访问

```shell
# kubectl get svc istio-ingressgateway -n istio-system
NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP  PORT(S)    AGE
istio-ingressgateway   LoadBalancer   10.84.93.45   <pending>     ...        11d
```

获取`ingressgateway` service的`http2`和`https`对应的端口

```shell
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
$ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
$ export TCP_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].nodePort}')
```

下面是istio-system命名空间的`istio-ingressgateway ` service中的一部分端口信息，可以看到`http2`和`https`的nodeport分别为`31194`和`31785`，对应上面的`INGRESS_PORT`和`SECURE_INGRESS_PORT`

```json
{
    "name": "http2",
    "nodePort": 31194,
    "port": 80,
    "protocol": "TCP",
    "targetPort": 80
},
{
    "name": "https",
    "nodePort": 31785,
    "port": 443,
    "protocol": "TCP",
    "targetPort": 443
},
```

获取istio-system命名空间中`ingressgateway` pod 的hostIP

```shell
$ export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
```

### 使用istio网关配置ingress

一个ingress[网关](https://istio.io/docs/reference/config/networking/gateway/)描述了在网格边缘用于接收入站HTTP/TCP连接的负载均衡，配置了暴露的端口，协议等。[kubernetes ingress资源](https://kubernetes.io/docs/concepts/services-networking/ingress/)不包括任何流量路由配置。ingress 流量的路由使用istio路由规则，与内部服务请求相同：

1. 创建istio `Gateway`，网关监听地址为`httpbin.example.com`，端口为`80`(位于默认的`ingressgateway`pod中)

   ```yaml
   kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: httpbin-gateway
   spec:
     selector:
       istio: ingressgateway # use Istio default gateway implementation
     servers:
     - port:
         number: 80 #gateway暴露80端口
         name: http
         protocol: HTTP
       hosts: #gateway暴露的主机名
       - "httpbin.example.com"
   EOF
   ```

2. 通过`Gateway`配置进入的流量路由，将URI为`httpbin.example.com`，且目的地为`/status`或`/delay`的请求分发到`httpbin`服务的`8000`端口，其他请求会返回404响应。

   ```yaml
   kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: httpbin
   spec:
     hosts: #virtual service的hosts字段与Gateway的servers.hosts字段需要匹配
     - "httpbin.example.com" 
     gateways:
     - httpbin-gateway
     http:
     - match:
       - uri:
           prefix: /status
       - uri:
           prefix: /delay
       route:
       - destination:
           port:
             number: 8000
           host: httpbin
   EOF
   ```

   可以看到流量被导入了`httpbin` service暴露的8000端口上

   ```shell
   $ oc get svc |grep httpbin
   httpbin       ClusterIP      10.84.222.69   <none>    8000/TCP   19h
   ```

   > 来自网格内部其他服务的请求则不受此规则的约束，会使用默认的轮询路由进行请求分发。为了限制内部的调用规则，可以将特定的值`mesh`添加到`gateways`列表中。由于内部服务主机名(如`httpbin.default.svc.cluster.local`) 可能与外部不同，因此需要将主机名添加到`hosts`列表中。

3. 使用curl命令访问`httpbin`服务，此时通过`-H`选项修改了HTTP请求首部的`Host`字段，使用http2的nodeport方式访问：

   ```shell
   $ curl -s -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/status/200
   HTTP/1.1 200 OK
   server: istio-envoy
   date: Thu, 21 May 2020 03:22:50 GMT
   content-type: text/html; charset=utf-8
   access-control-allow-origin: *
   access-control-allow-credentials: true
   content-length: 0
   x-envoy-upstream-service-time: 21
   ```

4. 访问其他URL路径则返回404错误

   ```shell
   $ curl -s -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/headers
   HTTP/1.1 404 Not Found
   date: Thu, 21 May 2020 03:25:20 GMT
   server: istio-envoy
   transfer-encoding: chunked
   ```

### 使用浏览器访问ingress服务

由于无法像使用curl一样修改请求的`Host`首部字段，因此无法使用浏览器访问`httpbin`服务。为了解决这个问题，可以在`Gateway`和`VirtualService`中的host字段使用通配符`*`。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"  #指定通配符，监听所有流量，不限制外部流量的地址
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /headers
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

### 问题定位

1. 检查环境变量`INGRESS_HOST`和`INGRESS_PORT`，保证这两个值是有效的

   ```shell
   $ kubectl get svc -n istio-system
   $ echo INGRESS_HOST=$INGRESS_HOST, INGRESS_PORT=$INGRESS_PORT
   ```

2. 校验相同的端口上没有其他istio ingress网格

   ```shell
   $ kubectl get gateway --all-namespaces
   ```

3. 校验没有在相同的IP和端口上定义kubernetes ingress资源

   ```shell
   $ kubectl get ingress --all-namespaces
   ```

4. 如果没有负载均衡，可以参照上面步骤使用node port方式

### 卸载

```shell
$ kubectl delete gateway httpbin-gateway
$ kubectl delete virtualservice httpbin
$ kubectl delete --ignore-not-found=true -f samples/httpbin/httpbin.yaml
```

## [Ingress(kubernetes)](https://istio.io/docs/tasks/traffic-management/ingress/kubernetes-ingress/)

执行[ingress流量控制](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/)中的[Before you begin](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control#before-you-begin)和[Determining the ingress IP and ports](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)小节的操作，部署`httpbin`服务。本节介绍如何通过kubernetes的`Ingress`(非istio的gateway)进行访问。

下面展示如何配置一个80端口的`Ingress`，用于HTTP流量：

1. 创建一个istio `Gateway`，将来自`httpbin.example.com:80/status/*`的流量分发到service `httpbin`的`8000`端口

   ```shell
   $ kubectl apply -f - <<EOF
   apiVersion: networking.k8s.io/v1beta1
   kind: Ingress
   metadata:
     annotations:
       kubernetes.io/ingress.class: istio
     name: ingress
   spec:
     rules:
     - host: httpbin.example.com
       http:
         paths:
         - path: /status/*
           backend:
             serviceName: httpbin
             servicePort: 8000
   EOF
   ```

   注意需要使用 `kubernetes.io/ingress.class` annotation来告诉istio网关控制器处理该`ingress`，否则会忽略该ingress。

2. 使用curl命令访问httpbin服务。Ingress的流量也需要经过istio `ingressgateway`。

   ```shell
   $ curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/status/200
   HTTP/1.1 200 OK
   server: istio-envoy
   date: Fri, 22 May 2020 06:12:56 GMT
   content-type: text/html; charset=utf-8
   access-control-allow-origin: *
   access-control-allow-credentials: true
   content-length: 0
   x-envoy-upstream-service-time: 20
   ```

   httpbin的服务发现是通过EDS实现的，使用如下命令查看：

   ```shell
   $ istioctl proxy-config cluster istio-ingressgateway-569669bb67-b6p5r|grep 8000
   httpbin.default.svc.cluster.local                   8000    -    outbound      EDS
   outbound_.8000_._.httpbin.default.svc.cluster.local   -     -       -          EDS
   ```

3. 访问其他未暴露的路径，返回HTTP 404错误：

   ```shell
   $ curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/headers
   HTTP/1.1 404 Not Found
   date: Fri, 22 May 2020 06:24:30 GMT
   server: istio-envoy
   transfer-encoding: chunked
   ```

### 下一步

#### TLS

Ingress支持[TLS设置](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)。Istio也支持TLS设置，但相关的`secret`必须存在`istio-ingressgateway` deployment所在的命名空间中。可以使用[cert-manager](https://istio.io/docs/ops/integrations/certmanager/)生成这些证书。

#### 指定路径类型

默认情况下，Istio会将路径视为完全匹配，如果路径使用`/*`或`*`结尾，则该路径为前缀匹配。不支持其他正则匹配。

kubernetes 1.18中新增了一个字段`pathType`，允许声明为`Exact`或`Prefix`。

#### 指定`IngressClass`

kubernetes 1.18中新增了一个资源`IngressClass`，替换了`Ingress`资源的 `kubernetes.io/ingress.class` annotation。如果使用该资源，则需要将`controller`设置为`istio.io/ingress-controller`：

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: IngressClass
metadata:
  name: istio
spec:
  controller: istio.io/ingress-controller
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress
spec:
  ingressClassName: istio
  ...
```

### 卸载

```shell
$ kubectl delete ingress ingress
$ kubectl delete --ignore-not-found=true -f samples/httpbin/httpbin.yaml
```

## [安全网关](https://istio.io/docs/tasks/traffic-management/ingress/secure-ingress-mount/)

本节讲述如何使用simple或mutual TLS暴露安全的HTTPS服务。证书是通过SDS进行密钥发现的。

TLS需要的私钥，服务端证书，根证书是使用基于文件装载的方法配置的。

执行[ingress流量控制](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/)中的[Before you begin](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control#before-you-begin)和[Determining the ingress IP and ports](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)小节的操作，部署`httpbin`服务，并获取 `INGRESS_HOST` 和`SECURE_INGRESS_PORT`变量。

### 生成服务端证书和私钥

下面使用[openssl](https://man.openbsd.org/openssl.1)生成需要的证书和密钥

1. 生成一个根证书和一个私钥，用于签名服务的证书

   ```shell
   $ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
   ```

2. 为 `httpbin.example.com`生成一个证书和私钥

   ```shell
   $ openssl req -out httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout httpbin.example.com.key -subj "/CN=httpbin.example.com/O=httpbin organization"
   $ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in httpbin.example.com.csr -out httpbin.example.com.crt
   ```

### 单主机配置TLS ingress网关

1. 为ingree网关创建一个secret

   > secret的名字不能以`istio`或`prometheus`开头，且secret不应该包含`token`字段

   ```shell
   $ kubectl create -n istio-system secret tls httpbin-credential --key=httpbin.example.com.key --cert=httpbin.example.com.crt
   ```

2. 在`server`部分定义一个443端口的`Gateway`，将`credentialName`指定为`httpbin-credential`，与`secret`的名字相同，TLS的mode为`SIMPLE`。

   ```yaml
   $ cat <<EOF | kubectl apply -f -
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: mygateway
   spec:
     selector:
       istio: ingressgateway # use istio default ingress gateway
     servers:
     - port:
         number: 443
         name: https
         protocol: HTTPS
       tls: #对暴露的服务使用SIMPLE模式的tls，即单向tls验证
         mode: SIMPLE
         credentialName: httpbin-credential # must be the same as secret
       hosts:
       - httpbin.example.com
   EOF
   ```

3. 配置进入`Gateway`的流量路由。与上一节中的`VirtualService`相同

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: httpbin
   spec:
     hosts:
     - "httpbin.example.com"
     gateways:
     - httpbin-gateway
     http:
     - match:
       - uri:
           prefix: /status
       - uri:
           prefix: /delay
       route:
       - destination:
           port:
             number: 8000  #可以看到tls只是这是在gateway上的，当进入网格之后就不需要了
           host: httpbin
   EOF
   ```

4. 使用curl向`SECURE_INGRESS_PORT`发送`HTTPS`请求访问`httpbin`服务，请求中携带了公钥`example.com.crt` 。`--resolve`标记可以在使用curl访问TLS的网关IP时，在[SNI](https://en.wikipedia.org/wiki/Server_Name_Indication)中支持`httpbin.example.com`。`--cacert`选择支持使用生成的证书校验服务。

   > `-HHost:httpbin.example.com` 选项仅在`SECURE_INGRESS_PORT`不同于实际的网关端口(443)时才会需要，例如，通过映射的`NodePort`方式访问服务时。

   通过将请求发送到`/status/418` URL路径时，可以看到`httpbin`确实被访问了，httpbin服务会返回[418 I’m a Teapot](https://tools.ietf.org/html/rfc7168#section-2.3.3)代码。.

   ```shell
   $ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
   --cacert example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
   
   > --cacert example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
   * Added httpbin.example.com:31967:172.20.127.211 to DNS cache
   * About to connect() to httpbin.example.com port 31967 (#0)
   *   Trying 172.20.127.211...
   * Connected to httpbin.example.com (172.20.127.211) port 31967 (#0)
   * Initializing NSS with certpath: sql:/etc/pki/nssdb
   *   CAfile: example.com.crt
     CApath: none
   * SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
   * Server certificate:
   *       subject: O=httpbin organization,CN=httpbin.example.com
   *       start date: May 22 09:03:01 2020 GMT
   *       expire date: May 22 09:03:01 2021 GMT
   *       common name: httpbin.example.com
   *       issuer: CN=example.com,O=example Inc.
   > GET /status/418 HTTP/1.1
   > User-Agent: curl/7.29.0
   > Accept: */*
   > Host:httpbin.example.com
   >
   < HTTP/1.1 418 Unknown
   < server: istio-envoy
   < date: Fri, 22 May 2020 09:08:29 GMT
   < x-more-info: http://tools.ietf.org/html/rfc2324
   < access-control-allow-origin: *
   < access-control-allow-credentials: true
   < content-length: 135
   < x-envoy-upstream-service-time: 2
   <
   
       -=[ teapot ]=-
   
          _...._
        .'  _ _ `.
       | ."` ^ `". _,
       \_;`"---"`|//
         |       ;/
         \_     _/
           `"""`
   * Connection #0 to host httpbin.example.com left intact
   ```

   查看curl输出中的*Server certificate*中的信息，上述返回值的最后有一个茶壶的图片，说明运行成功。

5. 删除老的网关secret，创建一个新的secret，并使用该secret修改ingress网关的凭据

   ```shell
   $ kubectl -n istio-system delete secret httpbin-credential
   ```

   ```shell
   $ mkdir new_certificates
   $ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout new_certificates/example.com.key -out new_certificates/example.com.crt
   $ openssl req -out new_certificates/httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout new_certificates/httpbin.example.com.key -subj "/CN=httpbin.example.com/O=httpbin organization"
   $ openssl x509 -req -days 365 -CA new_certificates/example.com.crt -CAkey new_certificates/example.com.key -set_serial 0 -in new_certificates/httpbin.example.com.csr -out new_certificates/httpbin.example.com.crt
   $ kubectl create -n istio-system secret tls httpbin-credential \
   --key=new_certificates/httpbin.example.com.key \
   --cert=new_certificates/httpbin.example.com.crt
   ```

6. 使用新的证书链访问`httpbin`服务

   ```shell
   $ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
   --cacert new_certificates/example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
   ```

7. 如果使用老的证书访问，则返回错误

   ```shell
   $ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
   > --cacert example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
   * Added httpbin.example.com:31967:172.20.127.211 to DNS cache
   * About to connect() to httpbin.example.com port 31967 (#0)
   *   Trying 172.20.127.211...
   * Connected to httpbin.example.com (172.20.127.211) port 31967 (#0)
   * Initializing NSS with certpath: sql:/etc/pki/nssdb
   *   CAfile: example.com.crt
     CApath: none
   * Server certificate:
   *       subject: O=httpbin organization,CN=httpbin.example.com
   *       start date: May 22 09:24:07 2020 GMT
   *       expire date: May 22 09:24:07 2021 GMT
   *       common name: httpbin.example.com
   *       issuer: CN=example.com,O=example Inc.
   * NSS error -8182 (SEC_ERROR_BAD_SIGNATURE)
   * Peer's certificate has an invalid signature.
   * Closing connection 0
   curl: (60) Peer's certificate has an invalid signature.
   ```

### 多主机配置TLS ingress网关

本节会为多个主机(`httpbin.example.com`和`helloworld-v1.example.com`)配置一个ingress网关。ingress网关会在`credentialName`中查找唯一的凭据。

1. 删除之前创建的secret并为`httpbin`重建凭据

   ```shell
   $ kubectl -n istio-system delete secret httpbin-credential
   $ kubectl create -n istio-system secret tls httpbin-credential \
   --key=httpbin.example.com.key \
   --cert=httpbin.example.com.crt
   ```

2. 启动`helloworld-v1`

   ```yaml
   $ cat <<EOF | kubectl apply -f -
   apiVersion: v1
   kind: Service
   metadata:
     name: helloworld-v1
     labels:
       app: helloworld-v1
   spec:
     ports:
     - name: http
       port: 5000
     selector:
       app: helloworld-v1
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: helloworld-v1
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: helloworld-v1
         version: v1
     template:
       metadata:
         labels:
           app: helloworld-v1
           version: v1
       spec:
         containers:
         - name: helloworld
           image: istio/examples-helloworld-v1
           resources:
             requests:
               cpu: "100m"
           imagePullPolicy: IfNotPresent #Always
           ports:
           - containerPort: 5000
   EOF
   ```

3. 为`helloworld-v1.example.com`创建证书和私钥

   ```shell
   $ openssl req -out helloworld-v1.example.com.csr -newkey rsa:2048 -nodes -keyout helloworld-v1.example.com.key -subj "/CN=helloworld-v1.example.com/O=helloworld organization"
   $ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 1 -in helloworld-v1.example.com.csr -out helloworld-v1.example.com.crt
   ```

4. 为`helloworld-credential`创建secret

   ```shell
   $ kubectl create -n istio-system secret tls helloworld-credential --key=helloworld-v1.example.com.key --cert=helloworld-v1.example.com.crt
   ```

5. 定义一个网关，网关端口为443。在`credentialName`字段分别设置`httpbin-credential`和`helloworld-credential`，TLS模式为`SIMPLE`。

   ```yaml
   $ cat <<EOF | kubectl apply -f -
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: mygateway
   spec:
     selector:
       istio: ingressgateway # use istio default ingress gateway
     servers:
     - port:
         number: 443
         name: https-httpbin #httpbin的gateway配置
         protocol: HTTPS
       tls:
         mode: SIMPLE
         credentialName: httpbin-credential
       hosts:
       - httpbin.example.com
     - port:
         number: 443
         name: https-helloworld #https-helloword的gateway配置
         protocol: HTTPS
       tls:
         mode: SIMPLE
         credentialName: helloworld-credential
       hosts:
       - helloworld-v1.example.com
   EOF
   ```

6. 配置gateway的流量路由，为新应用添加对应的virtual service

   ```yaml
   $ cat <<EOF | kubectl apply -f -
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: helloworld-v1
   spec:
     hosts:
     - helloworld-v1.example.com
     gateways:
     - mygateway
     http:
     - match:
       - uri:
           exact: /hello
       route:
       - destination:
           host: helloworld-v1
           port:
             number: 5000
   EOF
   ```

7. 向`helloworld-v1.example.com`发送HTTPS请求

   ```shell
   $ curl -v -HHost:helloworld-v1.example.com --resolve "helloworld-v1.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
   --cacert example.com.crt "https://helloworld-v1.example.com:$SECURE_INGRESS_PORT/hello"
   
   ...
   < HTTP/1.1 200 OK
   < content-type: text/html; charset=utf-8
   < content-length: 60
   < server: istio-envoy
   < date: Sat, 23 May 2020 07:38:40 GMT
   < x-envoy-upstream-service-time: 143
   <
   Hello version: v1, instance: helloworld-v1-5dfcf5d5cd-2l44c
   * Connection #0 to host helloworld-v1.example.com left intact
   ```

8. 向 `httpbin.example.com`发送请求

   ```shell
   $ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
   --cacert example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
   
    ...
       -=[ teapot ]=-
   
          _...._
        .'  _ _ `.
       | ."` ^ `". _,
       \_;`"---"`|//
         |       ;/
         \_     _/
           `"""`
   * Connection #0 to host httpbin.example.com left intact
   ```

### 配置一个mutual TLS ingress网关

删除之前的secreting创建一个新的secret，server会使用该CA证书校验client，使用`cacert`保存CA证书。

```shel
$ kubectl -n istio-system delete secret httpbin-credential
$ kubectl create -n istio-system secret generic httpbin-credential --from-file=tls.key=httpbin.example.com.key \
--from-file=tls.crt=httpbin.example.com.crt --from-file=ca.crt=example.com.crt
```

1. 将gateway的TLS模式设置为`MUTUAL`

   ```yaml
   $ cat <<EOF | kubectl apply -f -
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
    name: mygateway
   spec:
    selector:
      istio: ingressgateway # use istio default ingress gateway
    servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: MUTUAL #对网关暴露的服务httpbin.example.com启用双向认证
        credentialName: httpbin-credential # must be the same as secret
      hosts:
      - httpbin.example.com
   EOF
   ```

2. 使用先前的方式发送HTTPS请求，可以看到访问失败

   ```shell
   $ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
   > --cacert example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
   * Added httpbin.example.com:31967:172.20.127.211 to DNS cache
   * About to connect() to httpbin.example.com port 31967 (#0)
   *   Trying 172.20.127.211...
   * Connected to httpbin.example.com (172.20.127.211) port 31967 (#0)
   * Initializing NSS with certpath: sql:/etc/pki/nssdb
   *   CAfile: example.com.crt
     CApath: none
   * NSS: client certificate not found (nickname not specified)
   * NSS error -12227 (SSL_ERROR_HANDSHAKE_FAILURE_ALERT)
   * SSL peer was unable to negotiate an acceptable set of security parameters.
   * Closing connection 0
   curl: (35) NSS: client certificate not found (nickname not specified)
   ```

3. 使用公钥`example.com.crt`生成client的证书和私钥。在`curl`中传入客户端的证书和私钥，使用`--cert`传入客户端证书，使用`--key`传入私钥

   ```shell
   $ openssl req -out client.example.com.csr -newkey rsa:2048 -nodes -keyout client.example.com.key -subj "/CN=client.example.com/O=client organization"
   $ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 1 -in client.example.com.csr -out client.example.com.crt
   ```
   
   ```shell
   $ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" --cacert example.com.crt --cert ./client.example.com.crt --key ./client.example.com.key "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
   ...
       -=[ teapot ]=-
   
          _...._
        .'  _ _ `.
       | ."` ^ `". _,
       \_;`"---"`|//
         |       ;/
         \_     _/
           `"""`
   * Connection #0 to host httpbin.example.com left intact
   ```

istio支持几种不同的Secret格式，来支持与多种工具的集成，如[cert-manager](https://istio.io/docs/ops/integrations/certmanager/):

- 一个TLS Secret使用`tls.key`和`tls.crt`；对于mutual TLS，会用到`ca.crt`
- 一个generic Secret会用到`key`和`cert`；对于mutual TLS，会用到`cacert`
- 一个generic Secret会用到`key`和`cert`；对于mutual TLS，会用到一个单独的名为 `<secret>-cacert`的generic Secret，以及一个`cacert` key。例如`httpbin-credential` 包含`key` 和`cert`，`httpbin-credential-cacert` 包含`cacert`

### 问题定位

- 检查`INGRESS_HOST`和`SECURE_INGRESS_PORT`环境变量

  ```shell
  $ kubectl get svc -n istio-system
  $ echo INGRESS_HOST=$INGRESS_HOST, SECURE_INGRESS_PORT=$SECURE_INGRESS_PORT
  ```

- 检查`istio-ingressgateway`控制器的错误日志

  ```shell
  $ kubectl logs -n istio-system "$(kubectl get pod -l istio=ingressgateway \
  -n istio-system -o jsonpath='{.items[0].metadata.name}')"
  ```

- 校验`istio-system`命名空间中成功创建了secret。上例中应该存在`httpbin-credential`和`helloworld-credential` 

  ```shell
  $ kubectl -n istio-system get secrets
  ```

- 校验ingress网关agent将密钥/证书对上传到了ingress网关

  ```shell
  $ kubectl logs -n istio-system "$(kubectl get pod -l istio=ingressgateway \
  -n istio-system -o jsonpath='{.items[0].metadata.name}')"
  ```

#### 定位mutul TLS问题

- 校验CA加载到了 `istio-ingressgateway` pod中，查看是否存在`example.com.crt`

  ```shell
  $ kubectl exec -it -n istio-system $(kubectl -n istio-system get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}') -- ls -al /etc/istio/ingressgateway-ca-certs
  ```

- 如果创建了`istio-ingressgateway-ca-certs` secret，但没有加载CA证书，删除ingress网关pod，强制加载该证书

  ```shell
  $ kubectl delete pod -n istio-system -l istio=ingressgateway
  ```

- 校验CA证书的`Subject`字段是否正确

  ```shell
  $ kubectl exec -i -n istio-system $(kubectl get pod -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].metadata.name}')  -- cat /etc/istio/ingressgateway-ca-certs/example.com.crt | openssl x509 -text -noout | grep 'Subject:'
          Subject: O=example Inc., CN=example.com
  ```

  log中可以看到添加了`httpbin-credential` secret。如果使用mutual TLS，那么也会出现 `httpbin-credential-cacert` secret。校验log中显示了网关agent从ingress网关接收到了SDS请求，资源的名称为 `httpbin-credential`，且ingress网关获取到了密钥/证书对。如果使用了mutual TLS，日志应该显示将密钥/证书发送到ingress网关，网关agent接收到了带 `httpbin-credential-cacert` 资源名称的SDS请求，并回去到了根证书。

### 卸载

1. 删除`Gateway`配置，`VirtualService`和secret

   ```shell
   $ kubectl delete gateway mygateway
   $ kubectl delete virtualservice httpbin
   $ kubectl delete --ignore-not-found=true -n istio-system secret httpbin-credential \
   helloworld-credential
   $ kubectl delete --ignore-not-found=true virtualservice helloworld-v1
   ```

2. 删除证书目录

   ```shell
   $ rm -rf example.com.crt example.com.key httpbin.example.com.crt httpbin.example.com.key httpbin.example.com.csr helloworld-v1.example.com.crt helloworld-v1.example.com.key helloworld-v1.example.com.csr client.example.com.crt client.example.com.csr client.example.com.key ./new_certificates
   ```

3. 停止`httpbin`和`helloworld-v1` 服务：

   ```shell
   $ kubectl delete deployment --ignore-not-found=true httpbin helloworld-v1
   $ kubectl delete service --ignore-not-found=true httpbin helloworld-v1
   ```

## [不终止TLS的ingress网关](https://istio.io/docs/tasks/traffic-management/ingress/ingress-sni-passthrough/)

上一节中描述了如何配置HTTPS ingree来访问一个HTTP服务。本节中描述如何配置HTTPS ingrss来访问HTTPS服务等。通过配置ingress网关来执行SNI方式的访问，而不会在请求进入ingress时终止TLS。

本例中使用一个NGINX服务器作为HTTPS服务。

### 生成客户端和服务端的证书和密钥

1. 生成一个根证书和私钥，用于签名服务

   ```shell
   $ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
   ```

2. 为 `nginx.example.com`创建证书和私钥

   ```shell
   $ openssl req -out nginx.example.com.csr -newkey rsa:2048 -nodes -keyout nginx.example.com.key -subj "/CN=nginx.example.com/O=some organization"
   $ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in nginx.example.com.csr -out nginx.example.com.crt
   ```

### 部署NGINX服务

1. 创建kubernetes Secret保存服务的证书

   ```shell
   $ kubectl create secret tls nginx-server-certs --key nginx.example.com.key --cert nginx.example.com.crt
   ```

2. 为NGINX服务创建配置文件

   ```shell
   $ cat <<EOF > ./nginx.conf
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
       ssl_certificate /etc/nginx-server-certs/tls.crt;
       ssl_certificate_key /etc/nginx-server-certs/tls.key;
     }
   }
   EOF
   ```

3. 为NGINX服务创建kubernetes configmap

   ```shell
   $ kubectl create configmap nginx-configmap --from-file=nginx.conf=./nginx.conf
   ```

4. 部署NGINX服务

   ```yaml
   $ cat <<EOF | istioctl kube-inject -f - | kubectl apply -f -
   apiVersion: v1
   kind: Service
   metadata:
     name: my-nginx
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
         volumes:
         - name: nginx-config
           configMap:
             name: nginx-configmap
         - name: nginx-server-certs
           secret:
             secretName: nginx-server-certs #保存了NGINX服务的证书和私钥
   EOF
   ```

5. 为了测试NGINX服务部署成功，向服务发送不使用证书的方式请求，并校验打印信息是否正确：

   ```shell
   $ kubectl exec -it $(kubectl get pod  -l run=my-nginx -o jsonpath={.items..metadata.name}) -c istio-proxy -- curl -v -k --resolve 
   ...
   * Server certificate:
   *  subject: CN=nginx.example.com; O=some organization
   *  start date: May 25 02:09:02 2020 GMT
   *  expire date: May 25 02:09:02 2021 GMT
   *  issuer: O=example Inc.; CN=example.com
   *  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
   ...
   ```

### 配置一个ingress gateway

1. 定义一个gateway，端口为`443`.注意TLS的模式为`PASSTHROUGH`，表示gateway会放行ingress流量，不终止TLS

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: mygateway
   spec:
     selector:
       istio: ingressgateway # use istio default ingress gateway
     servers:
     - port:
         number: 443
         name: https
         protocol: HTTPS
       tls:
         mode: PASSTHROUGH #不终止TLS
       hosts:
       - nginx.example.com
   EOF
   ```

2. 配置经过Gateway的流量路由

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: nginx
   spec:
     hosts:
     - nginx.example.com
     gateways:
     - mygateway
     tls:
     - match:
       - port: 443 #将gateway的流量导入kubernetes的my-nginx service
         sniHosts:
         - nginx.example.com
       route:
       - destination:
           host: my-nginx
           port:
             number: 443
   EOF
   ```

3. 根据[指导](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)配置`SECURE_INGRESS_PORT`和`INGRESS_HOST`环境变量

4. 通过ingress访问nginx，可以看到访问成功

   ```shell
   $ curl -v --resolve nginx.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST --cacert example.com.crt https://nginx.example.com:$SECURE_INGRESS_PORT
   ...
   * SSL connection using TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
   * Server certificate:
   *       subject: O=some organization,CN=nginx.example.com
   *       start date: May 25 02:09:02 2020 GMT
   *       expire date: May 25 02:09:02 2021 GMT
   *       common name: nginx.example.com
   *       issuer: CN=example.com,O=example Inc.
   ...
   <title>Welcome to nginx!</title>
   ...
   ```

### 卸载

1. 移除kubernetes资源

   ```shell
   $ kubectl delete secret nginx-server-certs
   $ kubectl delete configmap nginx-configmap
   $ kubectl delete service my-nginx
   $ kubectl delete deployment my-nginx
   $ kubectl delete gateway mygateway
   $ kubectl delete virtualservice nginx
   ```

2. 删除证书和密钥

   ```shell
   $ rm example.com.crt example.com.key nginx.example.com.crt nginx.example.com.key nginx.example.com.csr
   ```

3. 删除生成的配置文件

   ```shell
   $ rm ./nginx.conf
   ```

### TIPS

Gateway支持的[TLS模式](https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSmode)如下：

| Name             | Description                                                  |
| ---------------- | ------------------------------------------------------------ |
| PASSTHROUGH      | 客户端提供的SNI字符串将用作VirtualService TLS路由中的匹配条件，以根据服务注册表确定目标服务 |
| SIMPLE           | 使用标准TLS语义的安全连接                                    |
| MUTUAL           | 通过提供服务器证书进行身份验证，使用双边TLS来保护与下游的连接 |
| AUTO_PASSTHROUGH | 与直通模式相似，不同之处在于具有此TLS模式的服务器不需要关联的VirtualService即可从SNI值映射到注册表中的服务。目标详细信息（例如服务/子集/端口）被编码在SNI值中。代理将转发到SNI值指定的上游（Envoy）群集（一组端点）。 |
| ISTIO_MUTUAL     | 通过提供用于身份验证的服务器证书，使用相互TLS使用来自下游的安全连接 |

