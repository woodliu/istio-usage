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

上面部署的Jaeger对应的k8s service名为`tracing`，查看该service，可以看到容器和service暴露的端口均为16686。

```shell
# oc describe svc tracing
Name:              tracing
Namespace:         istio-system
Labels:            app=jaeger
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"jaeger"},"name":"tracing","namespace":"istio-system"},"s...
Selector:          app=jaeger
Type:              ClusterIP
IP:                10.84.179.208
Port:              http-query  80/TCP
TargetPort:        16686/TCP
Endpoints:         10.80.2.226:16686
Session Affinity:  None
Events:            <none>
```

在openshift上创建一个route(ingress)，即可访问该Jaeger。

### 使用Boofinfo生成traces

1. 在浏览器上多次访问`http://$GATEWAY_URL/productpage`来生成跟踪信息

   为了查看跟踪数据，需要服务发送请求。请求的数量取决于istio的采样率，采样率是在安装istio的时候设置的，默认为1%，即在看到第一个trace之前需要至少发送100个请求。使用如下命令向productpage发送100个请求：

   ```shell
   $ for i in $(seq 1 100); do curl -s -o /dev/null "http://$GATEWAY_URL/productpage"; done
   ```

2. 在dashboard的左边，在**Service**下拉列表中选择`productpage.default`，并点击**Find Traces**，可以看到生成了两条span

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200902140220198-57119483.png)

3. 点击最近时间内到`/productpage`的请求的细节

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200902140635859-538449695.png)

   此外还可以点击右上角的**Alternate Views**切换视图，可以更直观地看到请求的访问情况：

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200902141306718-1690436740.png)

4. trace由一组span构成，每个span对应一个Bookinfo服务，在执行到`/productpage`的请求或内部Istio组件(如`istio-ingressgateway`)时生成。

## Kiali

本节展示如何可视化istio网格的方方面面。

本节将安装Kiali插件并使用基于Web的图形用户界面查看网格和Istio配置对象的服务图，最后，使用Kiali Developer API以consumable  JSON的形式生成图形数据。

> 本任务并没有涵盖Kiali的所有特性，更多参见[Kiali website](http://kiali.io/documentation/latest/features/)。

本节将使用Boofinfo应用。

### 部署

Kiali依赖Prometheus，因此首先要部署Prometheus。使用如下命令快速部署一个用于演示的Prometheus：

```shell
$ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.7/samples/addons/prometheus.yaml
```

使用如下方式快速部署一个用于演示的Kiali。如果要用于生产环境，建议参考[quick start guide](https://kiali.io/documentation/latest/quick-start) 和[customizable installation methods](https://kiali.io/documentation/latest/installation-guide)。

```shell
$ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.7/samples/addons/kiali.yaml
```

> 注：由于Kiali依赖Prometheus，因此需要将Kiali和Prometheus关联起来。修改上面kiali.yaml中的如下字段，然后部署即可：
>
>         custom_metrics_url: http://prometheus.istio-system.svc.cluster.local:9090
>         url: http://prometheus.istio-system.svc.cluster.local:9090

同样地，为名为`kiali`的k8s service创建一个`route`即可访问，端口为20001。

### 生成服务图

1. 使用如下命令检查kiali的service

   ```shell
   $ kubectl -n istio-system get svc kiali
   ```

2. 可以使用如下三种方式向网格发送流量

   - 通过浏览器访问`http://$GATEWAY_URL/productpage`

   - 使用如下`curl`命令

     ```shell
     $ curl http://$GATEWAY_URL/productpage
     ```

   - 使用`watch`持续访问

     ```shell
     $ watch -n 1 curl -o /dev/null -s -w %{http_code} $GATEWAY_URL/productpage
     ```

3. 在**Overview**页面浏览网格概况，该页面展示了网格中所有命名空间下的服务。可以看到default命名空间下有流量(Bookinfo应用安装到的default命名空间下)

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200902150532409-2091806055.png)

4. 为了查看一个命名空间的图表，在Boofinfo所在的命名空间中查看`bookinfo`的图表如下，三角形表示service，方形表示app

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200902153528382-123854955.png)

5. 可以在`Display`中选择不同的Edge来展示metrics的概况，如上图显示了`Response Time`。unknow是因为没有走istio的gateway，而使用了openshift的route

6. 为了显示服务网格中的不同类型的图表，在**Graph Type**下拉框中可以选择如下图标类型：**App**, **Versioned App**, **Workload**, **Service**

   - **App**图表类型会将所有版本的app聚合为一个图表节点。下图可以看到，它使用一个**reviews** 节点来表示三个版本的reviews app

     ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200902154203683-1218800621.png)

   - **Versioned App**图表类型使用一个node展示了不同版本的app，同时对特定app的不同版本进行分组。下图展示了**reviews** 组，其包含3个小的节点，三个节点表示三个版本的reviews app

     ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200902154852134-1507554673.png)

   - **Workload** 图表类型使用节点展示了服务网格中的每个负载。该图形类型不要求使用`app` 和`version`标签，因此如果选择不在组件上使用这些标签时，就可以使用该图表类型。

     ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200902155313503-75060219.png)

   - **Service** 图表类型使用节点展示网格中的每个服务，但排除所有应用程序和工作负载

     ![image-20200902155441754](C:\Users\liuchanglin\AppData\Roaming\Typora\typora-user-images\image-20200902155441754.png)

### 检查Istio配置

为了查看Istio配置的详细配置，可以在左边菜单栏中点击 **Applications**, **Workloads**和**Services**。下图展示了Bookinfo应用的信息

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200902170259187-1847515499.png)

### 创建加权路由

可以使用Kiali加权路由向导来定义指定百分比的请求流量来路由到两个或更多负载。

