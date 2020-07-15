# Istio安全-认证

[TOC]

## 认证策略

本节会介绍如何启用，配置和使用istio的认证策略，了解更多关于[认证](https://istio.io/latest/docs/concepts/security/#authentication)的底层概念。

首先了解istio的[认证策略](https://istio.io/latest/docs/concepts/security/#authentication-policies)和相关的[mutual TLS认证](https://istio.io/latest/docs/concepts/security/#mutual-tls-authentication)概念，然后使用`default`配置安装istio

### 配置

下面例子会创建两个命名空间`foo`和`bar`，以及两个服务`httpbin`和`sleep`，这两个服务都运行了Envoy代理。其次还在`legacy`命名空间中运行了不带sidecar的`httpbin`和`sleep`的实例。*最好使用自动注入sidecar的方式*。

```shell
$ kubectl create ns foo
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml) -n foo
$ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml) -n foo
$ kubectl create ns bar
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml) -n bar
$ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml) -n bar
$ kubectl create ns legacy
$ kubectl apply -f samples/httpbin/httpbin.yaml -n legacy
$ kubectl apply -f samples/sleep/sleep.yaml -n legacy
```

使用curl验证配置是否正确，从`foo`, `bar` 或`legacy`中的sleep pod向`httpbin.foo`, `httpbin.bar` 或`httpbin.legacy`发送HTTP请求，所有的请求都应该返回HTTP 200状态码。

```shell
$ kubectl exec $(kubectl get pod -l app=sleep -n bar -o jsonpath={.items..metadata.name}) -c sleep -n bar -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n"
```

下面命令可以方便地遍历所有可达性组合：

```shell
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
sleep.foo to httpbin.foo: 200
sleep.foo to httpbin.bar: 200
sleep.foo to httpbin.legacy: 200
sleep.bar to httpbin.foo: 200
sleep.bar to httpbin.bar: 200
sleep.bar to httpbin.legacy: 200
sleep.legacy to httpbin.foo: 200
sleep.legacy to httpbin.bar: 200
sleep.legacy to httpbin.legacy: 200
```

校验没有配置对等认证策略

```shell
$ kubectl get peerauthentication --all-namespaces
No resources found.
```

校验没有为例子的服务配置任何destination rules。可以通过校验现有destination rules是否存在`host:`来查看是否存在匹配的内容：

```shell
$ kubectl get destinationrules.networking.istio.io --all-namespaces -o yaml | grep "host:"
```

### 自动mutual TLS

默认情况下，istio会跟踪迁移到Istio代理的服务器工作负载，并自动配置客户端代理发送mutual TLS流量到这些负载，并发送明文流量到没有sidecar的负载。

因此拥有代理的负载之间的流量会使用mutual TLS。例如，获取发送到`httpbin/header`的请求对应的响应，当使用mutual TLS时，代理会在到达后端的上游请求中注入 `X-Forwarded-Client-Cert` 首部。出现该首部表明使用了mutual TLS：

```shell
$ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl http://httpbin.foo:8000/headers -s | grep X-Forwarded-Client-Cert
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/foo/sa/httpbin;Hash=4b69f5cb0582b9a06f2178666d1fc082ec7538aa76eb29e28a5e048713ced049;Subject=\"\";URI=spiffe://cluster.local/ns/foo/sa/sleep"
```

当服务端没有sidecar，则请求中不会被注入 `X-Forwarded-Client-Cert` 首部，暗示请求使用了明文：

```shell
$ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl http://httpbin.legacy:8000/headers -s | grep X-Forwarded-Client-Cert
```

### 全局启用istio的mutual TLS STRIC模式

由于istio会自动将代理和负载之间的流量升级到mutual TLS，此时负载仍然接收明文流量。为了防止整个网格中出现非mutual TLS，需要在网格范围将对等认证策略设置为mutual TLS `STRICT`。网格范围的对等认证策略不应该出现`selector`字段，且必须应用到根命名空间：

```shell
$ kubectl apply -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
  namespace: "istio-system"
spec:
  mtls:
    mode: STRICT
EOF
```

> 注：上例中将`istio-system`假设为根命名空间，如果安装时选用了不同的命名空间，则使用该命名空间替换`istio-system`

对等认证策略会产生如下影响：网格中所有的负载只能接收使用TLS加密的请求。由于没有使用`selector`字段指定值，因此该策略会应用到网格中的所有负载。

重新执行如下命令：

```shell
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
sleep.foo to httpbin.foo: 200
sleep.foo to httpbin.bar: 200
sleep.foo to httpbin.legacy: 200
sleep.bar to httpbin.foo: 200
sleep.bar to httpbin.bar: 200
sleep.bar to httpbin.legacy: 200
sleep.legacy to httpbin.foo: 000
command terminated with exit code 56
sleep.legacy to httpbin.bar: 000
command terminated with exit code 56
sleep.legacy to httpbin.legacy: 200
```

可以看到不包含代理的客户端(`sleep.legacy`)的到包含代理的服务端(`httpbin.foo`或`httpbin.bar`.)的请求失败了。由于此时使用了严格的mutual TLS，但不包含代理的负载无法满足该要求。

### 卸载

```shell
$ kubectl delete peerauthentication -n istio-system default
```

### 针对单个命名空间或负载启用mutual TLS

#### 命名空间范围的策略

为了修改特定命名空间内的所有负载的mutual TLS，需要使用命名空间范围的策略。命名空间范围的策略与网格范围的策略的规范相同，但需要在`metadata`下指定命名空间。例如，下面在`foo`命名空间中启用了严格的mutual TLS对等认证策略。

```shell
$ kubectl apply -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
  namespace: "foo"
spec:
  mtls:
    mode: STRICT
EOF
```

由于该策略仅对foo命名空间生效，可以看到只有`sleep.legacy`(不包含代理)到 `httpbin.foo`的请求失败了

```shell
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
sleep.foo to httpbin.foo: 200
sleep.foo to httpbin.bar: 200
sleep.foo to httpbin.legacy: 200
sleep.bar to httpbin.foo: 200
sleep.bar to httpbin.bar: 200
sleep.bar to httpbin.legacy: 200
sleep.legacy to httpbin.foo: 000
command terminated with exit code 56
sleep.legacy to httpbin.bar: 200
sleep.legacy to httpbin.legacy: 200
```

#### 为单个负载启用mutual TLS

为了给特定的负载设置对等认证策略，需要使用`selector`字段指定匹配到期望负载的标签。然而，Istio无法为(到达服务的)出站的mutual TLS流量聚合工作负载级别的策略(*可以理解为对等认证策略是匹配负载(如pod)的，还需要destination rule匹配服务(DNS)*)，需要配置destination rule管理该行为。

例如，下面对等认证策略和destination rule为`httpbin.bar`负载启用了严格的mutual TLS。

```shell
$ cat <<EOF | kubectl apply -n bar -f -
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "httpbin"
  namespace: "bar"
spec:
  selector:
    matchLabels:
      app: httpbin
  mtls:
    mode: STRICT
EOF
```

```shell
$ cat <<EOF | kubectl apply -n bar -f -
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "httpbin"
spec:
  host: "httpbin.bar.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
EOF
```

执行探测命令，可以看到`sleep.legacy` 到`httpbin.bar`的请求失败了。

```shell
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
sleep.foo to httpbin.foo: 200
sleep.foo to httpbin.bar: 200
sleep.foo to httpbin.legacy: 200
sleep.bar to httpbin.foo: 200
sleep.bar to httpbin.bar: 200
sleep.bar to httpbin.legacy: 200
sleep.legacy to httpbin.foo: 000
command terminated with exit code 56
sleep.legacy to httpbin.bar: 000
command terminated with exit code 56
sleep.legacy to httpbin.legacy: 200
```

如果要为单个端口设置mutual TLS，则需要配置`portLevelMtls`字段。例如，下面对等认证策略需要在除了`80`的端口上启用mutual TLS

```shell
$ cat <<EOF | kubectl apply -n bar -f -
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "httpbin"
  namespace: "bar"
spec:
  selector:
    matchLabels:
      app: httpbin
  mtls:
    mode: STRICT
  portLevelMtls:
    80:
      mode: DISABLE
EOF
```

此时也需要一个destination rule

```shell
$ cat <<EOF | kubectl apply -n bar -f -
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "httpbin"
spec:
  host: httpbin.bar.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
    portLevelSettings:
    - port:
        number: 8000
      tls:
        mode: DISABLE
EOF
```

1. 对等认证策略中的端口值为容器的端口，而destination rule中的值为service的端口
2. 仅当端口绑定到服务时才能使用`portLevelMtls`。其他情况下，istio会忽略该字段

#### 策略优先级

指定负载的对等认证策略要优先于命名空间范围的策略。可以通过禁用`httpbin.foo`负载的mutual TLS来测试这种特性。注意，foo命名空间已经启用了命名空间范围的mutual TLS，从`sleep.legacy` 到`httpbin.foo`的请求会失败(见上文)。

```shell
$ cat <<EOF | kubectl apply -n foo -f -
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "overwrite-example"
  namespace: "foo"
spec:
  selector:
    matchLabels:
      app: httpbin
  mtls:
    mode: DISABLE
EOF
```

```shell
$ cat <<EOF | kubectl apply -n foo -f -
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "overwrite-example"
spec:
  host: httpbin.foo.svc.cluster.local
  trafficPolicy:
    tls:
      mode: DISABLE
EOF
```

重新从`sleep.legacy`发起请求，可以看到成功返回200，表明指定服务的策略要优先于指定命名空间的策略。

```shell
$ kubectl exec $(kubectl get pod -l app=sleep -n legacy -o jsonpath={.items..metadata.name}) -c sleep -n legacy -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n"
200
```

此时`foo`命名空间中有2个对等认证策略。

```shell
$ oc get peerauthentications.security.istio.io
NAME                AGE
default             16h
overwrite-example   106s
```

#### 卸载

```shell
$ kubectl delete peerauthentication default overwrite-example -n foo
$ kubectl delete peerauthentication httpbin -n bar
$ kubectl delete destinationrules overwrite-example -n foo
$ kubectl delete destinationrules httpbin -n bar
```

### 终端用户认证

为了试验该特性，需要一个有效的JWT。JWT必须与使用的JWKS终端相匹配。本节测试[JWT test](https://raw.githubusercontent.com/istio/istio/release-1.6/security/tools/jwt/samples/demo.jwt) 和[JWKS endpoint](https://raw.githubusercontent.com/istio/istio/release-1.6/security/tools/jwt/samples/jwks.json)。

为了方便，通过`ingressgateway`暴露`httpbin.foo`。

```shell
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
  namespace: foo
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
EOF
```

```shell
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
  namespace: foo
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - route:
    - destination:
        port:
          number: 8000
        host: httpbin.foo.svc.cluster.local
EOF
```

获取ingress IP

```shell
$ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

运行一个测试请求。*PS：官方文档好像有点[问题](https://github.com/istio/istio.io/issues/7730)，直接在ingressgateway pod中执行测试*

```shell
# curl 127.0.0.1:8080/headers  -s -o /dev/null -w "%{http_code}\n"
200
```

为ingress gateway添加一个需要终端用户JWT的请求认证策略

```shell
$ kubectl apply -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "RequestAuthentication"
metadata:
  name: "jwt-example"
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.6/security/tools/jwt/samples/jwks.json"
EOF
```

将该策略应用到负载(`ingressgateway`)上，选择的命名空间为`istio-system`。

如果在认证首部提供了token，istio会使用[公钥集](https://raw.githubusercontent.com/istio/istio/release-1.6/security/tools/jwt/samples/jwks.json)进行认证，如果token无效，则请求会被拒绝。但是会接收不带token的请求。为了观察这种行为，发送不带token，带错误的token和带无效token的请求。

```shell
# curl 127.0.0.1:8080/headers -s -o /dev/null -w "%{http_code}\n"
200
```

```shell
# curl --header "Authorization: Bearer deadbeef" 127.0.0.1:8080/headers -s -o /dev/null -w "%{http_code}\n"
401
```

```shell
# curl --header "Authorization: Bearer $TOKEN" 127.0.0.1:8080/headers -s -o /dev/null -w "%{http_code}\n"
200
```

为了观察JWT验证的其他方面，使用 [`gen-jwt.py`](https://github.com/istio/istio/tree/release-1.6/security/tools/jwt/samples/gen-jwt.py) 生成新的token来测试不同的issuer，audiences，expiry date等。该脚本可以从istio的库中下载：

```shell
$ wget https://raw.githubusercontent.com/istio/istio/release-1.6/security/tools/jwt/samples/gen-jwt.py
$ chmod +x gen-jwt.py
```

此外还需要用到`key.pem`文件

```shell
$ wget https://raw.githubusercontent.com/istio/istio/release-1.6/security/tools/jwt/samples/key.pem
```

例如，一下命令会创建一个token，5s过期。可以看到istio认证请求一开始是成功的，5s后会被拒绝。

```shell
$ TOKEN=$(./gen-jwt.py ./key.pem --expire 5)
$ for i in `seq 1 10`; do curl --header "Authorization: Bearer $TOKEN" $INGRESS_HOST/headers -s -o /dev/null -w "%{http_code}\n"; sleep 1; done
200
200
200
200
200
401
401
401
401
401
```

也可以给ingress gateway配置一个JWT策略。通常用于为绑定到网关的所有服务定义JWT策略，而不只为单个服务定义JWT策略。

#### 请求有效的token

为了拒绝不带有效token的请求，需要添加一个`DENY`字段来处理无请求主体的请求，如下的`notRequestPrincipals: ["*"]`。只有提供了有效的JWT token后，才会认为请求主体是有效的。下面规则会拒绝没有有效token的请求。

```shell
$ kubectl apply -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "frontend-ingress"
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
EOF
```

重新请求，可以发现此时不带token的请求返回了403错误码：

```shell
# curl 127.0.0.1:8080/headers  -s -o /dev/null -w "%{http_code}\n"
403
```

#### 为每条路径请求有效的token

要使用基于每个主机、路径或方法的token来优化授权，需要将授权策略更改为只对/headers生效。当授权规则生效时，对 `$INGRESS_HOST/headers`的请求会返回错误码403，而针对其他路径的请求则会成功，如`$INGRESS_HOST/ip`。

```shell
$ kubectl apply -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "frontend-ingress"
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
    to:
    - operation:
        paths: ["/headers"]
EOF
```

```shell
# curl 127.0.0.1:8080/headers  -s -o /dev/null -w "%{http_code}\n"
403
# curl 127.0.0.1:8080/ip -s -o /dev/null -w "%{http_code}\n"
200
```

### 卸载

移除认证策略

```shell
$ kubectl -n istio-system delete requestauthentication jwt-example
$ kubectl -n istio-system delete authorizationpolicy frontend-ingress
```

移除pod

```shell
$ kubectl delete ns foo bar legacy
```

## Mutual TLS迁移

本节展示如何保证负载迁移到istio后仅使用mutual TLS通信。

当调用其它负载时，istio会自动配置负载sidecar使用mutual TLS。istio默认会使用`PERMISSIVE`模式配置目标负载。当启用`PERMISSIVE`模式时，服务可以同时接收明文和mutual TLS的流量。为了仅允许mutual TLS流量，需要将配置切换为`STRICT`模式。

可以使用[Grafana dashboard](https://istio.io/latest/docs/tasks/observability/metrics/using-istio-dashboard/)校验哪些负载会发送明文流量到使用PERMISSIVE模式的负载。

### 配置集群

创建两个命名空间，`foo`和`bar`，部署带sidecar的[httpbin](https://github.com/istio/istio/tree/release-1.6/samples/httpbin)和[sleep](https://github.com/istio/istio/tree/release-1.6/samples/sleep) 

```shell
$ kubectl create ns foo
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml) -n foo
$ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml) -n foo
$ kubectl create ns bar
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml) -n bar
$ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml) -n bar
```

创建另外一个命名空间`legacy`，并部署不带sidecar的sleep

```shell
$ kubectl create ns legacy
$ kubectl apply -f samples/sleep/sleep.yaml -n legacy
```

从三个命名空间的sleep pod中发送请求到`httpbin.foo`，所有的请求都应该返回HTTP 200状态码。

```shell
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar"; do kubectl exec "$(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name})" -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
sleep.foo to httpbin.foo: 200
sleep.foo to httpbin.bar: 200
sleep.bar to httpbin.foo: 200
sleep.bar to httpbin.bar: 200
sleep.legacy to httpbin.foo: 200
sleep.legacy to httpbin.bar: 200
```

保证没有认证策略或destination rules

```shell
$ kubectl get peerauthentication --all-namespaces | grep -v istio-system
NAMESPACE      NAME                          AGE
```

```shell
$ kubectl get destinationrule --all-namespaces
No resources found.
```

### 按命名空间锁定mutual TLS

在将所有的客户端迁移到istio并注入Envoy sidecar后，配置foo命名空间仅允许接收mutual TLS流量。

```shell
$ kubectl apply -n foo -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
```

此时从 `sleep.legacy` 到 `httpbin.foo` 的请求会失败：

```shell
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar"; do kubectl exec "$(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name})" -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
sleep.foo to httpbin.foo: 200
sleep.foo to httpbin.bar: 200
sleep.bar to httpbin.foo: 200
sleep.bar to httpbin.bar: 200
sleep.legacy to httpbin.foo: 000
command terminated with exit code 56
sleep.legacy to httpbin.bar: 200
```

如果在安装istio时启用了 `values.global.proxy.privileged=true`，则可以使用`tcpdump`校验流量是否加密。

```shell
$ kubectl exec -nfoo "$(kubectl get pod -nfoo -lapp=httpbin -ojsonpath={.items..metadata.name})" -c istio-proxy -it -- sudo tcpdump dst port 80  -A
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
```

如果无法将所有的服务迁移到istio，则需要使用`PERMiISSIVE`模式。但是如果使用了PERMISSIVE模式，则不会使用任何认证和授权，默认使用明文流量。推荐使用[istio认证](https://istio.io/latest/docs/tasks/security/authorization/authz-http/)为不同的路径配置不同的策略。

### 为整个网格锁定mutual TLS

```shell
$ kubectl apply -n istio-system -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
```

现在 `foo` 和`bar`命名空间会强制使用mutual TLS流量，因此从`sleep.legacy`发出的所有请求都会失败。

```shell
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar"; do kubectl exec "$(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name})" -c sleep -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "sleep.${from} to httpbin.${to}: %{http_code}\n"; done; done
sleep.foo to httpbin.foo: 200
sleep.foo to httpbin.bar: 200
sleep.bar to httpbin.foo: 200
sleep.bar to httpbin.bar: 200
sleep.legacy to httpbin.foo: 000
command terminated with exit code 56
sleep.legacy to httpbin.bar: 000
command terminated with exit code 56
```

### 卸载

```shell
$ kubectl delete peerauthentication --all-namespaces --all
```

```shell
kubectl delete ns foo bar legacy
```