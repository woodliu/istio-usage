# Istio的运维-诊断工具

涵盖官方文档的[诊断工具](https://istio.io/docs/ops/diagnostic-tools/)章节

[TOC]

## 使用istioctl命令行工具

首先可以通过[日志](https://istio.io/docs/ops/diagnostic-tools/component-logging/)或[Introspection](https://istio.io/docs/ops/diagnostic-tools/controlz/)检查各个组件，如果不足以支持问题定位，可以参考如下操作：

[istioctl](https://istio.io/docs/reference/commands/istioctl)是一个可以用于调试和诊断istio服务网格的工具。Istio项目为Bash和ZSH运行下的`istioctl`提供了自动补全功能。

> 建议安装对应istio版本的`istioctl`。

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

可以使用`istioctl proxy-status`或`istioctl ps`命令查看网格的状态。如果输出结果中缺少某个代理，说明该代理当前没有连接到Pilot实例，因而无法接收到任何配置。如果状态为`stale`，表示当前存在网络故障，或Pilot需要扩容。

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

> istio的listener，route，cluster和endpoint与Envoy中的[概念](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/terminology#:~:text=Listener%3A%20A%20listener%20is%20a,hosts%20that%20Envoy%20connects%20to.)类似。
>
> - Cluster：在Envoy中，Cluster是一个服务集群，Cluster中包含一个到多个endpoint，每个endpoint都可以提供服务，Envoy根据负载均衡算法将请求发送到这些endpoint中。cluster分为inbound和outbound两种，前者对应Envoy所在节点上的服务；后者占了绝大多数，对应Envoy所在节点的外部服务。可以使用如下方式分别查看inbound和outbound的cluster：
>
>   ```shell
>   # istioctl pc cluster productpage-v1-64794f5db4-4h8c8.default --direction inbound -ojson
>   # istioctl pc cluster productpage-v1-64794f5db4-4h8c8.default --direction outbound -ojson
>   ```
>
> - Listeners：Envoy采用listener来接收并处理downstream发过来的请求。更多监听端口参见下文
>
> - Routes：配置Envoy的路由规则。Istio下发的缺省路由规则中对每个端口(服务)设置了一个路由规则，根据host来对请求进行路由分发，routes的目的为其他服务的cluster
>
> - Endpoint：cludter对应的后端服务，可以通过istio pc endpoint查看inbound和outbound对应的endpoint信息

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
NAME                                                  CDS      LDS       EDS       RDS        PILOT                     VERSION
details-v1-78d78fbddf-psnmk.default                   SYNCED   SYNCED    SYNCED    SYNCED     istiod-788cf6c878-4pq5g   1.6.0
istio-ingressgateway-569669bb67-dsd5h.istio-system    SYNCED   SYNCED    SYNCED    NOT SENT   istiod-788cf6c878-4pq5g   1.6.0
productpage-v1-85b9bf9cd7-d8hm8.default               SYNCED   SYNCED    SYNCED    SYNCED     istiod-788cf6c878-4pq5g   1.6.0
prometheus-79878ff5fd-tjdxx.istio-system              SYNCED   SYNCED    SYNCED    SYNCED     istiod-788cf6c878-4pq5g   1.6.0
ratings-v1-6c9dbf6b45-xlf2q.default                   SYNCED   SYNCED    SYNCED    SYNCED     istiod-788cf6c878-4pq5g   1.6.0
reviews-v1-564b97f875-q5l9r.default                   SYNCED   SYNCED    SYNCED    SYNCED     istiod-788cf6c878-4pq5g   1.6.0
reviews-v2-568c7c9d8f-vcd94.default                   SYNCED   SYNCED    SYNCED    SYNCED     istiod-788cf6c878-4pq5g   1.6.0
reviews-v3-67b4988599-psllq.default                   SYNCED   SYNCED    SYNCED    SYNCED     istiod-788cf6c878-4pq5g   1.6.0
sleep-78484c89dd-fmxbc.default                        SYNCED   SYNCED    SYNCED    SYNCED     istiod-788cf6c878-4pq5g   1.6.0
```

### 检索Envoy和istiod的差异

`proxy-status`加上proxy ID可以检索Envoy加载的配置和istiod发送的配置之间的差异，通过这种方式可以确定哪部分内容没有被同步，并确定可能存在的问题。

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

`proxy-config`命令可以查看一个Envoy实例的配置，用于定位无法通过查看istio配置和用户资源发现的问题。例如，使用如下命令可以获取特定pod的clusters，listeners或routes概要。*注：首先通过istioctl ps查看出不匹配的代理，然后使用istioctl pc查看具体的不匹配的信息。*

```shell
$ istioctl proxy-config cluster -n istio-system istio-ingressgateway-7d6874b48f-qxhn5
SERVICE FQDN                                           PORT    SUBSET  DIRECTION  TYPE
BlackHoleCluster                                       -       -       -          STATIC
agent                                                  -       -       -          STATIC
details.default.svc.cluster.local                      9080    -       outbound   EDS
istio-ingressgateway.istio-system.svc.cluster.local    80      -       outbound   EDS
istio-ingressgateway.istio-system.svc.cluster.local    443     -       outbound   EDS
istio-ingressgateway.istio-system.svc.cluster.local    15021   -       outbound   EDS
istio-ingressgateway.istio-system.svc.cluster.local    15443   -       outbound   EDS
istiod.istio-system.svc.cluster.local                  443     -       outbound   EDS
istiod.istio-system.svc.cluster.local                  853     -       outbound   EDS
istiod.istio-system.svc.cluster.local                  15010   -       outbound   EDS
istiod.istio-system.svc.cluster.local                  15012   -       outbound   EDS
istiod.istio-system.svc.cluster.local                  15014   -       outbound   EDS
kube-dns.kube-system.svc.cluster.local                 53      -       outbound   EDS
kube-dns.kube-system.svc.cluster.local                 9153    -       outbound   EDS
kubernetes.default.svc.cluster.local                   443     -       outbound   EDS
...                                                                               
productpage.default.svc.cluster.local                  9080    -       outbound   EDS
prometheus.istio-system.svc.cluster.local              9090    -       outbound   EDS
prometheus_stats                                       -       -       -          STATIC
ratings.default.svc.cluster.local                      9080    -       outbound   EDS
reviews.default.svc.cluster.local                      9080    -       outbound   EDS
sds-grpc                                               -       -       -          STATIC
xds-grpc                                               -       -       -          STRICT_DNS
zipkin                                                 -       -       -          STRICT_DNS
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
   | 15001 | TCP      | Envoy                       | Envoy Outbound                          |
   | 15006 | TCP      | Envoy                       | Envoy Inbound                           |
   | 15008 | TCP      | Envoy                       | Envoy Tunnel port (Inbound)             |
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

   可以看到`TYPE`字段是没有`HTTPS`的，`HTTPS`作为`TCP`类型。下面是`productpage`的listeners，删减了部分信息。`10.84`开头的是各个kubernetes service的`CLUSTER-IP`，以`172.20`开头的是kubernetes的node IP，以nodePort方式暴露服务。

   ```shell
   $ istioctl proxy-config listeners productpage-v1-85b9bf9cd7-d8hm8.default
   ADDRESS          PORT     TYPE    
   0.0.0.0          443      TCP     <--+
   10.84.71.37      443      TCP        |
   10.84.223.189    443      TCP        |
   10.84.100.226    15443    TCP        |
   10.84.121.154    443      TCP        |
   10.84.142.44     443      TCP        | #从0.0.0.0_15001相关IP:PORT上接收出站的non-HTTP流量
   10.84.155.219    443      TCP        |
   172.20.127.212   9100     TCP        |
   10.84.205.103    443      TCP        | 
   10.84.167.116    443      TCP        |
   172.20.127.211   9100     TCP     <--+
   10.84.113.197    9979     HTTP+TCP<--+
   0.0.0.0          9091     HTTP+TCP   |
   10.84.30.227     9092     HTTP+TCP   |
   10.84.108.37     8080     HTTP+TCP   |
   10.84.158.64     8443     HTTP+TCP   |
   10.84.202.185    8080     HTTP+TCP   |
   10.84.21.252     8443     HTTP+TCP   |
   10.84.215.56     8443     HTTP+TCP   |
   0.0.0.0          60000    HTTP+TCP   | # 从0.0.0.0_15001的相关端口上接收出站的HTTP+TCP流量
   10.84.126.74     8778     HTTP+TCP   |
   10.84.126.74     8080     HTTP+TCP   |
   10.84.123.207    8080     HTTP+TCP   |
   10.84.30.227     9091     HTTP+TCP   |
   10.84.229.5      8080     HTTP+TCP<--+
   0.0.0.0          9080     HTTP+TCP     # 从 0.0.0.0_15006 上接收所有到9080的入站流量
   0.0.0.0          15001    TCP          # 从IP tables接收pod的所有出站流量，并移交给虚拟侦听器
   0.0.0.0          15006    HTTP+TCP     # Envoy 入站
   0.0.0.0          15090    HTTP         # Envoy Prometheus 遥测
   0.0.0.0          15021    HTTP         # 健康检查
   ```

   下面是`productpage` Pod中实际监听的端口信息，与上述对应。

   ```shell
   $ ss -ntpl                                 
   State          Recv-Q        Send-Q        Local Address:Port          Peer Address:Port
   LISTEN         0             128                 0.0.0.0:15090              0.0.0.0:*
   LISTEN         0             128               127.0.0.1:15000              0.0.0.0:*
   LISTEN         0             128                 0.0.0.0:9080               0.0.0.0:*
   LISTEN         0             128                 0.0.0.0:15001              0.0.0.0:*
   LISTEN         0             128                 0.0.0.0:15006              0.0.0.0:*
   LISTEN         0             128                 0.0.0.0:15021              0.0.0.0:*
   LISTEN         0             128                       *:15020                    *:* 
   ```

2. 从上述输出概要中可以看到每个sidecar都有一个绑定到 `0.0.0.0:15006`的listener，IP tables会将所有入站的Pod流量导入该listener；以及一个绑定到 `0.0.0.0:15001`的listener，IP tables会将所有出站流量导入该listener，该listener有一个字段`useOriginalDst`设置为true，表示会使用最佳匹配原始目的地的方式将请求分发到virtual listener，如果没有找到任何virtual listener，将会直接发送到连接目的地的`PassthroughCluster`。

   ```shell
   $ istioctl pc listener productpage-v1-85b9bf9cd7-d8hm8.default --port 15001 -o json
   [
       {
           "name": "virtualOutbound",
           "address": {
               "socketAddress": {
                   "address": "0.0.0.0",
                   "portValue": 15001
               }
           },
           "filterChains": [
               {
                   "filters": [
                       {
                           "name": "istio.stats",
                           "typedConfig": {
                               "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                               "typeUrl": "type.googleapis.com/envoy.extensions.filters.network.wasm.v3.Wasm",
                               "value": {
                                   "config": {
                                       "configuration": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\"\n}\n",
                                       "root_id": "stats_outbound",
                                       "vm_config": {
                                           "code": {
                                               "local": {
                                                   "inline_string": "envoy.wasm.stats"
                                               }
                                           },
                                           "runtime": "envoy.wasm.runtime.null",
                                           "vm_id": "tcp_stats_outbound"
                                       }
                                   }
                               }
                           }
                       },
                       {
                           "name": "envoy.tcp_proxy",
                           "typedConfig": {
                               "@type": "type.googleapis.com/envoy.config.filter.network.tcp_proxy.v2.TcpProxy",
                               "statPrefix": "PassthroughCluster",
                               "cluster": "PassthroughCluster",
                               "accessLog": [
                                   {
                                       "name": "envoy.file_access_log",
                                       "typedConfig": {
                                           "@type": "type.googleapis.com/envoy.config.accesslog.v2.FileAccessLog",
                                           "path": "/dev/stdout",
                                           "format": "[%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%DYNAMIC_METADATA(istio.mixer:status)%\" \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"
                                       }
                                   }
                               ]
                           }
                       }
                   ],
                   "name": "virtualOutbound-catchall-tcp"
               }
           ],
           "useOriginalDst": true,
           "trafficDirection": "OUTBOUND"
       }
   ]
   ```

3. 应用的请求为一个出站HTTP请求，会到达`9080`端口，这意味着请求将传递给`0.0.0.0:9080` virtual listener。该listener会查看其RDS中配置的路由。这种情况下会查找istiod配置的RDS的`9080`路由

   ```shell
   $ istioctl pc listener productpage-v1-85b9bf9cd7-d8hm8.default  -o json --address 0.0.0.0 --port 9080
   [
       {
           "name": "0.0.0.0_9080",
           "address": {
               "socketAddress": {
                   "address": "0.0.0.0",
                   "portValue": 9080
               }
           },
           "filterChains": [
               {
                   "filterChainMatch": {
                       "applicationProtocols": [
                           "http/1.0",
                           "http/1.1",
                           "h2c"
                       ]
                   },
                   "filters": [
                       {
                           "name": "envoy.http_connection_manager",
                           "typedConfig": {
                               "@type": "type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager",
                               "statPrefix": "outbound_0.0.0.0_9080",
                               "rds": {
                                   "configSource": {
                                       "ads": {}
                                   },
                                   "routeConfigName": "9080"
                               },
   ...
   ]
   ```

4. 9080的路由配置中每个服务只有一个virtual host。由于应用的请求会传递到reviews服务，因此Envoy会选择请求与域相匹配的virtual host。一旦匹配到域，Envoy会查看匹配请求的第一个路由。下面场景中，由于没有配置任何高级路由，因此只有一条可以匹配的路由，该路由告诉Envoy将请求发送到 `outbound|9080||reviews.default.svc.cluster.local`集群。

   ```shell
   $ istioctl proxy-config routes productpage-v1-85b9bf9cd7-d8hm8.default  --name 9080 -o json
   [
       {
           "name": "9080",
           "virtualHosts": [
   ...
               {
                   "name": "reviews.default.svc.cluster.local:9080", 
                   "domains": [
                       "reviews.default.svc.cluster.local",
                       "reviews.default.svc.cluster.local:9080",
                       "reviews",
                       "reviews:9080",
                       "reviews.default.svc.cluster",
                       "reviews.default.svc.cluster:9080",
                       "reviews.default.svc",
                       "reviews.default.svc:9080",
                       "reviews.default",
                       "reviews.default:9080",
                       "10.84.110.152",
                       "10.84.110.152:9080"
                   ],
                   "routes": [
                       {
                           "name": "default",
                           "match": {
                               "prefix": "/"
                           },
                           "route": {
                               "cluster": "outbound|9080||reviews.default.svc.cluster.local",
                               "timeout": "0s",
                               "retryPolicy": {
                                   "retryOn": "connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes",
                                   "numRetries": 2,
                                   "retryHostPredicate": [
                                       {
                                           "name": "envoy.retry_host_predicates.previous_hosts"
                                       }
                                   ],
                                   "hostSelectionRetryMaxAttempts": "5",
                                   "retriableStatusCodes": [
                                       503
                                   ]
                               },
                               "maxGrpcTimeout": "0s"
                           },
                           "decorator": {
                               "operation": "reviews.default.svc.cluster.local:9080/*"
                           }
                       }
                   ],
                   "includeRequestAttemptCount": true
               }
           ],
           "validateClusters": false
       }
   ]
   ```

5. cluster的配置用于从istiod中检索后端。Envoy会使用`serviceName`作为key在Endpoints列表中进行查找，并将请求传递到这些后端。

   ```shell
   $  istioctl pc cluster productpage-v1-85b9bf9cd7-d8hm8.default  --fqdn reviews.default.svc.cluster.local -o json
   [
       {
          ...
           "name": "outbound|9080||reviews.default.svc.cluster.local",
           "type": "EDS",
           "edsClusterConfig": {
               "edsConfig": {
                   "ads": {}
               },
               "serviceName": "outbound|9080||reviews.default.svc.cluster.local"
           },
           "connectTimeout": "10s",
           "circuitBreakers": {
               "thresholds": [
                   {
                       "maxConnections": 4294967295,
                       "maxPendingRequests": 4294967295,
                       "maxRequests": 4294967295,
                       "maxRetries": 4294967295
                   }
               ]
           },
           "filters": [
               {
                   "name": "istio.metadata_exchange",
                   "typedConfig": {
                       "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                       "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
                       "value": {
                           "protocol": "istio-peer-exchange"
                       }
                   }
               }
           ]
       }
   ]
   ```

6. 使用`proxy-config endpoint`命令查看本集群中当前可用的后端，也可以直接指定端口查看`--port`

   ```shell
   $ istioctl pc endpoint productpage-v1-85b9bf9cd7-d8hm8.default --cluster "outbound|9080||reviews.default.svc.cluster.local"
   ENDPOINT            STATUS      OUTLIER CHECK     CLUSTER
   10.80.3.55:9080     HEALTHY     OK                outbound|9080||reviews.default.svc.cluster.local
   10.80.3.56:9080     HEALTHY     OK                outbound|9080||reviews.default.svc.cluster.local
   10.80.3.58:9080     HEALTHY     OK                outbound|9080||reviews.default.svc.cluster.local
   ```

流量方向为：listener(应用出站port)->route(`routeConfigName`)->cluster(domain)->endpoint(serviceName)

### 检查bootstrap配置

到目前为止已经查看了istiod的大部分配置，然而，Envoy需要一些如哪里可以发现istiod的bootstrap配置，使用如下方式查看：

```shell
$ istioctl proxy-config bootstrap -n istio-system istio-ingressgateway-569669bb67-dsd5h.istio-system
{
    "bootstrap": {
        "node": {
            "id": "router~10.83.0.14~istio-ingressgateway-569669bb67-dsd5h.istio-system~istio-system.svc.cluster.local",
            "cluster": "istio-ingressgateway",
            "metadata": {
                    "CLUSTER_ID": "Kubernetes",
                    "CONFIG_NAMESPACE": "istio-system",
                    "EXCHANGE_KEYS": "NAME,NAMESPACE,INSTANCE_IPS,LABELS,OWNER,PLATFORM_METADATA,WORKLOAD_NAME,MESH_ID,SERVICE_ACCOUNT,CLUSTER_ID",
                    "INSTANCE_IPS": "10.83.0.14,fe80::6871:95ff:fe5b:9e3e",
                    "ISTIO_PROXY_SHA": "istio-proxy:12cfbda324320f99e0e39d7c393109fcd824591f",
                    "ISTIO_VERSION": "1.6.0",
                    "LABELS": {
                                "app": "istio-ingressgateway",
                                "chart": "gateways",
                                "heritage": "Tiller",
                                "istio": "ingressgateway",
                                "pod-template-hash": "569669bb67",
                                "release": "istio",
                                "service.istio.io/canonical-name": "istio-ingressgateway",
                                "service.istio.io/canonical-revision": "latest"
                            },
                    "MESH_ID": "cluster.local",
                    "NAME": "istio-ingressgateway-569669bb67-dsd5h",
                    "NAMESPACE": "istio-system",
                    "OWNER": "kubernetes://apis/apps/v1/namespaces/istio-system/deployments/istio-ingressgateway",
...
                    "ROUTER_MODE": "sni-dnat",
                    "SDS": "true",
                    "SERVICE_ACCOUNT": "istio-ingressgateway-service-account",
                    "TRUSTJWT": "true",
                    "WORKLOAD_NAME": "istio-ingressgateway"
                },
...
}
```

### 校验到istiod的连通性

服务网格中的所有Envoy代理容器都应该连接到istiod，使用如下步骤测试：

1. 创建一个`sleep` pod

   ```shell
   $ kubectl create namespace foo
   $ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml) -n foo
   ```

2. 使用curl测试到istiod的连通性。下面调用v1注册API，使用默认的istiod配置参数，并启用双向TLS认证

   ```shell
   $ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl -sS istiod.istio-system:15014/debug/endpointz
   ```


## 通过istioctl的输出理解网格

> 如下内容是一个实验特性，仅用于评估

istio 1.3中包含一个[istioctl experimental describe](https://istio.io/docs/reference/commands/istioctl/#istioctl-experimental-describe-pod)命令。该CLI命令提供了解影响pod的配置所需的信息。本节展示如何使用该experimental 子命令查看一个pod是否在网格中，以及检查该pod的配置。该命令的基本使用方式为：

```shell
$ istioctl experimental describe pod <pod-name>[.<namespace>] #或
$ istioctl experimental describe pod <pod-name> -n <namespace> 
```

### 校验pod是否在网格中

如果一个pod不在网格中，`istioctl describe`会显示一个告警信息，此外，如果pod缺少[istio需要的配置](https://istio.io/docs/ops/deployment/requirements/)时也会给出告警信息。

```shell
$ istioctl experimental describe pod  mutatepodimages-7575797d95-qn7p5
Pod: mutatepodimages-7575797d95-qn7p5
   Pod does not expose ports
WARNING: mutatepodimages-7575797d95-qn7p5 is not part of mesh; no Istio sidecar
--------------------
Error: failed to execute command on sidecar: error 'execing into mutatepodimages-7575797d95-qn7p5/default istio-proxy container: container istio-proxy is not valid for pod mutatepodimages-7575797d95-qn7p5
```

如果一个pod在网格中，则不会产生告警。

```shell
$ istioctl x describe pod ratings-v1-6c9dbf6b45-xlf2q
Pod: ratings-v1-6c9dbf6b45-xlf2q
   Pod Ports: 9080 (details), 15090 (istio-proxy)
--------------------
Service: details
   Port: http 9080/HTTP targets pod port 9080
Pilot reports that pod enforces HTTP/mTLS and clients speak HTTP
```

输出为：

- pod的服务容器端口，上述为`ratings`的9080端口
- pod中的istio-proxy端口，15090
- pod服务使用的协议，9080端口的http协议
- pod设置的mutual  TLS

### 校验destination rule配置

可以使用`istioctl describe`检查应用到一个pod的destination rule。例如执行如下命令部署destination rule

```shell
$ kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```

查看`ratings` pod

```shell
$ export RATINGS_POD=$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')
$ istioctl x describe pod $RATINGS_POD
Pod: ratings-v1-6c9dbf6b45-xlf2q
   Pod Ports: 9080 (ratings), 15090 (istio-proxy)
--------------------
Service: ratings
   Port: http 9080/HTTP targets pod port 9080
DestinationRule: ratings for "ratings"
   Matching subsets: v1
      (Non-matching subsets v2,v2-mysql,v2-mysql-vm)
   Traffic Policy TLS Mode: ISTIO_MUTUAL
Pilot reports that pod enforces HTTP/mTLS and clients speak mTLS
```

输出为：

- 应用到`ratings` 服务的`ratings` destination rule
- 匹配pod的`ratings` destination rule，上述为`v1`
- destination rule定义的其他subset
- pod接收HTTP或mutual TLS，但客户端使用mutual TLS

### 校验virtual service配置

部署如下virtual service

```shell
$ kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

查看`v1`版本的`reviews`服务：

```shell
$ export REVIEWS_V1_POD=$(kubectl get pod -l app=reviews,version=v1 -o jsonpath='{.items[0].metadata.name}')
istioctl x describe pod $REVIEWS_V1_POD
$ istioctl x describe pod $REVIEWS_V1_POD
Pod: reviews-v1-564b97f875-q5l9r
   Pod Ports: 9080 (reviews), 15090 (istio-proxy)
--------------------
Service: reviews
   Port: http 9080/HTTP targets pod port 9080
DestinationRule: reviews for "reviews"
   Matching subsets: v1
      (Non-matching subsets v2,v3)
   Traffic Policy TLS Mode: ISTIO_MUTUAL
VirtualService: reviews
   1 HTTP route(s)
```

输出结果与前面的`ratings` pod类型，但多了到pod的virtual service路由。

`istioctl describe`命令不仅仅显示了影响pod的virtual service。如果一个virtual service配置的host为一个pod，但流量不可达，会输出告警信息，这种请求可能发生在当一个virtual service如果没有可达的pod subset时。例如：

```shell
$ export REVIEWS_V2_POD=$(kubectl get pod -l app=reviews,version=v2 -o jsonpath='{.items[0].metadata.name}')
istioctl x describe pod $REVIEWS_V2_POD
[root@bastion istio-1.6.0]# istioctl x describe pod $REVIEWS_V2_POD
Pod: reviews-v2-568c7c9d8f-vcd94
...
VirtualService: reviews
   WARNING: No destinations match pod subsets (checked 1 HTTP routes)
      Route to non-matching subset v1 for (everything)
```

告警信息给出导致问题的原因，检查的路由数目，以及其他路由信息。例如，由于virtual service将所有的流量到导入了`v1` subset，因此`v2` pod无法接收到任何流量。

如果删除如下destination rule：

```shell
$ kubectl delete -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```

可以看到如下信息：

```shell
$ istioctl x describe pod $REVIEWS_V1_POD
Pod: reviews-v1-564b97f875-q5l9r
   Pod Ports: 9080 (reviews), 15090 (istio-proxy)
--------------------
Service: reviews
   Port: http 9080/HTTP targets pod port 9080
VirtualService: reviews
   WARNING: No destinations match pod subsets (checked 1 HTTP routes)
      Warning: Route to subset v1 but NO DESTINATION RULE defining subsets!
```

输出展示了已删除destination rule，但没有删除依赖它的virtual service。该virtual service将流量路由到`v1` subset，但没有定义`v1` subset的destination rule。因此流量无法分发到`v1`版本的pod。

恢复环境：

```shell
$ kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```

### 校验流量路由

`istioctl describe`也可以展示流量权重。如理如下命令会将90%的流量导入`reviews`服务的`v1` subset，将10%的流量导入`reviews`服务的`v2` subset。

```shell
$ kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-90-10.yaml
```

查看reviews v1` pod：

```shell
$ istioctl x describe pod $REVIEWS_V1_POD
...
VirtualService: reviews
   Weight 90%
```

输出显示90%的`reviews`服务的流量导入到了`v1` subset中。

部署其他类型的路由，如部署指定HTTP首部的路由：

```shell
$ kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-jason-v2-v3.yaml
```

再次查看pod：

```shell
$ istioctl x describe pod $REVIEWS_V1_POD
...
VirtualService: reviews
   WARNING: No destinations match pod subsets (checked 2 HTTP routes)
      Route to non-matching subset v2 for (when headers are end-user=jason)
      Route to non-matching subset v3 for (everything)
```

由于查看了位于`v1` subset的pod，而virtual service将包含 `end-user=jason` 的流量分发给`v2` subset，其他流量分发给`v3` subset，`v1` subset没有任何流量导入，此时会输出告警信息。

### 检查strict mutual TLS(官方文档待更新)

根据[mutual TLS迁移指南](https://istio.io/docs/tasks/security/authentication/mtls-migration/)，可以给ratings服务启用strict mutual TLS。

```shell
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: ratings-strict
spec:
  selector:
    matchLabels:
      app: ratings
  mtls:
    mode: STRICT
EOF
```

执行如下命令查看`ratings` pod，输出显示`ratings` pod已经使用mutual TLS防护。

```shell
$ istioctl x describe pod $RATINGS_POD
Pilot reports that pod enforces mTLS and clients speak mTLS
```

有时，将mutual TLS切换位`STRICT`模式时会对部署的组件造成影响，通常是因为destination rule不匹配新配置造成的。例如，如果配置Bookinfo客户端不使用mutualTLS，而使用[明文的HTTP destination rules](https://raw.githubusercontent.com/istio/istio/release-1.6/samples/bookinfo/networking/destination-rule-all.yaml):

```shell
$ kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
```

如果在浏览器傻瓜打开Bookinfo，会显示`Ratings service is currently unavailable`，使用如下命令查看原因：

```shell
$ istioctl x describe pod $RATINGS_POD
...
WARNING Pilot predicts TLS Conflict on ratings-v1-f745cf57b-qrxl2 port 9080 (pod enforces mTLS, clients speak HTTP)
  Check DestinationRule ratings/default and AuthenticationPolicy ratings-strict/default
```

输出中有一个描述destination rule和authentication policy冲突的告警信息。

使用如下方式恢复：

```shell
$ kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```

### 卸载

```shell
$ kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
$ kubectl delete -f samples/bookinfo/networking/bookinfo-gateway.yaml
$ kubectl delete -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
$ kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

## 使用istioctl analyse诊断配置

`istioctl analyze`是一个可以探测istio配置中潜在错误的诊断工具，它可以诊断现有的集群或一组本地配置文件，会同时诊断这两者。

可以使用如下方式诊断当前的kubernetes：

```shell
$ istioctl analyze --all-namespaces
```

例如，如果某些命名空间没有启用istiod注入，会打印如下告警信息：

```verilog
Warn [IST0102] (Namespace openshift) The namespace is not enabled for Istio injection. Run 'kubectl label namespace openshift istio-injection=enabled' to enable it, or 'kubectl label namespace openshift istio-injection=disabled' to explicitly mark it as not needing injection
```

### 分析现有的集群/本地文件或二者

上述的例子用于分析一个存在的集群，但该工具也可以支持分析本地kubernetes yaml的配置文件集，或同时分析本地文件和集群。当分析一个本地文件集时，这些文件集应该是完全自包含的。通常用于分析需要部署到集群的一个完整的配置文件集。

分析特定的本地kubernetes yaml文件集：

```shell
$ istioctl analyze --use-kube=false a.yaml b.yaml
```

分析当前目录中的所有yaml文件：

```shell
$ istioctl analyze --use-kube=false *.yaml
```

模拟将当前目录中的files部署到当前集群中：

```shell
$ istioctl analyze *.yaml
```

使用`istioctl analyze --help`命令查看完整的选项。更多analyse的使用参见[Q&A](https://istio.io/docs/ops/diagnostic-tools/istioctl-analyze/#q-a).

## 组件内省

Istio组件是用一个灵活的内省框架构建的，它使检查和操作运行组件的内部状态变得简单。组件会开放一个端口，用于通过web浏览器的交互式视图获取组件的状态，或使用外部工具通过REST访问。

`Mixer`, `Pilot`和`Galley` 都实现了ControlZ 功能(1.6版本可以查看istiod)。当启用这些组件时将记录一条消息，指示要连接的IP地址和端口，以便与ControlZ交互。

```verilog
2018-07-26T23:28:48.889370Z     info    ControlZ available at 100.76.122.230:9876
```

可以使用如下命令进行端口转发，类似kubectl的`port-forward`，用于远程访问。

```shell
$ istioctl dashboard controlz <podname> -n <namespaces>
```

## 组件日志

### 日志作用域

组件的日志按照作用域进行分类。取决于组件提供的功能，不同的组件有不同的作用域。所有的组件都有一个`default`的作用域，用于未分类的日志消息。

各个组件的日志作用域参见：[reference documentation](https://istio.io/docs/reference/commands/)。

每个作用域对应一个唯一的日志级别：

1. none
2. error
3. warning
4. info
5. debug

其中none表示没有划分作用域的输出，`debug`会最大化输出。默认的作用域为`info`，用于在一般情况下为istio提供何时的日志输出。

可以使用 `--log_output_level` 控制输出级别：

### 控制输出

日志信息通常会发送到组件的标准输出流中。 `--log_target`选项可以将输出重定向到任意(数量的)位置，可以通过逗号分割的列表给出文件系统的路径。`stdout` 和`stderr`分别表示标准输出和标准错误输出流。

### 日志滚动

istio组件能够自动管理日志滚动，将大的日志切分为小的日志文件。`--log_rotate`选项允许指定用于滚动的基本文件名。派生的名称将用于单个日志文件。

`--log_rotate_max_age`选项指定文件发生滚动前的最大时间(天为单位)，`--log_rotate_max_size`选项用于指定文件滚动发生前的最大文件大小(MB为单位)， `--log_rotate_max_backups`选项控制保存的滚动文件的最大数量，超过该值的老文件会被自动删除。

### 组件调试

`--log_caller` 和`--log_stacktrace_level`选项可以控制日志信息是否包含程序员级别的信息。在跟踪组件bug时很有用，但日常用不到。

### 参考

- [Istio流量管理实现机制深度解析](https://zhaohuabing.com/post/2018-09-25-istio-traffic-management-impl-intro/#virtual-listener)
- [理解 Istio Service Mesh 中 Envoy Sidecar 代理的路由转发](https://jimmysong.io/blog/envoy-sidecar-routing-of-istio-service-mesh-deep-dive/)