1. 将bookinfo图表切换为**Versioned app graph**

   - 确保在**Display**下拉框中选择了**Requests percentage**来查看路由到每个负载的流量百分比

   - 确保在**Display**下拉框中选择了**Service Nodes**来显示service节点

     ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200902171558299-342161384.png)

2. 通过点击`reviews`服务(三角形)关注`bookinfo`图表中的`reviews`服务

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903092114045-922301504.png)

3. 在右上角的下拉框中选择**Show Details**进入ratings的service设置

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903092209125-1452874994.png)

4. 在**Action**下拉框中，选择**Create Weighted Routing**来访问加权路由向导

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903092323000-703073457.png)

5. 通过拖动滑块来调整到负载的流量百分比。将`reviews-v1`设置为30%，`reviews-v2`设置为0%，将`reviews-v3`设置为70%。

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903092422243-1664496989.png)

6. 点击**Create**创建该路由。对应会在istio中创建用于流量划分的一个VirtualService和一个DestinationRule：

   ```yaml
   apiVersion: networking.istio.io/v1beta1
   kind: VirtualService
   metadata:
     labels:
       kiali_wizard: weighted_routing
     name: reviews
     namespace: default
   spec:
     hosts:
     - reviews.default.svc.cluster.local
     http:
     - route:
       - destination:
           host: reviews.default.svc.cluster.local
           subset: v1
         weight: 30
       - destination:
           host: reviews.default.svc.cluster.local
           subset: v2
         weight: 0
       - destination:
           host: reviews.default.svc.cluster.local
           subset: v3
         weight: 70
   ```

   ```yaml
   kind: DestinationRule
   metadata:
     labels:
       kiali_wizard: weighted_routing
     name: reviews
     namespace: default
   spec:
     host: reviews.default.svc.cluster.local
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

7. 点击左边导航栏的**Graph**按钮返回到`bookinfo`图表

8. 向bookinfo应用发送请求。例如使用如下命令每秒发送一个请求：

   ```shell
   $ watch -n 1 curl -o /dev/null -s -w %{http_code} $GATEWAY_URL/productpage
   ```

9. 一段时间后，可以看到流量百分比会体现为新的流量路由，即v1版本接收到30%的流量，v2版本没有流量，v3版本接收到70%的流量

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903093952583-1712584017.png)

### 验证Istio配置

Kiali可以验证Istio的资源来确保它们遵循正确的约定和语义。根据配置错误的严重性，可以将Istio资源配置中检测到的任何问题标记为错误或警告。

下面将尝试对服务端口名称进行无效性修改来查看Kiali如何报告错误：

1. 将`details`服务的端口名称从`http`修改为`foo`

   ```shell
   # kubectl patch service details -n default --type json -p '[{"op":"replace","path":"/spec/ports/0/name", "value":"foo"}]'
   ```

2. 点击左侧导航栏的**Services** 转到服务列表

3. 选择`bookinfo`所在的命名空间

4. 注意在`details`一行中显示的错误图标

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903095134989-835181854.png)

5. 点击**Name**一栏对应的`details`来查看详细信息

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903095532765-1273194629.png)

6. 将端口名称切换回`http`，可以看到`bookinfo`又变的正常了

   ```shell
   # kubectl patch service details -n default --type json -p '[{"op":"replace","path":"/spec/ports/0/name", "value":"http"}]'
   ```

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903095746223-530273971.png)

### 查看和修改Istio的配置YAML

Kiali提供了一个YAML编辑器，可以用于查看和修改Istio的配置资源。YAML编辑器也提供了校验配置的功能。

1. 创建一条Bookinfo的destination rules

   ```shell
   $ kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
   ```

2. 点击导航栏左边的`Istio Config`转到istio的配置列表

3. 选择`bookinfo`所在的命名空间

4. 注意在右上角有提示配置的错误和告警信息

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903100625478-1720279913.png)

5. 将鼠标悬停在`details`行**Configuration** 列中的错误图标上，可以查看其他消息。

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903100414057-1870543494.png)

6. 单击**Name**列中的`details`链接，导航到`details`的destination rule视图。

7. 可以看到有校验未通过的错误提示

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903100955144-1641983635.png)

8. 点击`YAML`查看Istio的destination rule规则，Kiali用颜色高亮除了未通过有效性校验的行

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903101041230-1667799364.png)

9. 将鼠标悬停在红色图标上可以查看工具提示消息，该消息提示触发错误的验证检查。

   ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200903101450101-544298644.png)

10. 删除创建的bookinfo的destination rule，进行环境恢复

    ```shell
    $ kubectl delete -f samples/bookinfo/networking/destination-rule-all.yaml
    ```

> 在官方文档中，上述YAML中是存在黄色的告警图标，但实际部署时并没有出现。

### 关于Kiali Developer API

为了生成表示图表和其他指标，健康，以及配置信息的JSON文件，可以参考[Kiali Developer API](https://www.kiali.io/api)。例如，可以通过访问`$KIALI_URL/api/namespaces/graph?namespaces=default&graphType=app`来获取使用`app` 图表类型的JSON表示格式。

Kiali Developer API建立在Prometheus查询之上，并取决于标准的Istio metric配置，它还会执行kubernetes API调用来获取服务的其他详细信息。为了更好地使用Kiali，请在自己的应用组件中使用metadata labels `app`和`version`。

> **请注意，Kiali Developer API可能在不同版本间发送变更，且不保证向后兼容。**

### 其他特性

Kiali还有其他丰富的特性，如[与Jaeger跟踪的集成](https://kiali.io/documentation/latest/features/#_detail_traces)。

更多细节和特性，参见Kiali[官方文档](https://kiali.io/documentation/latest/features/)。

### 卸载

```shell
$ kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-1.7/samples/addons/kiali.yaml
```

