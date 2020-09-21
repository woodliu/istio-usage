## Istio的配置分析

[TOC]

### Analyzer 的消息格式

 `istioctl analyze` 命令提供了如下消息格式：

```shell
<level> [<code>] (<affected-resource>) <message-details>
```

<affected-resource>字段可以展开为：

```
<resource-kind> <resource-name>.<resource-namespace>
```

例如：

```
Error [IST0101] (VirtualService httpbin.default) Referenced gateway not found: "httpbin-gateway-bogus"
```

 `<message-details>`字段包含关于解决问题的详细信息。当对于集群范围的资源(如`namespace`)时，会忽略namespace前缀。

### ConflictingMeshGatewayVirtualServiceHosts

| Message Name | ConflictingMeshGatewayVirtualServiceHosts                    |
| ------------ | ------------------------------------------------------------ |
| Message Code | IST0109                                                      |
| Description  | Conflicting hosts on VirtualServices associated with mesh gateway |
| Level        | Error                                                        |

当Istio检测到virtual service资源之间存在重叠导致的冲突时，会出现该消息。例如，定义了多个使用相同的主机名的virtual service，并将其附加到网格网关上，这样就会产生上述错误。注意，Istio支持合并附加到ingress网关的virtual services。

#### 问题解决

为了解决该问题，可以使用如下动作来解决该问题：

- 将多个冲突的virtual service合并为一个
- 将附加到一个网格网关的多个virtual service的主机名配置为唯一的
- 通过`exportTo`字段将资源指定到某个指定的命名空间中

#### 举例

例如，`team1`命名空间中的 `productpage` virtual service 与`team2`命名空间中的`custom` virtual service因为同时设置了如下条件导致了冲突：

- 都附加到了默认的`mesh`网关上(没有指定用户网关)
- 定义了相同的host `productpage.default.svc.cluster.local`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
  namespace: team-1
spec:
  hosts:
  - productpage.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: productpage
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: custom
  namespace: team-2
spec:
  hosts:
  - productpage.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: productpage.team-2.svc.cluster.local
---
```

可以通过设置exportTo字段来解决该问题，这样，virtual service的范围就限制于其所在的命名空间。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
  namespace: team-1
spec:
  exportTo:
  - "." #当前命名空间
  hosts:
  - productpage.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: productpage
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: custom
  namespace: team-2
spec:
  exportTo:
  - "." #当前命名空间
  hosts:
  - productpage.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: productpage.team-2.svc.cluster.local
---
```

### ConflictingSidecarWorkloadSelectors

| Message Name | ConflictingSidecarWorkloadSelectors                          |
| ------------ | ------------------------------------------------------------ |
| Message Code | IST0110                                                      |
| Description  | A Sidecar resource selects the same workloads as another Sidecar resource |
| Level        | Error                                                        |

