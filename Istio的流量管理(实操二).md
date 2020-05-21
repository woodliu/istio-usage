# Istio的流量管理(实操二)(istio 系列四)

使用官方的[Bookinfo](https://istio.io/docs/examples/bookinfo/)应用进行测试。涵盖官方文档[Traffic Management](https://istio.io/docs/tasks/traffic-management/)章节中的inrgess和egress部分。

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

1. 创建istio `Gateway`，将来自`httpbin.example.com`的流量导入网格的`80`端口(即默认的`ingressgateway`pod)

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
         number: 80
         name: http
         protocol: HTTP
       hosts:
       - "httpbin.example.com"
   EOF
   ```

2. 通过`Gateway`配置进入的流量路由，将来自`httpbin.example.com`，且目的地为`/status`或`/delay`的请求分发到`httpbin`服务的`8000`端口，其他请求会返回404响应。

   ```yaml
   kubectl apply -f - <<EOF
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
             number: 8000
           host: httpbin
   EOF
   ```

   可以看到流量被导入了`httpbin` service暴露的8000端口上

   ```shell
   $ oc get svc |grep httpbin
   httpbin       ClusterIP      10.84.222.69   <none>    8000/TCP   19h
   ```

   > 来自网格内部其他服务的请求则不受此规则的约束，会使用默认的轮询路由进行请求分发。为了限制内部的调用规则，可以将特定的值`mesh`添加到`gateways`列表中。由于内部服务的主机名可能与外部不同，因此需要将主机名添加到`hosts`列表中。

3. 使用curl命令访问`httpbin`服务，此时通过`-H`选项修改了HTTP请求首部的`Host`字段，使用http2的nodeport方式访问：

   ```shell
   $ curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/status/200
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
   $ curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/headers
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
    - "*"  #指定通配符，不限制外部流量的地址
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

3. **校验没有在相同的IP和端口上定义kubernetes ingress资源**

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

## [安全网关(文件挂载)](https://istio.io/docs/tasks/traffic-management/ingress/secure-ingress-mount/)

本节讲述如何使用simple或mutual TLS暴露安全的HTTPS服务。

TLS需要的私钥，服务端证书，根证书是使用基于文件装载的方法配置的。

执行[ingress流量控制](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/)中的[Before you begin](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control#before-you-begin) 的[Before you begin](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control#before-you-begin) 和[Determining the ingress IP and ports](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)小节的操作，部署`httpbin`服务，并获取 `INGRESS_HOST` 和`SECURE_INGRESS_PORT`变量。

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

### 使用挂载文件的方式配置TLS ingress网关

本节中在ingress网关上使用443端口处理HTTP流量。首先创建一个带证书和私钥的secret，该secret挂载到文件`/etc/istio/ingressgateway-certs`中，然后创建一个使用443端口的网关服务。

1. 创建一个kubernetes secret来保存服务的证书和私钥。使用`kubectl`在命名空间 `istio-system` 中创建`secret`  `istio-ingressgateway-certs` 。istio网关会自动加载该secret。

   > secret在命名空间 `istio-system` 中的名字必须是 `istio-ingressgateway-certs`，与该任务中使用的Istio默认入口网关的配置保持一致。

   ```shell
   $ kubectl create -n istio-system secret tls istio-ingressgateway-certs --key httpbin.example.com.key --cert httpbin.example.com.crt
   secret/istio-ingressgateway-certs created
   ```

   注意，默认`istio-system`命名空间中的所有pod都可以挂载该secret并访问私钥。可以在不同的命名空间中部署ingress网关并创建secret，这种情况下，只有该ingress网关的pod才能挂载该secert。

   校验ingress网关已经挂载了`tls.crt`和`tls.key`，如果没有立即看到，可能需要等一会：

   ```shell
   $ kubectl exec -it -n istio-system $(kubectl -n istio-system get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}') -- ls -al /etc/istio/ingressgateway-certs
   ```

2. 在`server`部分定义一个443端口的`Gateway`

   > 证书的私钥的位置必须是 `/etc/istio/ingressgateway-certs`,否则gateway将无法加载。证书最终会挂载到`istio-ingressgateway-xxx` pod的`/etc/istio/ingressgateway-certs`目录中

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: httpbin-gateway
   spec:
     selector:
       istio: ingressgateway # use istio default ingress gateway
     servers:
     - port:
         number: 443
         name: https
         protocol: HTTPS
       tls:  # 只是新增了tls部分
         mode: SIMPLE
         serverCertificate: /etc/istio/ingressgateway-certs/tls.crt 
         privateKey: /etc/istio/ingressgateway-certs/tls.key
       hosts:
       - "httpbin.example.com"
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
   $ curl -v -HHost:httpbin.example.com --resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST --cacert example.com.crt https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418
   
   * Added httpbin.example.com:31785:172.20.127.212 to DNS cache
   * About to connect() to httpbin.example.com port 31785 (#0)
   *   Trying 172.20.127.212...
   * Connected to httpbin.example.com (172.20.127.212) port 31785 (#0)
   * Initializing NSS with certpath: sql:/etc/pki/nssdb
   *   CAfile: example.com.crt
     CApath: none
   * SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
   * Server certificate:
   *       subject: O=httpbin organization,CN=httpbin.example.com
   *       start date: May 21 06:31:34 2020 GMT
   *       expire date: May 21 06:31:34 2021 GMT
   *       common name: httpbin.example.com
   *       issuer: CN=example.com,O=example Inc.
   > GET /status/418 HTTP/1.1
   > User-Agent: curl/7.29.0
   > Accept: */*
   > Host:httpbin.example.com
   >
   < HTTP/1.1 418 Unknown
   < server: istio-envoy
   < date: Thu, 21 May 2020 06:34:49 GMT
   < x-more-info: http://tools.ietf.org/html/rfc2324
   < access-control-allow-origin: *
   < access-control-allow-credentials: true
   < content-length: 135
   < x-envoy-upstream-service-time: 7
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

   查看curl输出中的*Server certificate*中的信息，上述返回值的最后有一个茶壶的图片，说明运行成功。(*上述的返回结果与官方给的有所出入*)

### 配置一个mutual TLS ingress网关

本节会在外部的客户端和网关之间配置[mutual TLS](https://en.wikipedia.org/wiki/Mutual_authentication).

1. 创建一个保存CA的kubernetes `secret`，服务端可以使用该`secret`验证客户端。使用`kubectl`在`istio-system`命名空间中创建一个名为`istio-ingressgateway-ca-certs`的secret。istio会自动加载该secret：

   > `istio-system`命名空间中的secret的名字必须为`istio-ingressgateway-ca-certs`，与该任务中使用的Istio默认入口网关的配置一致。

   ```shell
   $ kubectl create -n istio-system secret generic istio-ingressgateway-ca-certs --from-file=example.com.crt
   ```

2. 将前面的Gateway的TLS模式变更为`MUTUAL`并指定`caCertificates`

   > 证书的位置必须是 `/etc/istio/ingressgateway-ca-certs`，否则gateway将无法加载。证书的名字必须与创建secret的证书相同，本例中为 `example.com.crt`。

   ```yaml
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: httpbin-gateway
   spec:
     selector:
       istio: ingressgateway # use istio default ingress gateway
     servers:
     - port:
         number: 443
         name: https
         protocol: HTTPS
       tls:
         mode: MUTUAL
         serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
         privateKey: /etc/istio/ingressgateway-certs/tls.key
         caCertificates: /etc/istio/ingressgateway-ca-certs/example.com.crt
       hosts:
       - "httpbin.example.com"
   EOF
   ```

3. 访问httpbin服务，返回如下错误

   ```shell
   $ curl -HHost:httpbin.example.com --resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST --cacert example.com.crt https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418
   
   curl: (35) NSS: client certificate not found (nickname not specified)
   ```

   由于没有提供client的证书，tls协商会失败

4. 为`httpbin.example.com`服务创建一个客户端证书，将`httpbin-client.example.com` URI分配给客户端.

   *注意：isito[官方文档](https://istio.io/docs/tasks/traffic-management/ingress/secure-ingress-mount/#configure-a-mutual-tls-ingress-gateway)中下面命令中会将`httpbin-client.example.com.crt`的serial number设置为0，这会与上面创建的`/etc/istio/ingressgateway-certs/tls.crt`的serial number相同，导致出现`SEC_ERROR_REUSED_ISSUER_AND_SERIAL`错误，因此下面将`set_serial`的值设置为1*

   ```shell
   $ openssl req -out httpbin-client.example.com.csr -newkey rsa:2048 -nodes -keyout httpbin-client.example.com.key -subj "/CN=httpbin-client.example.com/O=httpbin's client organization"
   $ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 1 -in httpbin-client.example.com.csr -out httpbin-client.example.com.crt
   ```

5. 带上客户端证书重新执行curl，可以看到执行成功：

   ```shell
   $ curl -HHost:httpbin.example.com --resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST --cacert ./example.com.crt --cert ./httpbin-client.example.com.crt --key ./httpbin-client.example.com.key https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418
   
       -=[ teapot ]=-
   
          _...._
        .'  _ _ `.
       | ."` ^ `". _,
       \_;`"---"`|//
         |       ;/
         \_     _/
           `"""`
   ```

### 为多个主机配置TLS ingress网关

本节会为多个主机(`httpbin.example.com`和`bookinfo.com`.)配置一个ingress网关。ingress网关会给客户端请求的每个服务端提供唯一的证书。

与前面章节不同，istio的默认ingress网关将无法正在工作，因为它预配置为仅支持一个安全主机。因此需要首先使用另一个secret配置并重新部署ingress网关服务，这样才能处理第二个主机。

#### 为`bookinfo.com`创建一个服务端证书和私钥

```shell
$ openssl req -out bookinfo.com.csr -newkey rsa:2048 -nodes -keyout bookinfo.com.key -subj "/CN=bookinfo.com/O=bookinfo organization"
$ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 2 -in bookinfo.com.csr -out bookinfo.com.crt
```

#### 使用新的证书重新部署`istio-ingressgateway` 

1. 创建一个新的secret，保存了 `bookinfo.com`的证书：

   ```shell
   $ kubectl create -n istio-system secret tls istio-ingressgateway-bookinfo-certs --key bookinfo.com.key --cert bookinfo.com.crt
   ```

2. 为了挂载新创建的secret，需要更新`istio-ingressgateway`的deployment。修改`istio-ingressgateway`的deployment，创建如下 `gateway-patch.json` 文件。经过下面修改，会在deployment中增加一个卷，将secret `istio-ingressgateway-bookinfo-certs`挂载到`/etc/istio/ingressgateway-bookinfo-certs`中

   ```shell
   $ cat > gateway-patch.json <<EOF
   [{
     "op": "add",
     "path": "/spec/template/spec/containers/0/volumeMounts/0",
     "value": {
       "mountPath": "/etc/istio/ingressgateway-bookinfo-certs",
       "name": "ingressgateway-bookinfo-certs",
       "readOnly": true
     }
   },
   {
     "op": "add",
     "path": "/spec/template/spec/volumes/0",
     "value": {
     "name": "ingressgateway-bookinfo-certs",
       "secret": {
         "secretName": "istio-ingressgateway-bookinfo-certs",
         "optional": true
       }
     }
   }]
   EOF
   ```

3. 使用如下命令修改 `istio-ingressgateway` deployment

   ```shell
   $ kubectl -n istio-system patch --type=json deploy istio-ingressgateway -p "$(cat gateway-patch.json)"
   ```

4. 校验 `istio-ingressgateway` pod成功加载了密钥和证书，即目录中出现`tls.crt`和`tls.key` 

   ```shell
   $ kubectl exec -it -n istio-system $(kubectl -n istio-system get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}') -- ls -al /etc/istio/ingressgateway-bookinfo-certs
   ```

#### 配置`bookinfo.com` 主机流量

1. 创建不带gateway的[bookinfo](https://istio.io/docs/examples/bookinfo/)应用

   ```shell
   $ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
   ```

2. 为 `bookinfo.com`定义一个gateway：

   ```shell
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: bookinfo-gateway
   spec:
     selector:
       istio: ingressgateway # use istio default ingress gateway
     servers:
     - port:
         number: 443
         name: https-bookinfo
         protocol: HTTPS
       tls: # tls用到了上面secret挂载的证书和密钥
         mode: SIMPLE
         serverCertificate: /etc/istio/ingressgateway-bookinfo-certs/tls.crt
         privateKey: /etc/istio/ingressgateway-bookinfo-certs/tls.key
       hosts:
       - "bookinfo.com" #外部主机
   EOF
   ```

3. 为`bookinfo.com`定义一个路由。定义一个根[`samples/bookinfo/networking/bookinfo-gateway.yaml`](https://raw.githubusercontent.com/istio/istio/release-1.5/samples/bookinfo/networking/bookinfo-gateway.yaml)类似的`VirtualService`

   ```shell
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: bookinfo
   spec:
     hosts:
     - "bookinfo.com"
     gateways:
     - bookinfo-gateway
     http:
     - match:
       - uri:
           exact: /productpage
       - uri:
           exact: /login
       - uri:
           exact: /logout
       - uri:
           prefix: /api/v1/products
       route:
       - destination:
           host: productpage
           port:
             number: 9080
   EOF
   ```

4. 向Bpokinfo `productpage`发送请求：

   ```shell
   $ curl -o /dev/null -s -v -w "%{http_code}\n" -HHost:bookinfo.com --resolve bookinfo.com:$SECURE_INGRESS_PORT:$INGRESS_HOST --cacert example.com.crt -HHost:bookinfo.com https://bookinfo.com:$SECURE_INGRESS_PORT/productpage
   
   * Added bookinfo.com:31785:172.20.127.211 to DNS cache
   * About to connect() to bookinfo.com port 31785 (#0)
   *   Trying 172.20.127.211...
   * Connected to bookinfo.com (172.20.127.211) port 31785 (#0)
   * Initializing NSS with certpath: sql:/etc/pki/nssdb
   *   CAfile: example.com.crt
     CApath: none
   * SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
   * Server certificate:
   *       subject: O=bookinfo organization,CN=bookinfo.com
   *       start date: May 21 13:28:51 2020 GMT
   *       expire date: May 21 13:28:51 2021 GMT
   *       common name: bookinfo.com
   *       issuer: CN=example.com,O=example Inc.
   > GET /productpage HTTP/1.1
   > User-Agent: curl/7.29.0
   > Accept: */*
   > Host:bookinfo.com
   > Host:bookinfo.com
   >
   < HTTP/1.1 404 Not Found
   < date: Thu, 21 May 2020 13:34:23 GMT
   < server: istio-envoy
   < content-length: 0
   <
   * Connection #0 to host bookinfo.com left intact
   404
   ```

5. 校验`httbin.example.com` 是否也跟签名一样可达。

   ```shell
   $ curl -HHost:httpbin.example.com --resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST --cacert example.com.crt --cert ./httpbin-client.example.com.crt --key httpbin-client.example.com.key https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418
   
       -=[ teapot ]=-
   
          _...._
        .'  _ _ `.
       | ."` ^ `". _,
       \_;`"---"`|//
         |       ;/
         \_     _/
   ```

### 问题定位

- 检查`INGRESS_HOST`和`SECURE_INGRESS_PORT`环境变量

  ```shell
  $ kubectl get svc -n istio-system
  $ echo INGRESS_HOST=$INGRESS_HOST, SECURE_INGRESS_PORT=$SECURE_INGRESS_PORT
  ```

- 校验密钥和证书被成功加载到`istio-ingressgateway` pod中，存在`tls.crt` 和`tls.key`

  ```shell
  $ kubectl exec -it -n istio-system $(kubectl -n istio-system get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}') -- ls -al /etc/istio/ingressgateway-certs
  ```

- 如果创建了`istio-ingressgateway-certs` secret，但没有加载密钥和证书，删除ingress网关，强制ingress网关pod重新加载key和证书

  ```shell
  $ kubectl delete pod -n istio-system -l istio=ingressgateway
  ```

- 校验证书中的Subject字段是正确的

  ```shell
  $ kubectl exec -i -n istio-system $(kubectl get pod -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].metadata.name}')  -- cat /etc/istio/ingressgateway-certs/tls.crt | openssl x509 -text -noout | grep 'Subject:'
          Subject: CN=httpbin.example.com, O=httpbin organization
  ```

- 校验ingress网关的代理是否已经知道配置了证书

  ```shell
  $ kubectl exec -ti $(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].metadata.name}') -n istio-system -- pilot-agent request GET certs
  ```

- 校验`istio-ingressgateway` 的日志

  ```shell
  $ kubectl logs -n istio-system -l istio=ingressgateway
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

### 卸载

1. 删除`Gateway`配置，`VirtualService`和secret

   ```shell
   $ kubectl delete gateway --ignore-not-found=true httpbin-gateway bookinfo-gateway
   $ kubectl delete virtualservice httpbin
   $ kubectl delete --ignore-not-found=true -n istio-system secret istio-ingressgateway-certs istio-ingressgateway-ca-certs
   $ kubectl delete --ignore-not-found=true virtualservice bookinfo
   ```

2. 删除证书目录

   ```shell
   $ rm -rf example.com.crt example.com.key httpbin.example.com.crt httpbin.example.com.key httpbin.example.com.csr httpbin-client.example.com.crt httpbin-client.example.com.key httpbin-client.example.com.csr bookinfo.com.crt bookinfo.com.key bookinfo.com.csr
   ```

3. 删除重新部署`istio-ingressgateway`时的补丁文件：

   ```shell
   $ rm -f gateway-patch.json
   ```

4. 停止`httpbin`服务：

   ```shell
   $ kubectl delete --ignore-not-found=true -f samples/httpbin/httpbin.yaml
   ```

## 安全网关(SDS)

本节讲述如何使用simple或mutual TLS暴露安全的HTTPS服务。

TLS需要的私钥，服务端证书，根证书是使用基于Secret Discovery Service (SDS)方法配置的。

执行[ingress流量控制](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/)中的[Before you begin](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control#before-you-begin) 和[Determining the ingress IP and ports](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)小节的操作，部署`httpbin`服务，并获取 `INGRESS_HOST` 和`SECURE_INGRESS_PORT`变量。

### 生成客户端和服务端的证书和密钥

1. 创建根证书和私钥，用于签名服务

   ```shell
   $ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
   ```

2. 为 `httpbin.example.com`创建证书和私钥

   ```shell
   $ openssl req -out httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout httpbin.example.com.key -subj "/CN=httpbin.example.com/O=httpbin organization"
   $ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in httpbin.example.com.csr -out httpbin.example.com.crt
   ```

使用SDS配置TLS ingress网关







