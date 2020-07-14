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

为了给特定的负载设置对等认证策略，需要使用`selector`字段指定匹配到期望负载的标签。然而，Istio无法为(到达服务的)出站的mutual TLS流量聚合工作负载级别的策略(*可以理解为对等认证策略是设定在负载(如pod)上的，还需要destination rule设置到服务(DNS)的策略*)，需要配置destination rule管理该行为。

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

### 策略优先级