当一个命名空间中的多个sidecar选择相同的负载实例时会出现该消息，可能导致不可预知的行为。更多信息参见[Sidecar](https://istio.io/latest/docs/reference/config/networking/sidecar/)资源。

为了解决该问题，需要一个命名空间中的Sidecar负载选择器(`workloadSelector`)选择的负载实例不会出现重叠。

### GatewayPortNotOnWorkload

| Message Name | GatewayPortNotOnWorkload |
| ------------ | ------------------------ |
| Message Code | IST0104                  |
| Description  | Unhandled gateway port   |
| Level        | Warning                  |

当一个网关(通常是`istio-ingressgateway`)提供的端口并不在网关选择的kubernetes service负载上时会出现该消息。

例如，Istio的配置包含如下值：

```yaml
# Gateway with bogus port

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
  - port:
      number: 8004
      name: http2
      protocol: HTTP
    hosts:
    - "*"
```

在上面例子中会出现`GatewayPortNotOnWorkload`消息，因为一个默认的IngressGateway仅会打开端口 80, 443, 31400, 和15443，并不包括8004。使用istioctl analyze分析的结果如下：

```shell
# istioctl analyze
Warn [IST0104] (Gateway httpbin-gateway.default) The gateway refers to a port that is not exposed on the workload (pod selector istio=ingressgateway; port 8004)
Info [IST0118] (Service mutatepodimages.default) Port name  (port: 443, targetPort: 8443) doesn't follow the naming convention of Istio port.
Error: Analyzers found issues when analyzing namespace: default.
```

为了解决该问题，可以修改网关的配置，使用一个有效的负载端口，并重新部署即可。

### InternalError

| Message Name | InternalError                                                |
| ------------ | ------------------------------------------------------------ |
| Message Code | IST0001                                                      |
| Description  | There was an internal error in the toolchain. This is almost always a bug in the implementation. |
| Level        | Error                                                        |

极有可能是因为Istio的内部错误造成的。可以参见Istio的[issue页面](https://github.com/istio/istio/issues)来了解会提交问题。

### IstioProxyImageMismatch

| Message Name | IstioProxyImageMismatch                                      |
| ------------ | ------------------------------------------------------------ |
| Message Code | IST0105                                                      |
| Description  | The image of the Istio proxy running on the pod does not match the image defined in the injection configuration. |
| Level        | Warning                                                      |

当一个Pod出现如下条件时会发生该问题：

- 启用sidecar自动注入(默认是启用的，除非在安装时禁用)
- pod运行在一个启用sidecar注入的命名空间中(给命名空间打上标签`istio-injection=enabled`)
- 运行在sidecar中的代理版本与自动注入的版本不匹配

该问题通常会发生在更新Istio的控制面之后，在升级Istio后，所有带Istio sidecar运行的负载必须重启来注入新版本的sidecar。

为了解决该问题，可以通过滚动策略来重新部署应用的sidecar。对于kubernetes的deployment：

1. 如果使用kubernetes 1.15或更高版本，可以运行 `kubectl rollout restart <my-deployment>`来触发滚动
2. 或者，可以修改deployment的`template`字段来强制触发滚动。这通常是通过在模板的pod定义中添加诸如`force-redeploy = <current-timestamp>`之类的标签来触发deployment滚动的。

### JwtFailureDueToInvalidServicePortPrefix

| Message Name | JwtFailureDueToInvalidServicePortPrefix                      |
| ------------ | ------------------------------------------------------------ |
| Message Code | IST0119                                                      |
| Description  | Authentication policy with JWT targets Service with invalid port specification. |
| Level        | Warning                                                      |

当一个认证策略使用JWT认证时，而目标kubernetes service配置不正确时会出现该消息。正确的kubernetes service要求port使用http|http2|https作为前缀来命名(参见[Protocol Selection](https://istio.io/latest/docs/ops/configuration/traffic-management/protocol-selection/))，并且协议为TCP。默认的协议为TCP。

#### 举例

当集群中存在如下策略时：

```yaml
apiVersion: authentication.istio.io/v1alpha1
kind: Policy
metadata:
  name: secure-httpbin
  namespace: default
spec:
  targets:
    - name: httpbin
  origins:
    - jwt:
        issuer: "testing@secure.istio.io"
        jwksUri: "https://raw.githubusercontent.com/istio/istio-1.4/security/tools/jwt/samples/jwks.json"
```

target的service配置如下:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: default
  labels:
    app: httpbin
spec:
  ports:
  - name: svc-8080
    port: 8080
    targetPort: 80
    protocol: TCP
  selector:
    app: httpbin
```

在上面例子中，port `svc-8080`并没有遵守语法: `name: <http|https|http2>[-<suffix>]`。将会接收到如下消息：

```shell
Warn [IST0119] (Policy secure-httpbin.default) Authentication policy with JWT targets Service with invalid port specification (port: 8080, name: svc-8080, protocol: TCP, targetPort: 80).
```

JWT认证仅支持http，https或http2，修改Service 端口名，使其遵守 `<http|https|http2>[-<suffix>]`即可。

### MisplacedAnnotation

| Message Name | MisplacedAnnotation                                          |
| ------------ | ------------------------------------------------------------ |
| Message Code | IST0107                                                      |
| Description  | An Istio annotation is applied to the wrong kind of resource. |
| Level        | Warning                                                      |

当Istio的[annotation](https://istio.io/latest/docs/reference/config/annotations/)附加到一个无效的资源或错误的位置上时会出现该消息。例如，如果创建一个deployment，然后将annotation附加到该deployment，而不是pods上时就会出现该错误提示。

将annotation放到正确的位置即可修改该问题。

### MTLSPolicyConflict

| Message Name | MTLSPolicyConflict                                           |
| ------------ | ------------------------------------------------------------ |
| Message Code | IST0113                                                      |
| Description  | A DestinationRule and Policy are in conflict with regards to mTLS. |
| Level        | Error                                                        |

当一个destination rule资源和一个策略资源因为mutual TLS冲突时会出现该消息。当两个资源选择的TLS模式不匹配时就会出现这种情况。该冲突意味着，匹配到destination rule的流量将会被拒绝。

**该消息已经被废弃**，仅在使用alpha认证策略的服务网格中使用。(*了解该问题仍然可以避免配置错误*)

#### 哪些destination rules和策略与服务相关

为了有效解决mutual TLS的冲突，需要同时了解destination rule和策略是如何影响到一个服务的流量的。考虑一个`my-namespace`命名空间中的名为`my-service`的服务。为了确定应用到`my-service`上的是哪个策略对象，按顺序匹配以下资源：

1. `my-namespace`命名空间中的策略资源包含一个`target`，指定了`my-service`
2. `my-namespace`命名空间中的一个名为`default`的策略资源不包含一个`target`，意味着该策略适用于整个命名空间。
3. 名为`default`的网格策略资源

为了确定哪个destination rule应用到到达`my-service`的流量，首先要知道流量来自哪个命名空间。本例中，假设命名空间为`other-namespace`。destination rule按照如下方式进行匹配：

1. `other-namespace`命名空间中的destination rule会指定一个host来匹配 `my-service.my-namespace.svc.cluster.local`(可能会通过完整匹配或通配符匹配)。注意`exportTo`字段，该字段控制了配置资源的可见性，当目标资源与源服务在相同的命名空间时会被忽略(相同命名空间下的服务总是可见的)。

2. `my-namespace`命名空间中的destination rule会指定一个host来匹配 `my-service.my-namespace.svc.cluster.local`(可能会通过完整匹配或通配符匹配)。注意为了进行匹配，`exportTo`字段必须将该资源指定为公共的(即，值为`*`或不指定)。

3. 根命名空间(默认为`istio-system`)中的destination rule会匹配 `my-service.my-namespace.svc.cluster.local`。[`MeshConfig` ](https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig)资源中的`rootNamespace`属性会控制根命名空间。注意为了进行匹配，`exportTo`字段必须将该资源指定为公共的(即，值为`*`或不指定)。对`rootNamespace`的描述如下

   > The namespace to treat as the administrative root namespace for Istio configuration. When processing a leaf namespace Istio will search for declarations in that namespace first and if none are found it will search in the root namespace. **Any matching declaration found in the root namespace is processed as if it were declared in the leaf namespace.**

最后，注意在遵循这些规则时，Istio不会应用任何继承概念，它将使用符合指定条件的第一个资源。

#### 问题解决

查看如下输出：

```shell
Error [IST0113] (DestinationRule default-rule.istio-system) A DestinationRule
and Policy are in conflict with regards to mTLS for host
myhost.my-namespace.svc.cluster.local:8080. The DestinationRule
"istio-system/default-rule" specifies that mTLS must be true but the Policy
object "my-namespace/my-policy" specifies Plaintext.
```

可以看出两个资源是冲突的：

- 策略资源`my-namespace/my-policy`使用`Plaintext`来支持mutual TLS模式
- destination rule资源`istio-system/default-rule`,指定发送到host `myhost.my-namespace.svc.cluster.local:8080`的流量需要启用mutual TLS 

可以使用下面的方式之一来修复该问题：

- 修改策略资源`my-namespace/my-policy`来启用mutual TLS作为认证模式。
- 修改destination rule `istio-system/default-rule`，通过移除`ISTIO_MUTUAL`来禁用mutual TLS。注意`default-rule`位于`istio-system`命名空间，即默认的根命名空间中，意味着该destination rule会影响到网格中过的所有其他服务。
- 在与服务相同的命名空间(`my-namespace`)中添加一个新的destination rule，该destination rule不指定流量策略mutual TLS。由于该规则位于与服务相同的命名空间中，它不会覆盖全局destination rule `istio-system/default-rule`.

### MultipleSidecarsWithoutWorkloadSelectors

| Message Name | MultipleSidecarsWithoutWorkloadSelectors                     |
| ------------ | ------------------------------------------------------------ |
| Message Code | IST0111                                                      |
| Description  | More than one sidecar resource in a namespace has no workload selector |
| Level        | Error                                                        |

当一个命名空间中的多个sidecar资源没有定义任何[负载选择器](https://istio.io/latest/docs/reference/config/networking/sidecar/#WorkloadSelector)时会出现该消息，导致不可预知的行为。更多参见[Sidecar](https://istio.io/latest/docs/reference/config/networking/sidecar/)。

一个命名空间仅允许一个sidecar资源不指定负载选择器。

### NamespaceNotInjected

| Message Name | NamespaceNotInjected                            |
| ------------ | ----------------------------------------------- |
| Message Code | IST0102                                         |
| Description  | A namespace is not enabled for Istio injection. |
| Level        | Warning                                         |

当一个命名空间没有annotation(如`sidecar.istio.io/inject`.)指明该命名空间是否需要自动注入sidecar时会出现该提示。错误信息如下：

```shell
Warn [IST0102] (Namespace default) The namespace is not enabled for Istio
injection. Run 'kubectl label namespace default istio-injection=enabled' to
enable it, or 'kubectl label namespace default istio-injection=disabled' to
explicitly mark it as not needing injection Error: Analyzer found issues.
```

可以通过给命名空间打标签来解决该问题，明确声明是否需要启用sidecar自动注入：

```shell
$ kubectl label namespace <namespace-name> istio-injection=enabled
```

强烈建议明确指定sidecar注入行为。忘记注释命名空间是导致错误的常见原因。

### SchemaValidationError

| Message Name | SchemaValidationError                       |
| ------------ | ------------------------------------------- |
| Message Code | IST0106                                     |
| Description  | The resource has a schema validation error. |
| Level        | Error                                       |

当Istio的配置没有通过模式验证时会出现该错误。如：

```shell
Error [IST0106] (VirtualService ratings-bogus-weight-default.default) Schema validation error: percentage 888 is not in range 0..100
```

Istio配置为：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-bogus-weight-default
  namespace: default
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
      weight: 999
    - destination:
        host: ratings
        subset: v2
      weight: 888
```

本例中可以看到`weight`元素为一个无效的值。可以通过[消息的details字段](https://istio.io/latest/docs/reference/config/analysis/message-format/)来检查哪个元素或值没有符合模式要求，然后进行修正。

### VirtualServiceDestinationPortSelectorRequired

| Message Name | VirtualServiceDestinationPortSelectorRequired                |
| ------------ | ------------------------------------------------------------ |
| Message Code | IST0112                                                      |
| Description  | A VirtualService routes to a service with more than one port exposed, but does not specify which to use. |
| Level        | Error                                                        |

当一个virtual service路由到暴露多个port的服务，且没有指定使用哪个端口时会出现该错误。这种模糊性可能导致不确定的行为。

可以在virtual service [Destination](https://istio.io/latest/docs/reference/config/networking/virtual-service/#Destination)中增加port字段来解决该问题。

### UnknownAnnotation

| Message Name | UnknownAnnotation                                            |
| ------------ | ------------------------------------------------------------ |
| Message Code | IST0108                                                      |
| Description  | An Istio annotation is not recognized for any kind of resource |
| Level        | Warning                                                      |

当将格式为`*.istio.io`的无法识别的注释附加到名称空间时，会出现此消息。Istio仅能识别特定的[annotation名称](https://istio.io/latest/docs/reference/config/annotations/)。

### ReferencedResourceNotFound

| Message Name | ReferencedResourceNotFound                  |
| ------------ | ------------------------------------------- |
| Message Code | IST0101                                     |
| Description  | A resource being referenced does not exist. |
| Level        | Error                                       |

当Istio资源相关的资源不存在时会出现该错误。当Istio尝试查找引用的资源但无法找到时，将导致错误。错误信息如：

```
Error [IST0101] (VirtualService httpbin.default) Referenced gateway not found: "httpbin-gateway-bogus"
```

本例中，VirtualService引用了一个不存在的网关。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http2
      protocol: HTTP2
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway-bogus #  Should have been "httpbin-gateway"
  http:
  - route:
    - destination:
        host: httpbin-gateway
```

为了解决该问题，查看detaild错误消息中的资源类型，然后修正Istio配置即可。

### PortNameIsNotUnderNamingConvention

| Message Name | PortNameIsNotUnderNamingConvention                           |
| ------------ | ------------------------------------------------------------ |
| Message Code | IST0118                                                      |
| Description  | Port name is not under naming convention. Protocol detection is applied to the port. |
| Level        | Info                                                         |

当端口不遵守[Istio服务端口命名规范](https://istio.io/latest/docs/ops/deployment/requirements/)或端口未命名时会出现该错误。错误信息如：

```shell
Info [IST0118] (Service httpbin.default) Port name foo-http (port: 80, targetPort: 80) doesn't follow the naming convention of Istio port.
```

对应的Service为：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
spec:
  ports:
  - name: foo-http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin
```

可以通过将port `foo-http`按照语法`name: <protocol>[-<suffix>]`修改即可。

#### 问题解决

- 如果知道服务端口的协议，使用 `<protocol>[-<suffix>]` 格式重命名即可
- 如果不知道服务端口的协议，则需要[从Prometheus请求metrics](https://istio.io/latest/docs/tasks/observability/metrics/querying-metrics/)来获取
  - 执行请求`istio_requests_total`
  - 如果有输出，可以在metrics的`request_protocol`字段中看到使用的协议
  - 如果没有输出，则可以保留端口不变。

### PodMissingProxy

| Message Name | PodMissingProxy                   |
| ------------ | --------------------------------- |
| Message Code | IST0103                           |
| Description  | A pod is missing the Istio proxy. |
| Level        | Warning                           |

当没有sidecar或sidecar不正常时会出现该错误。

通常是因为启用了自动注入，但后续没有重启pod导致sidecar不存在。

可以使用如下命令修复：

```shell
$ kubectl rollout restart deployment
```

