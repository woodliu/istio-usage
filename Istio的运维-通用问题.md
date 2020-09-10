# Istio的运维-通用问题

[TOC]

## 流量管理问题

### Envoy拒绝了请求

请求可能由于各种原因而被拒绝，可以通过查看Envoy的日志来了解被拒绝的原因。默认情况下，日志会打印到容器的标准输出中，使用如下命令查看日志。通过这种方式也可以查看进出容器sidecar代理的流量信息。

```shell
$ kubectl logs PODNAME -c istio-proxy -n NAMESPACE
```

istio的[默认日志格式](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#default-format-string)如下。使用`meshConfig.accessLogFormat`配置日志输出格式，使用`istioctl proxy-config listeners <pod> -n <namespace> -o json`查看日志格式。

```
"format": "[%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%DYNAMIC_METADATA(istio.mixer:status)%\" \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"
```

下面是对http://www.baidu.com的访问日志。对日志的解析可以参考百分号`%`和引号，每个百分号对应一个字段，日志引号内的内容与日志格式是对应的，如`"GET / HTTP/1.1"`对应`"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%"`。

```shell
"GET / HTTP/1.1" 200 - "-" "-" 0 2381 31 31 "-" "curl/7.35.0" "dbdddb94-d1f5-4bdb-9fd4-a2e4002836d1" "www.baidu.com" "180.101.49.11:80" PassthroughCluster 10.83.0.36:60858 180.101.49.11:80 10.83.0.36:60856 - allow_any
```

需要注意的是Envoy的`RESPONSE_FLAGS`字段，一般的响应字段如下，更多参见Envoy的[响应标识](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags)

- `NR`：没有配置路由，检查`DestinationRule` 或`VirtualService`。
- `UO`: 上游发生了熔断，检查`DestinationRule`中的断路器设置。
- `UF`: 无法连接上游, 如果使用了istio认证，按照 [mutual TLS configuration conflict](https://istio.io/docs/ops/common-problems/network-issues/#503-errors-after-setting-destination-rule)进行检查。

### 路由规则似乎没有影响到流量

当前Envoy sidecar实现中，可能需要达到100个请求才能观测到不同版本间基于权重的流量分发。

如果在Bookinfo例子中，路由规则工作正常，但类似的版本路由规则对自己的应用却没有生效。可能是因为kubernetes service需要进行略微修改。kubernetes service必须必须遵守某些限制来使用Istio的路由特性。更多信息参见[Requirements for Pods and Services](https://istio.io/latest/docs/ops/deployment/requirements/)。

另外一个潜在的问题是路由规则可能需要等待一段时间才能生效。Istio在kubernetes的实现中使用了一个最终一致算法来保证所有的Envoy sidecar具有正确的配置(包括路由规则)。一个配置需要一定时间才能传递到所有的sidecar中。对于大型部署，将花费更长的时间进行配置的传递，并且可能会有几秒钟的延迟时间。

### 设置destination rule之后发生503错误

> 只有在安装的时候禁用了[自动mutual TLS](https://istio.io/docs/tasks/security/authentication/authn-policy/#auto-mutual-tls)时才会发生该错误

当设置了`DestinationRule`之后发生了HTTP 503错误，而在移除`DestinationRule`之后，该错误消失，说明`DestinationRule`导致了TLS连接问题。

例如，如果集群范围配置了mutual TLS，`DestinationRule`必须包含如下`trafficPolicy`：

```yaml
trafficPolicy:
  tls:
    mode: ISTIO_MUTUAL
```

否则，会使用默认的模式`DISABLE`，这样会导致客户端的sidecar代理发送HTTP明文请求，而非TLS加密的HTTPS请求，这样就会产生请求冲突(服务端期望接收加密的请求)。

当使用`DestinationRule`时应该保证`trafficPolicy`的TLS模式与全局配置匹配。

### 路由规则对ingress网关请求没有影响

假设使用一个ingress `Gateway`和对应的`VirtualService`来访问内部服务。例如，`VirtualService`如下：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - "myapp.com" # or maybe "*" if you are testing without DNS using the ingress-gateway IP (e.g., http://1.2.3.4/hello)
  gateways:
  - myapp-gateway
  http:
  - match:
    - uri:
        prefix: /hello
    route:
    - destination:
        host: helloworld.default.svc.cluster.local
  - match:
    ...
```

同时还有一个VirtualService将helloworld服务的流量路由到特定的subset

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
  - helloworld.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: helloworld.default.svc.cluster.local
        subset: v1
```

这种情况下可以看到到达helloworld服务的请求会经过ingress网关，但不会定向到subset v1，仍然使用默认的轮询路由。

使用了网关host(如`myapp.com`)的ingress会使用myapp `VirtualService`中的规则将流量分发到helloworld服务的所有endpoint上，只有host为`helloworld.default.svc.cluster.local`的**内部**请求才会被定向到subset v1，这种情况下，两个`VirtualService`可以看作是相互独立的.

为了控制来自网关的流量，需要在myapp `VirtualService`中指定subset：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - "myapp.com" # or maybe "*" if you are testing without DNS using the ingress-gateway IP (e.g., http://1.2.3.4/hello)
  gateways:
  - myapp-gateway
  http:
  - match:
    - uri:
        prefix: /hello
    route:
    - destination:
        host: helloworld.default.svc.cluster.local
        subset: v1 #指定流量的subset
  - match:
    ...
```

当然也可以将两个`VirtualService`合并为一个：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.com # cannot use "*" here since this is being combined with the mesh services
  - helloworld.default.svc.cluster.local
  gateways:
  - mesh # applies internally as well as externally
  - myapp-gateway
  http:
  - match:
    - uri:
        prefix: /hello
      gateways:
      - myapp-gateway #restricts this rule to apply only to ingress gateway
    route:
    - destination:
        host: helloworld.default.svc.cluster.local
        subset: v1
  - match:
    - gateways:
      - mesh # applies to all services inside the mesh
    route:
    - destination:
        host: helloworld.default.svc.cluster.local
        subset: v1
```

### Envoy崩溃

使用`ulimit -a`进行校验打开文件的描述符数目，如果设置值过小，会导致Envoy断言并崩溃，日志如下。可以使用如`ulimit -n 16384`增大该限制。

```verilog
[2017-05-17 03:00:52.735][14236][critical][assert] assert failure: fd_ != -1: external/envoy/source/common/network/connection_impl.cc:58
```

### Envoy无法连接自己的HTTP/1.0服务

Envoy需要上游的流量为HTTP/1.1或HTTP/2。例如在Envoy之后使用NGINX管理流量时，由于NGINX的默认版本为1.0，因此需要在NGINX配置中设置[proxy_http_version](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_http_version)。

```nginx
upstream http_backend {
    server 127.0.0.1:8080;

    keepalive 16;
}

server {
    ...

    location /http/ {
        proxy_pass http://http_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        ...
    }
}
```

### 当多个网关配置相同的TLS证书后发生404错误

使用相同的TLS证书配置多个网关将导致利用HTTP/2连接重用特性的浏览器在与另一个主机建立连接后访问第二个主机时抛出404错误。

例如，有2台主机使用相同的TLS证书：

- `istio-ingressgateway`安装了通配证书`*.test.com`
- `Gateway gw1`的host为`service1.test.com`，selector为`istio: ingressgateway`，TLS使用网关挂载的证书
- `Gateway gw2`的host为`service2.test.com`，selector为`istio: ingressgateway`，TLS使用网关挂载的证书
- `VirtualService` `vs1`的host为`service1.test.com`，gateway为`gw1`
- `VirtualService` `vs2`的host为`service2.test.com`，gateway为`gw2`

由于两个网关都使用了相同的负载(`istio: ingressgateway`)将请求处理到服务`service1.test.com` 和`service2.test.com`的请求。如果先到了`service1.test.com`的请求，它会返回通配证书，指明到`service2.test.com`的连接可以使用相同的证书。像Chrome和Firefox这样的浏览器将会为`service2.test.com`重用现有的连接。由于网关`gw1`没有路由到`service2.test.com`，此时会返回404。

可以通过一个通配的`Gateway`来解决问题，避免使用两个网关(`gw1`和`gw2`)，并将两个`VirtualService` 绑定到该`Gateway`即可。

- `Gateway` `gw`的host为`*.test.com`，selector为`istio: ingressgateway`，TLS使用网关挂载的证书
- `VirtualService` `vs1`的host为`service1.test.com`，gateway为`gw`
- `VirtualService` `vs2`的host为`service2.test.com`，gateway为`gw`

### 在一个网关上配置多个TLS hosts时端口冲突

如果一个`Gateway`的配置中`selector`的标签与另外一个现有的`Gateway`相同，且这两个`Gateway`暴露了相同的HTTPS端口，此时需要保证端口名称的唯一性。否则，这种配置不会立即返回错误，且在gateway运行时会忽略这种错误。例如：

```yaml
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
      mode: SIMPLE
      serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
      privateKey: /etc/istio/ingressgateway-certs/tls.key
    hosts:
    - "myhost.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: mygateway2
spec:
  selector:
    istio: ingressgateway # use istio default ingress gateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
      privateKey: /etc/istio/ingressgateway-certs/tls.key
    hosts:
    - "myhost2.com"
```

使用上述配置时，由于两个网关的端口都配置为`name: https`，因此到第二个主机 `myhost2.com`的请求会失败，使用curl进行请求时会出现如下错误：

```shell
curl: (35) LibreSSL SSL_connect: SSL_ERROR_SYSCALL in connection to myhost2.com:443
```

在pilot日志中可以看到类似的信息：

```shell
$ kubectl logs -n istio-system $(kubectl get pod -l istio=pilot -n istio-system -o jsonpath={.items..metadata.name}) -c discovery | grep "non unique port"
2018-09-14T19:02:31.916960Z info    model   skipping server on gateway mygateway2 port https.443.HTTPS: non unique port name for HTTPS port
```

为了避免这种问题，需要保证多个`protocol: HTTPS`端口的名称是唯一的，例如，将第二个Gateway的端口名改为`https2`：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: mygateway2
spec:
  selector:
    istio: ingressgateway # use istio default ingress gateway
  servers:
  - port:
      number: 443
      name: https2
      protocol: HTTPS
    tls:
      mode: SIMPLE
      serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
      privateKey: /etc/istio/ingressgateway-certs/tls.key
    hosts:
    - "myhost2.com"
```

## 安全问题

### 终端用户认证失败

使用Istio，可以通过[请求认证策略](https://preliminary.istio.io/latest/docs/tasks/security/authentication/authn-policy/#end-user-authentication)来启用对用户的认证。通过如下步骤来进行故障排查。







