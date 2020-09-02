# Istio可观测性

Istio的可观测性包括metrics，日志，分布式链路跟踪以及可视化展示。下面主要介绍如何在istio中部署基于Prometheus的metrics监控，基于jaeger的链路跟踪和基于kiali的可视化界面。

[TOC]

## Prometheus

### 配置说明

在istio网格中，每个组件都会暴露一个提供metrics的endpoint。Prometheus可以从这些endpoints上抓取metrics(通过[Prometheus配置文件](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)来设置scraping，端口以及TLS等)。

为了采集整个网格的metrics，需要配置Prometheus scraping组件：

1. 控制面(`istiod` deployment)
2. ingress和egress网关
3. Envoy sidecar
4. 用户应用(如果该应用也可以暴露Prometheus metrics的话)

为了简化metrics的配置，istio提供了如下两种操作模式：

#### Option 1:合并metrics

为了简化配置，istio可以通过`prometheus.io` annotations来控制所有scraping。这使Istio的scraping可以使用标准配置(例如[Helm `stable/prometheus`](https://github.com/helm/charts/tree/master/stable/prometheus) charts提供的配置)来做到开箱即用。

该选项默认是启用的，但可以在安装期间通过传入`--set meshConfig.enablePrometheusMerge=false`来禁用该功能。当启用时，会在所有的控制面pod上添加`prometheus.io` annotations来配置scraping。如果已经存在这些annotations，则会被覆盖。通过该选项，Envoy sidecar会将istio的metrics和应用的metrics进行合并，合并后的metrics暴露地址为：`/stats/prometheus:15020`.

> 通过kubect describe pod命令可以查看pod的annotation，需要注意的是控制面和数据暴露的端口是不同的。数据面暴露的端口为15020，如下：
>
> ```yaml
>       prometheus.io/path: /stats/prometheus
>       prometheus.io/port: 15020
>       prometheus.io/scrape: true
> ```
> 但istiod的端口为15014，ingress-gateway和egress-gateway的端口为15090。总体来说与官方[端口描述](https://istio.io/latest/docs/ops/deployment/requirements/#ports-used-by-istio)一致。

该选项会以明文方式暴露所有的metrics。

下面特性可能并不适用于所有场景：

- 在抓取metrcis时启用TLS
- 应用暴露的metrics与istio的metrics的名称相同。例如，应用暴露了一个名为`istio_requests_total`的metric，可能是因为应用本身也运行了Envoy
- 部署的Prometheus没有基于标准的`prometheus.io` annotation抓取metrics。

如果有必要，则可以在pod上添加`prometheus.istio.io/merge-metrics: "false"` annotation来禁用metrics合并功能。

#### Option 2:自定义抓取metrics配置

内置的demo profile会安装Prometheus，并包含了所有必要的scraping配置。可以在使用istioctl部署istio时传入参数`--set values.prometheus.enabled=true`来部署Prometheus。

但内置的Prometheus缺少高级自定义配置功能，如认证的持久化等，导致其不大合适在生产环境中使用。为了使用已经存在的Prometheus，需要在Prometheus配置中添加scraping配置[`prometheus/configmap.yaml`](https://raw.githubusercontent.com/istio/istio/release-1.7/manifests/charts/istio-telemetry/prometheus/templates/configmap.yaml)。

该配置会添加抓取控制面以及所有Envoy sidecar的metrcis的功能。此外，还配置了一个job，用来抓取添加了`prometheus.io` annotation的所有数据面pod的应用metrics。

##### TLS设置

控制面，网关和Envoy sidecar的metrics都以明文方式暴露。然而，应用metrics会遵循istio对负载的配置。例如如果启用了[Strict mTLS](https://istio.io/latest/docs/tasks/security/authentication/authn-policy/#globally-enabling-istio-mutual-tls-in-strict-mode)，那么Prometheus需要使用istio证书来配置scrape

对于自建的Prometheus，参考[为没有sidecar的应用程序提供证书和密钥](https://istio.io/latest/blog/2020/proxy-cert/)一文来为Prometheus提供证书，并添加TLS scraping配置。

### 总结

istio的metrics分为两类：istio自身的metrics和应用产生的metrics，前者以明文方式暴露，后者遵循对应负载的配置，即，如果负载启用了TLS，则Prometheus也需要配置TLS才能获取应用的metrics。

istio的metrics主要通过[kubernetes_sd_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)服务发现进行采集。其中`prometheus.io/path`和`prometheus.io/port` annotation分别对应metrics `meta_kubernetes_pod_annotation_prometheus_io_scrape`和`meta_kubernetes_pod_annotation_prometheus_io_path` 。简单配置如下：

```yaml
- job_name: 'kubernetes-pods'

        kubernetes_sd_configs:
        - role: pod

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
```

更多配置可以参见官方提供的[配置](https://raw.githubusercontent.com/istio/istio/release-1.7/manifests/charts/istio-telemetry/prometheus/templates/configmap.yaml)。

## Jaeger

### 概述

分布式跟踪使用户可以通过分布在多个服务中的网格跟踪请求。通过这种方式可以了解请求延迟，并通过可视化实现序列化和并行。

Istio利用[Envoy的分布式跟踪](https://www.envoyproxy.io/docs/envoy/v1.12.0/intro/arch_overview/observability/tracing)特性提供开箱即用的跟踪集成。特别地，istio提供了选项来安装不同的跟踪后端，以及配置代理来发送这些后端自动发送跟踪span。查看[Zipkin](https://istio.io/latest/docs/tasks/observability/distributed-tracing/zipkin/), [Jaeger](https://istio.io/latest/docs/tasks/observability/distributed-tracing/jaeger/)和[Lightstep](https://istio.io/latest/docs/tasks/observability/distributed-tracing/lightstep/)来了解istio如何与这些跟踪系统共同工作。

### 跟踪上下文的传递

虽然istio代理可以自动发送span，但它们需要一些提示来将整个跟踪联系在一起。应用需要传递合适的HTTP首部，这样当代理发送span信息时，这些span可以正确地关联到单个跟踪中。

为了实现上述目的，应用需要在传入请求中收集并传递如下首部到任何传出请求中：

- `x-request-id`
- `x-b3-traceid`
- `x-b3-spanid`
- `x-b3-parentspanid`
- `x-b3-sampled`
- `x-b3-flags`
- `x-ot-span-context`

此外，基于[OpenCensus](https://opencensus.io/) (如Stackdriver)的跟踪集成需要传递下面首部：

- `x-cloud-trace-context`
- `traceparent`
- `grpc-trace-bin`

如果查看istio的`productpage`例子的Python源码，可以看到应用会使用[OpenTracing](https://opentracing.io/)库从HTTP请求中抽取需要的首部：

```python
def getForwardHeaders(request):
    headers = {}

    # x-b3-*** headers can be populated using the opentracing span
    span = get_current_span()
    carrier = {}
    tracer.inject(
        span_context=span.context,
        format=Format.HTTP_HEADERS,
        carrier=carrier)

    headers.update(carrier)

    # ...

    incoming_headers = ['x-request-id', 'x-datadog-trace-id', 'x-datadog-parent-id', 'x-datadog-sampled']

    # ...

    for ihdr in incoming_headers:
        val = request.headers.get(ihdr)
        if val is not None:
            headers[ihdr] = val

    return headers
```

`reviews`应用会使用`requestHeaders`做类似的事情：

```java
@GET
@Path("/reviews/{productId}")
public Response bookReviewsById(@PathParam("productId") int productId, @Context HttpHeaders requestHeaders) {

  // ...

  if (ratings_enabled) {
    JsonObject ratingsResponse = getRatings(Integer.toString(productId), requestHeaders);
```

当在应用程序中执行下游调用时，需要包含这些首部。

### 使用Jaeger

本例将使用[Bookinfo](https://istio.io/latest/docs/examples/bookinfo/)。

### 部署

1. 使用如下方式快速部署一个用于演示的Jaeger。当然也可以参考[Jaeger](https://www.jaegertracing.io/)官方文件进行自定义部署。

   ```shell
   $ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.7/samples/addons/jaeger.yaml
   ```

2. 可以参考[此处](https://istio.io/latest/docs/tasks/observability/distributed-tracing/configurability/#customizing-trace-sampling)修改采样率

### 访问Jaeger











## Kiali









