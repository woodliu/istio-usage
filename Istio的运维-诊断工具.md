# Istio的运维-诊断工具

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

2. 从上述输出概要中可以看到每个sidecar都有一个绑定到 `0.0.0.0:15006`的listener，IP tables会将所有入站的Pod流量导入该listener；以及一个绑定到 `0.0.0.0:15001`的listener，IP tables会将所有出站流量导入该listener，该listener有一个字段useOrig`i`nalDst设置为true，表示会使用最佳匹配原始目的地的方式将请求分发到virtual listener，如果没有找到任何virtual listener，将会直接发送到连接目的地的`PassthroughCluster`。

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

6. 使用`proxy-config endpoint`命令查看本集群中当前可用的后端

   ```shell
   $ istioctl pc endpoint productpage-v1-85b9bf9cd7-d8hm8.default --cluster "outbound|9080||reviews.default.svc.cluster.local"
   ENDPOINT            STATUS      OUTLIER CHECK     CLUSTER
   10.80.3.55:9080     HEALTHY     OK                outbound|9080||reviews.default.svc.cluster.local
   10.80.3.56:9080     HEALTHY     OK                outbound|9080||reviews.default.svc.cluster.local
   10.80.3.58:9080     HEALTHY     OK                outbound|9080||reviews.default.svc.cluster.local
   ```

流量方向为：listener(应用出站port)->route(`routeConfigName`)->cluster->endpoint(serviceName)

### 检查bootstrap配置

到目前为止已经查看了istiod的大部分配置，然而，Envoy需要一些如哪里可以发现istiod的bootstrap配置，使用如下方式查看：

```shell
$  istioctl proxy-config bootstrap -n istio-system istio-ingressgateway-569669bb67-dsd5h.istio-system
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

   