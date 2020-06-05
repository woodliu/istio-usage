# Istio的运维-诊断工具

[TOC]

## 使用istioctl命令行工具

可以通过[日志](https://istio.io/docs/ops/diagnostic-tools/component-logging/)或[Introspection](https://istio.io/docs/ops/diagnostic-tools/controlz/)检查各个组件，如果不足以支持问题定位，可以参考如下操作：

[istioctl](https://istio.io/docs/reference/commands/istioctl)是一个可以用于调试和诊断istio服务网格的工具。Istio项目为Bash和ZSH运行下的`istioctl`提供了自动补全功能。

建议安装对应istio版本的`istioctl`。

### istioctl自动补全

- 将如下内容添加到 `~/.bash_profile` 文件中

  ```shell
  [[ -r "/usr/local/etc/profile.d/bash_completion.sh" ]] && . "/usr/local/etc/profile.d/bash_completion.sh"
  ```

- 使用bash时，将在`tools`命令中的`istioctl.bash`文件拷贝到`$HOME`目录下，然后执行如下操作即可

  ```shell
  $ source ~/istioctl.bash
  ```

### 查看网格的状态

可以使用`istioctl proxy-status`或`istioctl ps`命令查看网格的状态。如果输出结果中缺少某个代理，说明该代理当前没有连接到Pilot实例，因此无法接收到任何配置。如果状态为`stale`，表示当前存在网络故障，或Pilot需要扩容。

### 获取代理配置

可以使用`istioctl proxy-config`或`istioctl pc`检索代理配置信息。

例如，使用如下方式可以检索特定pod中的Envoy实例的集群配置信息。

```shell
$ istioctl proxy-config cluster <pod-name> [flags]
```

使用如下方式可以检索特定pod中的Envoy实例的bootstrap配置信息。

```shell
$ istioctl proxy-config bootstrap <pod-name> [flags]
```

使用如下方式可以检索特定pod中的Envoy实例的listener(监听器)配置信息。

```shell
$ istioctl proxy-config listener <pod-name> [flags]
```

使用如下方式可以检索特定pod中的Envoy实例的route(路由)配置信息。

```shell
$ istioctl proxy-config route <pod-name> [flags]
```

使用如下方式可以检索特定pod中的Envoy实例的endpoint (后端)配置信息。

```shell
$ istioctl proxy-config endpoints <pod-name> [flags]
```

更多参见下一节[调试Envoy和istiod](https://istio.io/docs/ops/diagnostic-tools/proxy-cmd/)

## 调试Envoy和istiod

istio提供了两个非常有用的命令来诊断流量管理配置问题：[proxy-status](https://istio.io/docs/reference/commands/istioctl/#istioctl-proxy-status) 和[proxy-config](https://istio.io/docs/reference/commands/istioctl/#istioctl-proxy-config)。`proxy-status`可以获取网格的概述并确定导致问题的代理。`proxy-config`可以检查Envoy配置并诊断该问题。

如果要尝试如下命令，可以：

- [安装Bookinfo](https://istio.io/docs/examples/bookinfo/#deploying-the-application) 
- 使用kubernetes集群中部署类似应用

### 获取网格概况

通过`proxy-status`命令可以查看网格的概况，了解是否有sidecar无法接收配置或无法保持同步。

如果某个代理没有出现在输出列表中，则说明该代理没有连接到istiod实例，因此也无法接收任何配置信息。状态信息如下：

- `SYNCED`：表示Envoy确认了istiod发过来的配置
- `NOT SENT`：表示istiod还没有发送配置到Envoy。通常时因为istiod当前没有需要发送的配置信息
- `STALE`：表示istiod发送了一个更新到Envoy，但没有接收到确认。通常表示Envoy和istiod之间的网络出现了问题，或istio本身出现了bug。

```shell
$ istioctl ps
NAME                                                   CDS        LDS                            EDS        RDS          PILOT                       VERSION
details-v1-78d78fbddf-psnmk.default                    SYNCED     SYNCED                         SYNCED     SYNCED       istiod-788cf6c878-4pq5g     1.6.0
istio-ingressgateway-569669bb67-dsd5h.istio-system     SYNCED     SYNCED                         SYNCED     NOT SENT     istiod-788cf6c878-4pq5g     1.6.0
productpage-v1-85b9bf9cd7-d8hm8.default                SYNCED     SYNCED                         SYNCED     SYNCED       istiod-788cf6c878-4pq5g     1.6.0
prometheus-79878ff5fd-tjdxx.istio-system               SYNCED     SYNCED                         SYNCED     SYNCED       istiod-788cf6c878-4pq5g     1.6.0
ratings-v1-6c9dbf6b45-xlf2q.default                    SYNCED     SYNCED                         SYNCED     SYNCED       istiod-788cf6c878-4pq5g     1.6.0
reviews-v1-564b97f875-q5l9r.default                    SYNCED     SYNCED                         SYNCED     SYNCED       istiod-788cf6c878-4pq5g     1.6.0
reviews-v2-568c7c9d8f-vcd94.default                    SYNCED     SYNCED                         SYNCED     SYNCED       istiod-788cf6c878-4pq5g     1.6.0
reviews-v3-67b4988599-psllq.default                    SYNCED     SYNCED                         SYNCED     SYNCED       istiod-788cf6c878-4pq5g     1.6.0
sleep-78484c89dd-fmxbc.default                         SYNCED     SYNCED                         SYNCED     SYNCED       istiod-788cf6c878-4pq5g     1.6.0
```

### 检索Envoy和istiod的差异

`proxy-status`命令配合proxy ID可以检索Envoy加载的配置和istiod发送的配置之间的差异，通过这种方式可以确定哪部分内容没有被同步，并确定可能存在的问题点。

下面的例子可以看到ingressgateway的listeners和routers配置都与istiod发过来的配置匹配，但clusters不匹配。

```shell
$ istioctl proxy-status details-v1-6dcc6fbb9d-wsjz4.default
--- Istiod Clusters
+++ Envoy Clusters
@@ -374,36 +374,14 @@
             "edsClusterConfig": {
                "edsConfig": {
                   "ads": {

                   }
                },
                "serviceName": "outbound|443||public-cr0bdc785ce3f14722918080a97e1f26be-alb1.kube-system.svc.cluster.local"
-            },
-            "connectTimeout": "1.000s",
-            "circuitBreakers": {
-               "thresholds": [
-                  {
-
-                  }
-               ]
-            }
-         }
-      },
-      {
-         "cluster": {
-            "name": "outbound|53||kube-dns.kube-system.svc.cluster.local",
-            "type": "EDS",
-            "edsClusterConfig": {
-               "edsConfig": {
-                  "ads": {
-
-                  }
-               },
-               "serviceName": "outbound|53||kube-dns.kube-system.svc.cluster.local"
             },
             "connectTimeout": "1.000s",
             "circuitBreakers": {
                "thresholds": [
                   {

                   }

Listeners Match
Routes Match
```

### 深入探究Envoy配置

`proxy-config`命令可以查看一个Envoy实例的配置，用于定位无法通过查看istio配置和用户资源发现的问题。例如，使用如下命令可以获取特定pod的clusters，listeners或路由概要。*注：首先通过istioctl ps查看出不匹配的内容，然后使用istioctl pc查看具体的不匹配的信息。*

```shell
$ istioctl proxy-config cluster -n istio-system istio-ingressgateway-7d6874b48f-qxhn5
SERVICE FQDN                                                               PORT      SUBSET     DIRECTION     TYPE
BlackHoleCluster                                                           -         -          -             STATIC
agent                                                                      -         -          -             STATIC
details.default.svc.cluster.local                                          9080      -          outbound      EDS
istio-ingressgateway.istio-system.svc.cluster.local                        80        -          outbound      EDS
istio-ingressgateway.istio-system.svc.cluster.local                        443       -          outbound      EDS
istio-ingressgateway.istio-system.svc.cluster.local                        15021     -          outbound      EDS
istio-ingressgateway.istio-system.svc.cluster.local                        15443     -          outbound      EDS
istiod.istio-system.svc.cluster.local                                      443       -          outbound      EDS
istiod.istio-system.svc.cluster.local                                      853       -          outbound      EDS
istiod.istio-system.svc.cluster.local                                      15010     -          outbound      EDS
istiod.istio-system.svc.cluster.local                                      15012     -          outbound      EDS
istiod.istio-system.svc.cluster.local                                      15014     -          outbound      EDS
kube-dns.kube-system.svc.cluster.local                                     53        -          outbound      EDS
kube-dns.kube-system.svc.cluster.local                                     9153      -          outbound      EDS
kubernetes.default.svc.cluster.local                                       443       -          outbound      EDS
...
productpage.default.svc.cluster.local                                      9080      -          outbound      EDS
prometheus.istio-system.svc.cluster.local                                  9090      -          outbound      EDS
prometheus_stats                                                           -         -          -             STATIC
ratings.default.svc.cluster.local                                          9080      -          outbound      EDS
reviews.default.svc.cluster.local                                          9080      -          outbound      EDS
sds-grpc                                                                   -         -          -             STATIC
xds-grpc                                                                   -         -          -             STRICT_DNS
zipkin           
```

为了调试Envoy，首先需要理解Envoy的clusters/listeners/routes/endpoints，以及它们之间是如何交互的。下面将使用带有`-o json`和筛选标志的`proxy config`命令来跟踪Envoy，因为它决定在哪里将请求从`productpage` pod发送到`reviews` pod的 `reviews:9080`

1. 如果请求了一个pod的listener概要，可以看到istio生成了如下listeners：

   -  一个`0.0.0.0:15006`的listener，用于接收到pod的入站流量；以及一个 `0.0.0.0:15001`的listener，用于接收所有到pod的出站流量，然后将请求交给一个 virtual listener。
   - 每个kubernetes service IP都对应一个virtual listener，非HTTP的listener用于出站的TCP/HTTPS流量
   - pod IP中的virtual listener暴露了接收入站流量的端口
   - 0.0.0.0的HTTP类型的virtual listener，用于出站的HTTP流量

   Istio使用的[端口](https://istio.io/docs/ops/deployment/requirements/#ports-used-by-istio)信息如下：

   | Port  | Protocol | Used by                     | Description                             |
   | ----- | -------- | --------------------------- | --------------------------------------- |
   | 15000 | TCP      | Envoy                       | Envoy admin port (commands/diagnostics) |
   | 15001 | TCP      | Envoy √                     | Envoy Outbound                          |
   | 15006 | TCP      | Envoy √                     | Envoy Inbound                           |
   | 15020 | HTTP     | Envoy                       | Istio agent Prometheus telemetry        |
   | 15021 | HTTP     | Envoy                       | Health checks                           |
   | 15090 | HTTP     | Envoy                       | Envoy Prometheus telemetry              |
   | 15010 | GRPC     | Istiod                      | XDS and CA services (plaintext)         |
   | 15012 | GRPC     | Istiod                      | XDS and CA services (TLS)               |
   | 8080  | HTTP     | Istiod                      | Debug interface                         |
   | 443   | HTTPS    | Istiod                      | Webhooks                                |
   | 15014 | HTTP     | Mixer, Istiod               | Control plane monitoring                |
   | 15443 | TLS      | Ingress and Egress Gateways | SNI                                     |
   | 9090  | HTTP     | Prometheus                  | Prometheus                              |
   | 42422 | TCP      | Mixer                       | Telemetry - Prometheus                  |
   | 15004 | HTTP     | Mixer, Pilot                | Policy/Telemetry - `mTLS`               |
   | 9091  | HTTP     | Mixer                       | Policy/Telemetry                        |

   可以看到`TYPE`字段是没有`HTTPS`的，`HTTPS`作为`TCP`类型

   ```shell
   $ istioctl proxy-config listeners productpage-v1-6c886ff494-7vxhs
   ADDRESS            PORT      TYPE
   10.96.0.10         53        TCP        <--+
   10.109.221.105     15012     TCP           |
   10.96.121.204      15443     TCP           | Receives outbound non-HTTP traffic for relevant IP:PORT pair from listener `0.0.0.0_15001`
   10.96.0.1          443       TCP           |
   10.109.221.105     443       TCP           |
   10.96.121.204      443       TCP        <--+
   0.0.0.0            9090      HTTP+TCP   <--+
   10.96.0.10         9153      HTTP+TCP      |
   0.0.0.0            15010     HTTP+TCP      |
   0.0.0.0            80        HTTP+TCP      | Receives outbound HTTP+TCP traffic for relevant port from listener `0.0.0.0_15001`
   0.0.0.0            15014     HTTP+TCP      |
   10.109.221.105     853       HTTP+TCP      |
   10.96.121.204      15021     HTTP+TCP   <--+
   0.0.0.0            9080      HTTP+TCP   // Receives all inbound traffic on 9080 from listener `0.0.0.0_15006`
   0.0.0.0            15001     TCP        // Receives all outbound traffic to the pod from IP tables and hands over to virtual listener
   0.0.0.0            15006     HTTP+TCP
   0.0.0.0            15090     HTTP
   0.0.0.0            15021     HTTP
   ```

   



