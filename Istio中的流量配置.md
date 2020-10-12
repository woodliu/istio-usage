## Istio中的流量配置

[TOC]

### Istio注入的容器

Istio的数据面会在pod中注入两个容器：`istio-init`和`istio-proxy`。

#### Istio-init

`istio-init`会通过创建iptables规则来接管流量：

- 命令行参数 -p `15001`表示出向流量被iptable重定向到Envoy的`15001`端口

- 命令行参数 -z `15006`表示入向流量被iptable重定向到Envoy的`15006`端口

- 命令行参数 -u `1337`参数用于排除用户ID为`1337`，即Envoy自身的流量，以避免Iptable把Envoy发出的数据又重定向到Envoy，形成死循环。在istio-proxy容器中执行如下命令可以看到Envoy使用的用户id为`1337`。

  ```shell
  $ id
  uid=1337(istio-proxy) gid=1337(istio-proxy) groups=1337(istio-proxy)
  ```

```shell
  istio-iptables
  -p
  15001
  -z
  15006
  -u
  1337
  -m
  REDIRECT
  -i
  *
  -x

  -b
  *
  -d
  15090,15021,15020
  --run-validation
  --skip-rule-apply
```
#### istio-proxy

`istio-proxy`容器中会运行两个程序：`pilot-agent`和`envoy`。

```shell
$ ps -ef|cat
UID          PID    PPID  C STIME TTY          TIME CMD
istio-p+       1       0  0 Sep10 ?        00:03:39 /usr/local/bin/pilot-agent proxy sidecar --domain default.svc.cluster.local --serviceCluster sleep.default --proxyLogLevel=warning --proxyComponentLogLevel=misc:error --trust-domain=cluster.local --concurrency 2
istio-p+      27       1  0 Sep10 ?        00:14:30 /usr/local/bin/envoy -c etc/istio/proxy/envoy-rev0.json --restart-epoch 0 --drain-time-s 45 --parent-shutdown-time-s 60 --service-cluster sleep.default --service-node sidecar~10.80.3.109~sleep-856d589c9b-x6szk.default~default.svc.cluster.local --local-address-ip-version v4 --log-format-prefix-with-location 0 --log-format %Y-%m-%dT%T.%fZ.%l.envoy %n.%v -l warning --component-log-level misc:error --concurrency 2
```

### Envoy架构

Envoy对入站/出站请求的处理过程如下，Envoy按照如下顺序依次在各个过滤器中处理请求。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200916144803520-1592409597.png)

典型的入站请求流程[如下](https://www.envoyproxy.io/docs/envoy/latest/intro/life_of_a_request#network-filter-chain-processing)，首先在监听过滤器链中解析入站报文中的TLS，然后通过传输socket建立连接，最后由网络过滤器链进行处理(含HTTP连接管理器)。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200913155624620-2072767193.png)

### Pilot-agent生成的初始配置文件

`pilot-agent`根据启动参数和K8S API Server中的配置信息生成Envoy的[bootstrap](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#bootstrap)文件(`/etc/istio/proxy/envoy-rev0.json`)，并负责启动Envoy进程(可以看到`Envoy`进程的父进程是`pilot-agent`)；`envoy`会通过xDS接口从istiod动态获取配置文件。`envoy-rev0.json`初始配置文件结构如下：

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200914110435327-1158938665.png)

- [node](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#config-core-v3-node)：给出了Envoy 实例的信息

  ```json
    "node": {
      "id": "sidecar~10.80.3.109~sleep-856d589c9b-x6szk.default~default.svc.cluster.local",
      "cluster": "sleep.default",
      "locality": {
      },
      "metadata": {"APP_CONTAINERS":"sleep,istio-proxy","CLUSTER_ID":"Kubernetes","EXCHANGE_KEYS":"NAME,NAMESPACE,INSTANCE_IPS,LABELS,OWNER,PLATFORM_METADATA,WORKLOAD_NAME,MESH_ID,SERVICE_ACCOUNT,CLUSTER_ID","INSTANCE_IPS":"10.80.3.109,fe80::40fb:daff:feed:e56c","INTERCEPTION_MODE":"REDIRECT","ISTIO_PROXY_SHA":"istio-proxy:f642a7fd07d0a99944a6e3529566e7985829839c","ISTIO_VERSION":"1.7.0","LABELS":{"app":"sleep","istio.io/rev":"default","pod-template-hash":"856d589c9b","security.istio.io/tlsMode":"istio","service.istio.io/canonical-name":"sleep","service.istio.io/canonical-revision":"latest"},"MESH_ID":"cluster.local","NAME":"sleep-856d589c9b-x6szk","NAMESPACE":"default","OWNER":"kubernetes://apis/apps/v1/namespaces/default/deployments/sleep","POD_PORTS":"[{\"name\":\"http-envoy-prom\",\"containerPort\":15090,\"protocol\":\"TCP\"}]","PROXY_CONFIG":{"binaryPath":"/usr/local/bin/envoy","concurrency":2,"configPath":"./etc/istio/proxy","controlPlaneAuthPolicy":"MUTUAL_TLS","discoveryAddress":"istiod.istio-system.svc:15012","drainDuration":"45s","envoyAccessLogService":{},"envoyMetricsService":{},"parentShutdownDuration":"60s","proxyAdminPort":15000,"proxyMetadata":{"DNS_AGENT":""},"serviceCluster":"sleep.default","statNameLength":189,"statusPort":15020,"terminationDrainDuration":"5s","tracing":{"zipkin":{"address":"zipkin.istio-system:9411"}}},"SDS":"true","SERVICE_ACCOUNT":"sleep","WORKLOAD_NAME":"sleep","k8s.v1.cni.cncf.io/networks":"istio-cni","sidecar.istio.io/interceptionMode":"REDIRECT","sidecar.istio.io/status":"{\"version\":\"8e6e902b765af607513b28d284940ee1421e9a0d07698741693b2663c7161c11\",\"initContainers\":[\"istio-validation\"],\"containers\":[\"istio-proxy\"],\"volumes\":[\"istio-envoy\",\"istio-data\",\"istio-podinfo\",\"istiod-ca-cert\"],\"imagePullSecrets\":null}","traffic.sidecar.istio.io/excludeInboundPorts":"15020","traffic.sidecar.istio.io/includeOutboundIPRanges":"*"}
    },
  ```

- [admin](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#config-bootstrap-v3-admin)：给出了Envoy的日志路径和管理端口，例如可以通过`curl -X POST localhost:15000/logging?level=trace`设置日志级别为`trace`。

  ```json
    "admin": {
      "access_log_path": "/dev/null", /* 管理服务器的访问日志路径 */
      "profile_path": "/var/lib/istio/data/envoy.prof", /* 管理服务器的CPU输出路径 */
      "address": { /* 管理服务器监听的TCP地址 */
        "socket_address": {
          "address": "127.0.0.1",
          "port_value": 15000
        }
      }
    },
  ```

- [dynamic_resources](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#config-bootstrap-v3-bootstrap-dynamicresources)：配置动态资源，用于配置[lds_config](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/config_source.proto#config-core-v3-configsource)，[cds_config](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/config_source.proto#config-core-v3-configsource)和[ads_config](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/config_source.proto#config-core-v3-apiconfigsource)

  > Envoy通过`xds-grpc `cluster(参见`static_resources`)来获得xDS服务的地址。

  ```json
    "dynamic_resources": {
      "lds_config": { /* 通过一个LDS配置Listeners */
        "ads": {},
        "resource_api_version": "V3" /* LDS使用的API版本号 */
      },
      "cds_config": { /* 通过一个CDS配置Cluster */
        "ads": {},
        "resource_api_version": "V3"
      },
      "ads_config": { /* API配置资源，用于指定API类型和Envoy获取xDS API的cluster */
        "api_type": "GRPC", /* 使用GRPC获取xDS信息 */
        "transport_api_version": "V3", /* xDS传输协议的版本号 */
        "grpc_services": [
          {
            "envoy_grpc": {
              "cluster_name": "xds-grpc" /* 动态获取xDS配置的cluster */
            }
          }
        ]
      }
    },
  ```

- [static_resources](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#config-bootstrap-v3-bootstrap-staticresources)：配置静态资源，主要包括[clusters](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/cluster.proto#config-cluster-v3-cluster)和[listeners](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/listener.proto#config-listener-v3-listener)两种资源：

  ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200914154659614-940913736.png)

  - clusters：下面给出了几个静态配置的cluster。

    ```json
        "clusters": [
          {
            "name": "prometheus_stats", /* 使用Prometheus暴露metrics，接口为127.0.0.1:15000/stats/prometheus */
            "type": "STATIC", /* 明确指定上游host的网络名(IP地址/端口等) */
            "connect_timeout": "0.250s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": { /* 仅用于类型为STATIC, STRICT_DNS或LOGICAL_DNS，用于给非EDS的clyster内嵌与EDS等同的endpoint */
              "cluster_name": "prometheus_stats", 
              "endpoints": [{
                "lb_endpoints": [{ /* 负载均衡的后端 */
                  "endpoint": {
                    "address":{
                      "socket_address": {
                        "protocol": "TCP",
                        "address": "127.0.0.1",
                        "port_value": 15000
                      }
                    }
                  }
                }]
              }]
            }
          },
          {
            "name": "agent", /* 暴露健康检查接口，可以使用curl http://127.0.0.1:15020/healthz/ready -v查看 */
            "type": "STATIC",
            "connect_timeout": "0.250s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "prometheus_stats",
              "endpoints": [{
                "lb_endpoints": [{ /* 负载均衡的后端 */
                  "endpoint": {
                    "address":{
                      "socket_address": {
                        "protocol": "TCP",
                        "address": "127.0.0.1",
                        "port_value": 15020
                      }
                    }
                  }
                }]
              }]
            }
          },
          {
            "name": "sds-grpc", /* 配置SDS cluster */
            "type": "STATIC",
            "http2_protocol_options": {},
            "connect_timeout": "1s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "sds-grpc",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "pipe": {
                        "path": "./etc/istio/proxy/SDS" /* 进行SDS的UNIX socket路径，用于在mTLS期间给istio-agent和proxy提供通信 */
                      }
                    }
                  }
                }]
              }]
            }
          },
          {
            "name": "xds-grpc", /* 动态xDS使用的grpc服务器配置 */
            "type": "STRICT_DNS",
            "respect_dns_ttl": true,
            "dns_lookup_family": "V4_ONLY",
            "connect_timeout": "1s",
            "lb_policy": "ROUND_ROBIN",
            "transport_socket": { /* 配置与上游连接的传输socket */
              "name": "envoy.transport_sockets.tls", /* 需要实例化的传输socket名称 */
              "typed_config": {
                "@type": "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext",
                "sni": "istiod.istio-system.svc", /* 创建TLS后端(即SDS服务器)连接时要使用的SNI字符串 */
                "common_tls_context": { /* 配置client和server使用的TLS上下文 */
                  "alpn_protocols": [/* listener暴露的ALPN协议列表，如果为空，则不使用ALPN */
                    "h2"
                  ],
                  "tls_certificate_sds_secret_configs": [/*通过SDS API获取TLS证书的配置 */
                    {
                      "name": "default",
                      "sds_config": { /* 配置sds_config时将会从静态资源加载secret */
                        "resource_api_version": "V3", /* xDS的API版本 */
                        "initial_fetch_timeout": "0s",
                        "api_config_source": { /* SDS API配置，如版本和SDS服务 */
                          "api_type": "GRPC",
                          "transport_api_version": "V3", /* xDS传输协议的API版本 */
                          "grpc_services": [
                            { /* SDS服务器对应上面配置的sds-grpc cluster */
                              "envoy_grpc": { "cluster_name": "sds-grpc" }
                            }
                          ]
                        }
                      }
                    }
                  ],
                  "validation_context": {
                    "trusted_ca": {
                      "filename": "./var/run/secrets/istio/root-cert.pem" /* 本地文件系统的数据源。挂载当前命名空间下的config istio-ca-root-cert，其中的CA证书与istio-system命名空间下的istio-ca-secret中的CA证书相同，用于校验对端istiod的证书 */
                    },
                    "match_subject_alt_names": [{"exact":"istiod.istio-system.svc"}] /* 验证证书中的SAN，即来自istiod的证书 */
                  }
                }
              }
            },
            "load_assignment": {
              "cluster_name": "xds-grpc", /* 可以看到xds-grpc的后端为istiod的15012端口 */
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "socket_address": {"address": "istiod.istio-system.svc", "port_value": 15012}
                    }
                  }
                }]
              }]
            },
            "circuit_breakers": { /* 断路器配置 */
              "thresholds": [
                {
                  "priority": "DEFAULT",
                  "max_connections": 100000,
                  "max_pending_requests": 100000,
                  "max_requests": 100000
                },
                {
                  "priority": "HIGH",
                  "max_connections": 100000,
                  "max_pending_requests": 100000,
                  "max_requests": 100000
                }
              ]
            },
            "upstream_connection_options": {
              "tcp_keepalive": {
                "keepalive_time": 300
              }
            },
            "max_requests_per_connection": 1,
            "http2_protocol_options": { }
          }
                  ,
          {
            "name": "zipkin", /* 分布式链路跟踪zipkin的cluster配置 */
            "type": "STRICT_DNS",
            "respect_dns_ttl": true,
            "dns_lookup_family": "V4_ONLY",
            "connect_timeout": "1s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "zipkin",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "socket_address": {"address": "zipkin.istio-system", "port_value": 9411}
                    }
                  }
                }]
              }]
            }
          }
        ],
    ```

    > 上面使用`type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext `API接口来对传输socket进行配置，`sni`和`common_tls_context` 都属于结构体[UpstreamTlsContext](https://github.com/envoyproxy/envoy/blob/master/api/envoy/extensions/transport_sockets/tls/v3/tls.proto#L27)中的成员变量。

    可以使用`istioctl pc cluster`命令查看静态cluster资源，第一列对应上面的`Cluster.name`，其中`sds-grpc`用于提供[SDS服务](https://www.envoyproxy.io/docs/envoy/latest/configuration/security/secret#secret-discovery-service-sds)，SDS的原理可以参见[官方文档](https://istio.io/latest/docs/concepts/security/#pki)。

    ```shell
    # istioctl pc cluster sleep-856d589c9b-x6szk.default |grep STATIC
    BlackHoleCluster                        -         -          -             STATIC
    agent                                   -         -          -             STATIC
    prometheus_stats                        -         -          -             STATIC
    sds-grpc                                -         -          -             STATIC
    sleep.default.svc.cluster.local         80        http       inbound       STATIC
    ```

  - listener，下面用到了Network filter中的[HTTP connection manager](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#http-connection-manager)

    ```json
        "listeners":[
          {
            "address": { /* listener监听的地址 */
              "socket_address": {
                "protocol": "TCP",
                "address": "0.0.0.0",
                "port_value": 15090
              }
            },
            "filter_chains": [
              {
                "filters": [
                  {
                    "name": "envoy.http_connection_manager",
                    "typed_config": { /* 对扩展API的配置 */
                      "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                      "codec_type": "AUTO", /* 由连接管理器判断使用哪种编解码器 */
                      "stat_prefix": "stats",
                      "route_config": { /* 连接管理器的静态路由表 */
                        "virtual_hosts": [ /* 路由表使用的虚拟主机列表 */
                          {
                            "name": "backend", /* 路由表使用的虚拟主机 */
                            "domains": [ /* 匹配该虚拟主机的域列表 */
                              "*"
                            ],
                            "routes": [ /* 匹配入请求的路由列表，使用第一个匹配的路由 */
                              {
                                "match": { /* 将HTTP地址为/stats/prometheus的请求路由到cluster prometheus_stats */
                                  "prefix": "/stats/prometheus" 
                                },
                                "route": {
                                  "cluster": "prometheus_stats"
                                }
                              }
                            ]
                          }
                        ]
                      },
                      "http_filters": [{ /* 构成filter链的filter，用于处理请求。此处并没有定义任何规则 */
                        "name": "envoy.router",
                        "typed_config": {
                          "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                        }
                      }]
                    }
                  }
                ]
              }
            ]
          },
          {
            "address": {
              "socket_address": {
                "protocol": "TCP",
                "address": "0.0.0.0",
                "port_value": 15021
              }
            },
            "filter_chains": [
              {
                "filters": [
                  {
                    "name": "envoy.http_connection_manager",
                    "typed_config": {
                      "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                      "codec_type": "AUTO",
                      "stat_prefix": "agent",
                      "route_config": { /* 静态路由表配置 */
                        "virtual_hosts": [
                          {
                            "name": "backend",
                            "domains": [
                              "*"
                            ],
                            "routes": [
                              {
                                "match": { /* 将HTTP地址为/healthz/ready的请求路由到cluster agent */
                                  "prefix": "/healthz/ready"
                                },
                                "route": {
                                  "cluster": "agent"
                                }
                              }
                            ]
                          }
                        ]
                      },
                      "http_filters": [{
                        "name": "envoy.router",
                        "typed_config": {
                          "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                        }
                      }]
                    }
                  }
                ]
              }
            ]
          }
        ]
      }
    ```

- tracing，对应上面static_resources里定义的zipkin cluster。

  ```json
    "tracing": {
      "http": {
        "name": "envoy.zipkin",
        "typed_config": {
          "@type": "type.googleapis.com/envoy.config.trace.v3.ZipkinConfig",
          "collector_cluster": "zipkin",
          "collector_endpoint": "/api/v2/spans",
          "collector_endpoint_version": "HTTP_JSON",
          "trace_id_128bit": true,
          "shared_span_context": false
        }
      }
    }
  ```

  基本[流程](https://www.servicemesher.com/istio-handbook/concepts/sidecar-traffic-route.html)如下：

  ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200915133137453-1374419547.png)

### Envoy管理接口获取的完整配置

可以在注入Envoy sidecar的pod中执行`curl -X POST localhost:15000/config_dump`来获取完整的配置信息。可以看到它主要包含`BootstrapConfig`，`ClustersConfig`，`ListenersConfig`，`RoutesConfig`，`SecretsConfig`这5部分。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200915133930217-895287568.png)

- Bootstrap：它与上面由Pilot-agent生成的`envoy-rev0.json`文件中的内容相同，即提供给Envoy proxy的初始化配置，给出了xDS服务器的地址等信息。

- Clusters：在Envoy中，Cluster是一个服务集群，每个cluster包含一个或多个endpoint(可以将cluster近似看作是k8s中的service)。

  从上图可以看出，ClustersConfig包含两种cluster配置：`static_clusters`和`dynamic_active_clusters`。前者中的cluster来自envoy-rev0.json中配置的静态cluster资源，包含`agent`，`prometheus_stats`，`sds-grpc`，`xds-grpc`和`zipkin`；后者是通过xDS接口从istio的控制面获取的动态配置信息，`dynamic_active_clusters`主要分为如下四种类型：

  - BlackHoleCluster：

    ```json
         "cluster": {
          "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster",
          "name": "BlackHoleCluster", /* cluster名称 */
          "type": "STATIC",
          "connect_timeout": "10s",
          "filters": [ /* 出站连接的过滤器配置 */
           {
            "name": "istio.metadata_exchange",
            "typed_config": { /* 对扩展API的配置 */
             "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
             "type_url": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
             "value": {
              "protocol": "istio-peer-exchange"
             }
            }
           }
          ]
         },
    ```

    > `BlackHoleCluster`使用的API类型为[type.googleapis.com/udpa.type.v1.TypedStruct](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/extension.proto.html?highlight=typedstruct#config-core-v3-typedextensionconfig)，表示控制面缺少该扩展的模式定义，client会使用[type_url](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/extension#extension-configuration)指定的API，将内容转换为类型化的配置资源。
    >
    > 在上面可以看到Istio使用了协议`istio-peer-exchange `，服务网格内部的两个Envoy实例之间使用该协议来交互Node Metadata。[NodeMetadata](https://github.com/istio/istio/blob/master/pilot/pkg/model/context.go#L418)的数据结构如下：
    >
    > ```go
    > type NodeMetadata struct {
    > 	// ProxyConfig defines the proxy config specified for a proxy.
    > 	// Note that this setting may be configured different for each proxy, due user overrides
    > 	// or from different versions of proxies connecting. While Pilot has access to the meshConfig.defaultConfig,
    > 	// this field should be preferred if it is present.
    > 	ProxyConfig *NodeMetaProxyConfig `json:"PROXY_CONFIG,omitempty"`
    > 
    > 	// IstioVersion specifies the Istio version associated with the proxy
    > 	IstioVersion string `json:"ISTIO_VERSION,omitempty"`
    > 
    > 	// Labels specifies the set of workload instance (ex: k8s pod) labels associated with this node.
    > 	Labels map[string]string `json:"LABELS,omitempty"`
    > 
    > 	// InstanceIPs is the set of IPs attached to this proxy
    > 	InstanceIPs StringList `json:"INSTANCE_IPS,omitempty"`
    > 
    > 	// Namespace is the namespace in which the workload instance is running.
    > 	Namespace string `json:"NAMESPACE,omitempty"`
    > 
    > 	// InterceptionMode is the name of the metadata variable that carries info about
    > 	// traffic interception mode at the proxy
    > 	InterceptionMode TrafficInterceptionMode `json:"INTERCEPTION_MODE,omitempty"`
    > 
    > 	// ServiceAccount specifies the service account which is running the workload.
    > 	ServiceAccount string `json:"SERVICE_ACCOUNT,omitempty"`
    > 
    > 	// RouterMode indicates whether the proxy is functioning as a SNI-DNAT router
    > 	// processing the AUTO_PASSTHROUGH gateway servers
    > 	RouterMode string `json:"ROUTER_MODE,omitempty"`
    > 
    > 	// MeshID specifies the mesh ID environment variable.
    > 	MeshID string `json:"MESH_ID,omitempty"`
    > 
    > 	// ClusterID defines the cluster the node belongs to.
    > 	ClusterID string `json:"CLUSTER_ID,omitempty"`
    > 
    > 	// Network defines the network the node belongs to. It is an optional metadata,
    > 	// set at injection time. When set, the Endpoints returned to a note and not on same network
    > 	// will be replaced with the gateway defined in the settings.
    > 	Network string `json:"NETWORK,omitempty"`
    > 
    > 	// RequestedNetworkView specifies the networks that the proxy wants to see
    > 	RequestedNetworkView StringList `json:"REQUESTED_NETWORK_VIEW,omitempty"`
    > 
    > 	// PodPorts defines the ports on a pod. This is used to lookup named ports.
    > 	PodPorts PodPortList `json:"POD_PORTS,omitempty"`
    > 
    > 	// TLSServerCertChain is the absolute path to server cert-chain file
    > 	TLSServerCertChain string `json:"TLS_SERVER_CERT_CHAIN,omitempty"`
    > 	// TLSServerKey is the absolute path to server private key file
    > 	TLSServerKey string `json:"TLS_SERVER_KEY,omitempty"`
    > 	// TLSServerRootCert is the absolute path to server root cert file
    > 	TLSServerRootCert string `json:"TLS_SERVER_ROOT_CERT,omitempty"`
    > 	// TLSClientCertChain is the absolute path to client cert-chain file
    > 	TLSClientCertChain string `json:"TLS_CLIENT_CERT_CHAIN,omitempty"`
    > 	// TLSClientKey is the absolute path to client private key file
    > 	TLSClientKey string `json:"TLS_CLIENT_KEY,omitempty"`
    > 	// TLSClientRootCert is the absolute path to client root cert file
    > 	TLSClientRootCert string `json:"TLS_CLIENT_ROOT_CERT,omitempty"`
    > 
    > 	CertBaseDir string `json:"BASE,omitempty"`
    > 
    > 	// IdleTimeout specifies the idle timeout for the proxy, in duration format (10s).
    > 	// If not set, no timeout is set.
    > 	IdleTimeout string `json:"IDLE_TIMEOUT,omitempty"`
    > 
    > 	// HTTP10 indicates the application behind the sidecar is making outbound http requests with HTTP/1.0
    > 	// protocol. It will enable the "AcceptHttp_10" option on the http options for outbound HTTP listeners.
    > 	// Alpha in 1.1, based on feedback may be turned into an API or change. Set to "1" to enable.
    > 	HTTP10 string `json:"HTTP10,omitempty"`
    > 
    > 	// Generator indicates the client wants to use a custom Generator plugin.
    > 	Generator string `json:"GENERATOR,omitempty"`
    > 
    > 	// DNSCapture indicates whether the workload has enabled dns capture
    > 	DNSCapture string `json:"DNS_CAPTURE,omitempty"`
    > 
    > 	// ProxyXDSViaAgent indicates that xds data is being proxied via the agent
    > 	ProxyXDSViaAgent string `json:"PROXY_XDS_VIA_AGENT,omitempty"`
    > 
    > 	// Contains a copy of the raw metadata. This is needed to lookup arbitrary values.
    > 	// If a value is known ahead of time it should be added to the struct rather than reading from here,
    > 	Raw map[string]interface{} `json:"-"`
    > }
    > ```
    >
    > Istio通过一些特定的TCP属性来启用TCP策略和控制(这些属性由Envoy代理生成)，并通过Envoy的Node Metadata来获取这些属性。Envoy使用ALPN隧道和基于前缀的协议来转发Node Metadata到对端的Envoy。Istio定义了一个新的协议`istio-peer-exchange`，由网格中的客户端和服务端的sidecar在TLS协商时进行宣告并确定优先级。启用**istio代理**的两端会通过ALPN协商将协议解析为`istio-peer-exchange`(因此仅限于istio服务网格内的交互)，后续的TCP交互将会按照`istio-peer-exchange`的协议规则进行[交互](https://istio.io/latest/docs/tasks/observability/metrics/tcp-metrics/#tcp-attributes)：
    >
    > 
    >
    > ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200915210742792-369235018.png)

    使用如下命令可以看到cluster `BlackHoleCluster`是没有endpoint的。

    ```shell
    # istioctl pc endpoint sleep-856d589c9b-x6szk.default --cluster BlackHoleCluster
    ENDPOINT     STATUS     OUTLIER CHECK     CLUSTER
    ```

    > 如下内容参考[官方博客](https://istio.io/latest/blog/2019/monitoring-external-service-traffic/#external-and-internal-services)：
    >
    > 对于外部服务，Istio提供了两种管理方式：通过将`global.outboundTrafficPolicy.mode` 设置为`REGISTRY_ONLY`来block所有到外部服务的访问；以及通过将`global.outboundTrafficPolicy.mode`设置为`ALLOW_ANY`来允许所有到外部服务的访问。默认会允许所有到外部服务的访问。
    >
    > **BlackHoleCluster** ：当`global.outboundTrafficPolicy.mode`设置为`REGISTRY_ONLY`时，Envoy会创建一个虚拟的cluster BlackHoleCluster。该模式下，所有到外部服务的访问都会被block(除非为每个服务添加[service entries](https://istio.io/latest/docs/reference/config/networking/service-entry))。为了实现该功能，默认的outbound  listener(监听地址为 `0.0.0.0:15001`)使用[原始目的地](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/service_discovery#original-destination)来设置TCP代理，`BlackHoleCluster` 作为一个静态cluster。由于`BlackHoleCluster` 没有任何endpoint，因此会丢弃所有到外部的流量。此外，Istio会为平台服务的每个端口/协议组合创建唯一的listener，如果对同一端口的外部服务发起请求，则不会命中虚拟listener。这种情况会对route配置进行扩展，添加到`BlackHoleCluster`的路由。如果没有匹配到其他路由，则Envoy代理会直接返回502 HTTP状态码(`BlackHoleCluster`可以看作是路由黑洞)。
    >
    > ```json
    >        {
    >         "name": "block_all",
    >         "domains": [
    >          "*"
    >         ],
    >         "routes": [
    >          {
    >           "match": {
    >            "prefix": "/"
    >           },
    >           "direct_response": {
    >            "status": 502
    >           },
    >           "name": "block_all"
    >          }
    >         ],
    >         "include_request_attempt_count": true
    >        },
    > ```

  - PassthroughCluster：可以看到PassthroughCluster也使用了`istio-peer-exchange`协议来处理TCP。

    ```json
         "cluster": {
          "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster",
          "name": "PassthroughCluster",
          "type": "ORIGINAL_DST", /* 指定type为ORIGINAL_DST，这是一个特殊的cluster */
          "connect_timeout": "10s",
          "lb_policy": "CLUSTER_PROVIDED",
          "circuit_breakers": {
           "thresholds": [
            {
             "max_connections": 4294967295,
             "max_pending_requests": 4294967295,
             "max_requests": 4294967295,
             "max_retries": 4294967295
            }
           ]
          },
          "protocol_selection": "USE_DOWNSTREAM_PROTOCOL",
          "filters": [
           {
            "name": "istio.metadata_exchange", /* 配置使用ALPN istio-peer-exchange协议来交换Node Metadata */
            "typed_config": {
             "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
             "type_url": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
             "value": {
              "protocol": "istio-peer-exchange"
             }
            }
           }
          ]
         },
    ```

    使用如下命令可以看到cluster `PassthroughCluster`是没有endpoint的。

    ```shell
    # istioctl pc endpoint sleep-856d589c9b-x6szk.default --cluster PassthroughCluster
    ENDPOINT     STATUS     OUTLIER CHECK     CLUSTER
    ```

    > **PassthroughCluster** ：当`global.outboundTrafficPolicy.mode`设置为`ALLOW_ANY`时，Envoy会创建一个虚拟的cluster `PassthroughCluster` 。该模式下，会允许所有到外部服务的访问。为了实现该功能，默认的outbound  listener(监听地址为 `0.0.0.0:15001`)使用`SO_ORIGINAL_DST`来配置TCP Proxy，`PassthroughCluster`作为一个静态cluster。
    >
    > `PassthroughCluster` cluster使用[原始目的地负载均衡](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/service_discovery#original-destination)策略来配置Envoy发送到原始目的地的流量。
    >
    > 与`BlackHoleCluster`类似，对于每个基于端口/协议的listener，都会添加虚拟路由，将`PassthroughCluster`作为为默认路由。
    >
    > ```json
    >    {
    >     "name": "allow_any",
    >     "domains": [
    >      "*"
    >     ],
    >     "routes": [
    >      {
    >       "match": {
    >        "prefix": "/"
    >       },
    >       "route": {
    >        "cluster": "PassthroughCluster",
    >        "timeout": "0s",
    >        "max_grpc_timeout": "0s"
    >       },
    >       "name": "allow_any"
    >      }
    >     ],
    >     "include_request_attempt_count": true
    >    },
    > ```

    由于`global.outboundTrafficPolicy.mode`只能配置某一个值，因此`BlackHoleCluster`和`PassthroughCluster`的出现是互斥的，`BlackHoleCluster`和`PassthroughCluster`的路由仅存在istio服务网格内，即注入sidecar的pod中。

    可以使用[Prometheus metrics](https://istio.io/latest/blog/2019/monitoring-external-service-traffic/#passthroughcluster-metrics)来监控到`BlackHoleCluster`和`PassthroughCluster`的访问。

  - inbound cluster：处理入站请求的cluster，对于下面的sleep应用来说，其只有一个本地后端`127.0.0.1:80`，并通过`load_assignment`指定了cluster名称和负载信息。由于该监听器上的流量不会出站，因此下面并没有配置过滤器。

    ```json
     "cluster": {
      "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster",
      "name": "inbound|80|http|sleep.default.svc.cluster.local",
      "type": "STATIC",
      "connect_timeout": "10s",
      "circuit_breakers": {
       "thresholds": [
        {
         "max_connections": 4294967295,
         "max_pending_requests": 4294967295,
         "max_requests": 4294967295,
         "max_retries": 4294967295
        }
       ]
      },
      "load_assignment": { /* 设置入站的cluster的endpoint的负载均衡 */
       "cluster_name": "inbound|80|http|sleep.default.svc.cluster.local",
       "endpoints": [
        {
         "lb_endpoints": [
          {
           "endpoint": {
            "address": {
             "socket_address": {
              "address": "127.0.0.1",
              "port_value": 80
             }
            }
           }
          }
         ]
        }
       ]
      }
     },
    ```

    也可以使用如下命令查看inbound的cluster信息：

    ```shell
    # istioctl pc cluster sleep-856d589c9b-c6xsm.default --direction inbound
    SERVICE FQDN                        PORT     SUBSET     DIRECTION     TYPE       DESTINATION RULE
    sleep.default.svc.cluster.local     80       http       inbound       STATIC
    ```

  - outbound cluster：这类cluster为Envoy节点外的服务，配置如何连接上游。下面的EDS表示该cluster的endpoint来自EDS服务发现。下面给出的outbound cluster是istiod的15012端口上的服务。基本结构如下，`transport_socket_matches`仅在使用TLS才会出现，用于配置与TLS证书相关的信息。

    > 可以使用`istioctl pc endpoint`查看EDS的内容
  >
    > ```shell
  > # istioctl pc endpoint sleep-856d589c9b-rn7dw.default --cluster "outbound|15012||istiod.istio-system.svc.cluster.local"
    > ENDPOINT              STATUS      OUTLIER CHECK     CLUSTER
    > 10.80.3.141:15012     HEALTHY     OK                outbound|15012||istiod.istio-system.svc.cluster.local
    > ```
  
    ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200917162028031-1835006792.png)
  
    具体内容如下：
  
    ```json
    {
     "version_info": "2020-09-15T08:05:54Z/4",
     "cluster": {
      "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster",
      "name": "outbound|15012||istiod.istio-system.svc.cluster.local",
      "type": "EDS", 
      "eds_cluster_config": { /* EDS的配置 */
       "eds_config": {
        "ads": {},
        "resource_api_version": "V3"
       },
       "service_name": "outbound|15012||istiod.istio-system.svc.cluster.local" /* EDS的cluster的可替代名称，无需与cluster名称完全相同 */
      },
      "connect_timeout": "10s",
      "circuit_breakers": { /* 断路器设置 */
       "thresholds": [
        {
         "max_connections": 4294967295,
         "max_pending_requests": 4294967295,
         "max_requests": 4294967295,
         "max_retries": 4294967295
        }
       ]
      },
      "filters": [ /* 设置Node Metadata交互使用的协议为istio-peer-exchange */
       {
        "name": "istio.metadata_exchange",
        "typed_config": {
         "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
         "type_url": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
         "value": {
          "protocol": "istio-peer-exchange"
         }
        }
       }
      ],
      "transport_socket_matches": [ /* 指定匹配的后端使用的带TLS的传输socket */
       {
        "name": "tlsMode-istio", /* match的名称 */
        "match": { /* 匹配后端的条件，注入istio sidecar的pod会打上标签：security.istio.io/tlsMode=istio */
         "tlsMode": "istio"
        },
        "transport_socket": { /* 匹配cluster的后端使用的传输socket的配置 */
         "name": "envoy.transport_sockets.tls",
         "typed_config": {
          "@type": "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext",
          "common_tls_context": { /* 配置client和server端使用的TLS上下文 */
           "alpn_protocols": [ /* 配置交互使用的ALPN协议集，供上游选择 */
            "istio-peer-exchange",
            "istio"
           ],
           "tls_certificate_sds_secret_configs": [ /* 通过SDS API获取TLS证书的配置 */
            {
             "name": "default",
             "sds_config": {
              "api_config_source": {
               "api_type": "GRPC",
               "grpc_services": [ /* SDS的cluster */
                {
                 "envoy_grpc": {
                  "cluster_name": "sds-grpc" /* 为上面静态配置的cluster */
                 }
                }
               ],
               "transport_api_version": "V3"
              },
              "initial_fetch_timeout": "0s",
              "resource_api_version": "V3"
             }
            }
           ],
           "combined_validation_context": { /* 包含一个CertificateValidationContext(即下面的default_validation_context)和SDS配置。当SDS服务返回动态的CertificateValidationContext时，动态和默认的CertificateValidationContext会合并为一个新的CertificateValidationContext来进行校验  */
            "default_validation_context": { /* 配置如何认证对端istiod服务的证书 */
             "match_subject_alt_names": [ /* Envoy会按照如下配置来校验证书中的SAN */
              {
               "exact": "spiffe://new-td/ns/istio-system/sa/istiod-service-account" /* Istio数据面使用serviceaccount进行授权，此处用于校验对端istiod证书中的SAN，参见下面的SDS介绍 */
              }
             ]
            },
            "validation_context_sds_secret_config": { /* SDS配置，也是通过静态的cluster sds-grpc提供SDS API服务 */
             "name": "ROOTCA", /* 用于认证对端的CA证书， */
             "sds_config": {
              "api_config_source": {
               "api_type": "GRPC",
               "grpc_services": [
                {
                 "envoy_grpc": {
                  "cluster_name": "sds-grpc" /* 获取该CA证书的SDS服务器 */
                 }
                }
               ],
               "transport_api_version": "V3"
              },
              "initial_fetch_timeout": "0s",
              "resource_api_version": "V3"
             }
            }
           }
          },
          "sni": "outbound_.15012_._.istiod.istio-system.svc.cluster.local" /* 创建TLS连接时使用的SNI字符串，即TLS的server_name扩展字段中的值 */
         }
        }
       },
       {
        "name": "tlsMode-disabled", /* 如果与没有匹配到的后端(即istio服务网格外的后端)进行通信时，则使用明文方式 */
        "match": {},
        "transport_socket": {
         "name": "envoy.transport_sockets.raw_buffer"
        }
       }
      ]
     },
     "last_updated": "2020-09-15T08:06:23.565Z"
    },
    ```

- Listeners：Envoy使用listener来接收并处理下游发来的请求。与cluster类似，listener也分为静态和动态两种配置。静态配置来自Istio-agent生成的`envoy-rev0.json`文件。动态配置为：

  - virtualOutbound Listener：Istio在注入sidecar时，会通过init容器来设置iptables规则，将所有出站的TCP流量拦截到本地的`15001`端口：

    ```shell
    -A ISTIO_REDIRECT -p tcp -j REDIRECT --to-ports 15001
    ```

    一个istio-agent配置中仅包含一个virtualOutbound listener，可以看到该listener并没有配置`transport_socket`，它的下游流量就是来自本pod的业务容器，并不需要进行TLS校验，直接将流量重定向到15001端口即可，然后转发给和原始目的IP:Port匹配的listener。

    ```json
        {
         "name": "virtualOutbound",
         "active_state": {
          "version_info": "2020-09-15T08:05:54Z/4",
          "listener": {
           "@type": "type.googleapis.com/envoy.config.listener.v3.Listener",
           "name": "virtualOutbound",
           "address": { /* 监听器监听的地址 */
            "socket_address": {
             "address": "0.0.0.0",
             "port_value": 15001
            }
           },
           "filter_chains": [ /* 应用到该监听器的过滤器链 */
            {
             "filters": [ /* 与该监听器建立连接时使用的过滤器，按顺序处理各个过滤器。如果过滤器列表为空，则默认会关闭连接 */
              {
               "name": "istio.stats",
               "typed_config": {
                "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                "type_url": "type.googleapis.com/envoy.extensions.filters.network.wasm.v3.Wasm",
                "value": {
                 "config": { /* Wasm插件配置 */
                  "root_id": "stats_outbound", /* 一个VM中具有相同root_id的一组filters/services会共享相同的RootContext和Contexts，如果该字段为空，所有该字段为空的filters/services都会共享具有相同vm_id的Context(s) */
                  "vm_config": { /* Wasm VM的配置 */
                   "vm_id": "tcp_stats_outbound", /* 使用相同vm_id和code将使用相同的VM */
                   "runtime": "envoy.wasm.runtime.null", /* Wasm运行时，v8或null */
                   "code": {
                    "local": {
                     "inline_string": "envoy.wasm.stats"
                    }
                   }
                  },
                  "configuration": {
                   "@type": "type.googleapis.com/google.protobuf.StringValue",
                   "value": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\"\n}\n"
                  }
                 }
                }
               }
              },
              {
               "name": "envoy.tcp_proxy", /* 处理TCP的过滤器 */
               "typed_config": {
                "@type": "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy",
                "stat_prefix": "PassthroughCluster",
                "cluster": "PassthroughCluster", /* 连接的上游cluster */
                "access_log": [
                 {
                  "name": "envoy.file_access_log",
                  "typed_config": { /* 配置日志的输出格式和路径 */
                   "@type": "type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog",
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
           "hidden_envoy_deprecated_use_original_dst": true,
           "traffic_direction": "OUTBOUND"
          },
          "last_updated": "2020-09-15T08:06:24.066Z"
         }
        },
    ```

    > 上面`envoy.tcp_proxy`过滤器的cluster为`PassthroughCluster`，这是因为将`global.outboundTrafficPolicy.mode`设置为了`ALLOW_ANY`，默认可以访问外部服务。如果`global.outboundTrafficPolicy.mode`设置为了`REGISTRY_ONLY`，则此处将变为cluster `BlackHoleCluster`，默认丢弃所有到外部服务的请求。

    > 上面使用wasm(WebAssembly)来记录遥测信息，Envoy官方文档中目前缺少对wasm的描述，可以参考开源代码的[描述](https://github.com/envoyproxy/envoy/blob/master/api/envoy/extensions/wasm/v3/wasm.proto)。从runtime字段为`null`可以看到并没有启用。可以在安装istio的时候使用[如下参数](https://istio.io/latest/docs/reference/config/proxy_extensions/wasm_telemetry/)来启用基于Wasm的遥测。
    >
    > ```shell
    > $ istioctl install --set values.telemetry.v2.metadataExchange.wasmEnabled=true --set values.telemetry.v2.prometheus.wasmEnabled=true
    > ```
    >
    > 启用之后，与wasm有关的用于遥测的过滤器配置变为了如下内容，可以看到其runtime使用了[envoy.wasm.runtime.v8](https://v8.dev/)。更多参见官方[博客](https://istio.io/latest/blog/2020/wasm-announce/)。
    >
    > ```json
    >           {
    >            "name": "istio.stats",
    >            "typed_config": {
    >             "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
    >             "type_url": "type.googleapis.com/envoy.extensions.filters.network.wasm.v3.Wasm",
    >             "value": {
    >              "config": {
    >               "root_id": "stats_outbound",
    >               "vm_config": { /* wasm虚拟机配置 */
    >                "vm_id": "tcp_stats_outbound",
    >                "runtime": "envoy.wasm.runtime.v8", /* 使用的wasm runtime */
    >                "code": {
    >                 "local": {
    >                  "filename": "/etc/istio/extensions/stats-filter.compiled.wasm" /* 编译后的wasm插件路径 */
    >                 }
    >                },
    >                "allow_precompiled": true
    >               },
    >               "configuration": {
    >                "@type": "type.googleapis.com/google.protobuf.StringValue",
    >                "value": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\"\n}\n"
    >               }
    >              }
    >             }
    >            }
    >           },
    > ```
    >
    > 在istio-proxy容器的`/etc/istio/extensions/`目录下可以看到wasm编译的相关程序，包含用于交换Node Metadata的`metadata-exchange-filter.wasm`和用于遥测的`stats-filter.wasm`，带`compiled`的wasm用于HTTP。
    >
    > ```shell
    > $ ls
    > metadata-exchange-filter.compiled.wasm  metadata-exchange-filter.wasm  stats-filter.compiled.wasm  stats-filter.wasm
    > ```
    >
    > Istio的filter处理[示意图](https://banzaicloud.com/blog/envoy-wasm-filter/)如下：
    >
    > ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200916214116374-1209783360.png)

  - VirtualInbound/Inbound Listener：与virtualOutbound listener类似，通过如下规则将所有入站的TCP流量重定向到15006端口

    ```shell
    -A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-ports 15006
    ```

    下面是一个demo环境中的典型配置，可以看到对于每个监听的地址，都配置了两个过滤器：一个带`transport_socket`，一个不带`transport_socket`，分别处理使用TLS的连接和不使用TLS的连接。主要的入站监听器为：

    - 处理基于IPv4的带TLS 的TCP连接
    - 处理基于IPv4的不带TLS 的TCP连接
    - 处理基于IPv6的带TLS 的TCP连接
    - 处理基于IPv6的不带TLS 的TCP连接
    - 处理基于IPv4的带TLS 的HTTP连接
    - 处理基于IPv4的不带TLS 的HTTP连接
    - 处理基于IPv6的带TLS 的HTTP连接
    - 处理基于IPv6的不带TLS 的HTTP连接
    - 处理业务的带TLS(不带TLS)的连接

    ![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200917145950359-961211851.png)

    下面给出如下内容的inbound listener：

    - 处理基于IPv4的带TLS 的TCP连接
    - 处理基于IPv4的不带TLS 的TCP连接
    - 处理业务的带TLS的连接

    ```json
        {
         "name": "virtualInbound",
         "active_state": {
          "version_info": "2020-09-15T08:05:54Z/4",
          "listener": {
           "@type": "type.googleapis.com/envoy.config.listener.v3.Listener",
           "name": "virtualInbound",
           "address": { /* 该listener绑定的地址和端口 */
            "socket_address": {
             "address": "0.0.0.0",
             "port_value": 15006
            }
           },
           "filter_chains": [
            /* 匹配所有IPV4地址，使用TLS且ALPN为istio-peer-exchange或istio的连接 */
            {
             "filter_chain_match": { /* 将连接匹配到该过滤器链时使用的标准 */
              "prefix_ranges": [ /* 当listener绑定到0.0.0.0/::时匹配的IP地址和前缀长度，下面表示整个网络地址 */
               {
                "address_prefix": "0.0.0.0",
                "prefix_len": 0
               }
              ],
              "transport_protocol": "tls", /* 匹配的传输协议 */
              "application_protocols": [ /* 使用的ALPN */
               "istio-peer-exchange",
               "istio"
              ]
             },
             "filters": [
              {
               "name": "istio.metadata_exchange", /* 交换Node Metadata的配置 */
               "typed_config": {
                "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                "type_url": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
                "value": {
                 "protocol": "istio-peer-exchange"
                }
               }
              },
              {
               "name": "istio.stats", /* 使用wasm进行遥测的配置 */
               "typed_config": {
                "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                "type_url": "type.googleapis.com/envoy.extensions.filters.network.wasm.v3.Wasm",
                "value": {
                 "config": {
                  "root_id": "stats_inbound",
                  "vm_config": {
                   "vm_id": "tcp_stats_inbound",
                   "runtime": "envoy.wasm.runtime.null",
                   "code": {
                    "local": {
                     "inline_string": "envoy.wasm.stats"
                    }
                   }
                  },
                  "configuration": {
                   "@type": "type.googleapis.com/google.protobuf.StringValue",
                   "value": "{\n  \"debug\": \"false\",\n  \"stat_prefix\": \"istio\"\n}\n"
                  }
                 }
                }
               }
              },
              {
               "name": "envoy.tcp_proxy", /* 配置连接上游cluster InboundPassthroughClusterIpv4时的访问日志，InboundPassthroughClusterIpv4 cluster用于处理基于IPv4的HTTP */
               "typed_config": {
                "@type": "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy",
                "stat_prefix": "InboundPassthroughClusterIpv4",
                "cluster": "InboundPassthroughClusterIpv4",
                "access_log": [
                 {
                  "name": "envoy.file_access_log",
                  "typed_config": {
                   "@type": "type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog",
                   "path": "/dev/stdout",
                   "format": "[%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% \"%DYNAMIC_METADATA(istio.mixer:status)%\" \"%UPSTREAM_TRANSPORT_FAILURE_REASON%\" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n"
                  }
                 }
                ]
               }
              }
             ],
             "transport_socket": { /* 匹配TLS的传输socket */
              "name": "envoy.transport_sockets.tls",
              "typed_config": {
               "@type": "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext",
               "common_tls_context": {
                "alpn_protocols": [ /* 监听器使用的ALPN列表 */
                 "istio-peer-exchange",
                 "h2",
                 "http/1.1"
                ],
                "tls_certificate_sds_secret_configs": [ /* 通过SDS API获取证书的配置 */
                 {
                  "name": "default",
                  "sds_config": {
                   "api_config_source": {
                    "api_type": "GRPC",
                    "grpc_services": [
                     {
                      "envoy_grpc": {
                       "cluster_name": "sds-grpc"
                      }
                     }
                    ],
                    "transport_api_version": "V3"
                   },
                   "initial_fetch_timeout": "0s",
                   "resource_api_version": "V3"
                  }
                 }
                ],
                "combined_validation_context": {
                 "default_validation_context": { /* 对对端的证书的SAN进行认证 */
                  "match_subject_alt_names": [ 
                   {
                    "prefix": "spiffe://new-td/"
                   },
                   {
                    "prefix": "spiffe://old-td/"
                   }
                  ]
                 },
                 "validation_context_sds_secret_config": { /* 配置通过SDS API获取证书 */
                  "name": "ROOTCA",
                  "sds_config": {
                   "api_config_source": {
                    "api_type": "GRPC",
                    "grpc_services": [
                     {
                      "envoy_grpc": {
                       "cluster_name": "sds-grpc"
                      }
                     }
                    ],
                    "transport_api_version": "V3"
                   },
                   "initial_fetch_timeout": "0s",
                   "resource_api_version": "V3"
                  }
                 }
                }
               },
               "require_client_certificate": true
              }
             },
             "name": "virtualInbound"
            },
            /* 与上面不同的是，此处匹配不带TLS的连接 */
            {
             "filter_chain_match": {
              "prefix_ranges": [
               {
                "address_prefix": "0.0.0.0",
                "prefix_len": 0
               }
              ]
             },
             "filters": [
              {
               "name": "istio.metadata_exchange",
                ...
              },
              {
               "name": "istio.stats",
    			...
              },
              {
               "name": "envoy.tcp_proxy",
                ...
              }
             ],
             "name": "virtualInbound"
            },
            ...
            /* 应用的监听器，监听端口为HTTP 80端口 */
            {
             "filter_chain_match": {
              "destination_port": 80, /* 匹配的请求的目的端口 */
              "application_protocols": [ /* 匹配的ALPN，仅在使用TLS时使用 */
               "istio",
               "istio-http/1.0",
               "istio-http/1.1",
               "istio-h2"
              ]
             },
             "filters": [
              {
               "name": "istio.metadata_exchange", /* 交换Node Metadata的配置 */
                ...
              },
              {
               "name": "envoy.http_connection_manager", /* HTTP连接管理过滤器 */
               "typed_config": {
                "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                "stat_prefix": "inbound_0.0.0.0_80",
                "route_config": { /* 静态路由表 */
                 "name": "inbound|80|http|sleep.default.svc.cluster.local", /* 路由配置的名称 */
                 "virtual_hosts": [ /* 构成路由表的虚拟主机列表 */
                  {
                   "name": "inbound|http|80", /* 构成路由表的虚拟主机名 */
                   "domains": [ /* 匹配到该虚拟主机的域列表 */
                    "*"
                   ],
                   "routes": [ /* 对入站请求的路由，将路径为"/"的HTTP请求路由到cluster inbound|80|http|sleep.default.svc.cluster.local*/
                    {
                     "match": {
                      "prefix": "/"
                     },
                     "route": {
                      "cluster": "inbound|80|http|sleep.default.svc.cluster.local",
                      "timeout": "0s",
                      "max_grpc_timeout": "0s"
                     },
                     "decorator": {
                      "operation": "sleep.default.svc.cluster.local:80/*"
                     },
                     "name": "default" /* 路由的名称 */
                    }
                   ]
                  }
                 ],
                 "validate_clusters": false
                },
                "http_filters": [ /* HTTP连接过滤器链 */
                 {
                  "name": "istio.metadata_exchange", /* 基于HTTP的Metadata的交换配置 */
    			   ...
                 },
                 {
                  "name": "istio_authn", /* istio的mTLS的默认值 */
                  "typed_config": {
                   "@type": "type.googleapis.com/istio.envoy.config.filter.http.authn.v2alpha1.FilterConfig",
                   "policy": {
                    "peers": [
                     {
                      "mtls": {
                       "mode": "PERMISSIVE"
                      }
                     }
                    ]
                   },
                   "skip_validate_trust_domain": true
                  }
                 },
                 {
                  "name": "envoy.filters.http.cors",
                  "typed_config": {
                   "@type": "type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors"
                  }
                 },
                 {
                  "name": "envoy.fault",
                  "typed_config": {
                   "@type": "type.googleapis.com/envoy.extensions.filters.http.fault.v3.HTTPFault"
                  }
                 },
                 {
                  "name": "istio.stats", /* 基于HTTP的遥测配置 */
                   ...
                 },
                 {
                  "name": "envoy.router",
                  "typed_config": {
                   "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                  }
                 }
                ],
                "tracing": {
                 "client_sampling": {
                  "value": 100
                 },
                 "random_sampling": {
                  "value": 1
                 },
                 "overall_sampling": {
                  "value": 100
                 }
                },
                "server_name": "istio-envoy", /* 设置访问日志格式 */
                "access_log": [
                 {
                  "name": "envoy.file_access_log",
                   ...
                 }
                ],
                "use_remote_address": false,
                "generate_request_id": true,
                "forward_client_cert_details": "APPEND_FORWARD",
                "set_current_client_cert_details": {
                 "subject": true,
                 "dns": true,
                 "uri": true
                },
                "upgrade_configs": [
                 {
                  "upgrade_type": "websocket"
                 }
                ],
                "stream_idle_timeout": "0s",
                "normalize_path": true
               }
              }
             ],
             "transport_socket": { /* TLS传输socket配置 */
              "name": "envoy.transport_sockets.tls",
               ...
             },
             "name": "0.0.0.0_80"
            },
    ```

  - Outbound listener: 下面是到Prometheus服务9092端口的outbound listener。`10.84.30.227`为Prometheus的k8s service地址，指定了后端的cluster `outbound|9092||prometheus-k8s.openshift-monitoring.svc.cluster.local`。`route_config_name`字段指定了该listener使用的route `prometheus-k8s.openshift-monitoring.svc.cluster.local:9092`。

    ```json
        {
         "name": "10.84.30.227_9092",
         "active_state": {
          "version_info": "2020-09-15T08:05:54Z/4",
          "listener": {
           "@type": "type.googleapis.com/envoy.config.listener.v3.Listener",
           "name": "10.84.30.227_9092",
           "address": {
            "socket_address": {
             "address": "10.84.30.227",
             "port_value": 9092
            }
           },
           "filter_chains": [
            {
             "filters": [
              {
               "name": "istio.stats",
               "typed_config": {
                "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                "type_url": "type.googleapis.com/envoy.extensions.filters.network.wasm.v3.Wasm",
                ...
               }
              },
              {
               "name": "envoy.tcp_proxy",/* TCP过滤器设置，设置连接到对应cluster的日志格式 */
               "typed_config": {
                "@type": "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy",
                "stat_prefix": "outbound|9092||prometheus-k8s.openshift-monitoring.svc.cluster.local",
                "cluster": "outbound|9092||prometheus-k8s.openshift-monitoring.svc.cluster.local",
                "access_log": [
                 ...
                ]
               }
              }
             ]
            },
            {
             "filter_chain_match": {
              "application_protocols": [
               "http/1.0",
               "http/1.1",
               "h2c"
              ]
             },
             "filters": [
              {
               "name": "envoy.http_connection_manager", /* 配置HTTP连接 */
               "typed_config": {
                "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                "stat_prefix": "outbound_10.84.30.227_9092",
                "rds": { /* RDS接口配置 */
                 "config_source": {
                  "ads": {},
                  "resource_api_version": "V3"
                 },
                 "route_config_name": "prometheus-k8s.openshift-monitoring.svc.cluster.local:9092" /* 指定路由配置 */
                },
                "http_filters": [
                 {
                  "name": "istio.metadata_exchange",
                  "typed_config": {
                   "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                   "type_url": "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm",
                   ...
                  }
                 },
                 {
                  "name": "istio.alpn",
                  "typed_config": {
                   "@type": "type.googleapis.com/istio.envoy.config.filter.http.alpn.v2alpha1.FilterConfig",
                   "alpn_override": [
                    {
                     "alpn_override": [
                      "istio-http/1.0",
                      "istio"
                     ]
                    },
                    {
                     "upstream_protocol": "HTTP11",
                     "alpn_override": [
                      "istio-http/1.1",
                      "istio"
                     ]
                    },
                    {
                     "upstream_protocol": "HTTP2",
                     "alpn_override": [
                      "istio-h2",
                      "istio"
                     ]
                    }
                   ]
                  }
                 },
                 {
                  "name": "envoy.filters.http.cors",
                  "typed_config": {
                   "@type": "type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors"
                  }
                 },
                 {
                  "name": "envoy.fault",
                  "typed_config": {
                   "@type": "type.googleapis.com/envoy.extensions.filters.http.fault.v3.HTTPFault"
                  }
                 },
                 {
                  "name": "istio.stats",
                  "typed_config": {
                   "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                   "type_url": "type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm",
                   ...
                  }
                 },
                 {
                  "name": "envoy.router",
                  "typed_config": {
                   "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                  }
                 }
                ],
                "tracing": {
                 ...
                },
                "access_log": [
                 {
                  "name": "envoy.file_access_log",
                  ...
                  }
                 }
                ],
                "use_remote_address": false,
                "generate_request_id": true,
                "upgrade_configs": [
                 {
                  "upgrade_type": "websocket"
                 }
                ],
                "stream_idle_timeout": "0s",
                "normalize_path": true
               }
              }
             ]
            }
           ],
           "deprecated_v1": {
            "bind_to_port": false
           },
           "listener_filters": [
            {
             "name": "envoy.listener.tls_inspector",
             "typed_config": {
              "@type": "type.googleapis.com/envoy.extensions.filters.listener.tls_inspector.v3.TlsInspector"
             }
            },
            {
             "name": "envoy.listener.http_inspector",
             "typed_config": {
              "@type": "type.googleapis.com/envoy.extensions.filters.listener.http_inspector.v3.HttpInspector"
             }
            }
           ],
           "listener_filters_timeout": "5s",
           "traffic_direction": "OUTBOUND",
           "continue_on_listener_filters_timeout": true
          },
          "last_updated": "2020-09-15T08:06:23.989Z"
         }
        },
    ```

    从上面的配置可以看出，路由配置位于HttpConnectionManager类型中，因此如果某个listener没有用到HTTP，则不会有对应的route。如下面的istiod15012端口上的服务，提供了基于gRPC协议的XDP和CA的服务(使用TLS)。

    ```json
    {
     "name": "10.84.251.157_15012",
     "active_state": {
      "version_info": "2020-09-16T07:48:42Z/22",
      "listener": {
       "@type": "type.googleapis.com/envoy.config.listener.v3.Listener",
       "name": "10.84.251.157_15012",
       "address": {
        "socket_address": {
         "address": "10.84.251.157",
         "port_value": 15012
        }
       },
       "filter_chains": [
        {
         "filters": [
          {
           "name": "istio.stats",
           ...
          },
          {
           "name": "envoy.tcp_proxy",
           "typed_config": {
            "@type": "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy",
            "stat_prefix": "outbound|15012||istiod.istio-system.svc.cluster.local",
            "cluster": "outbound|15012||istiod.istio-system.svc.cluster.local",
            "access_log": [
             ...
            ]
           }
          }
         ]
        }
       ],
       "deprecated_v1": {
        "bind_to_port": false
       },
       "traffic_direction": "OUTBOUND"
      },
      "last_updated": "2020-09-16T07:49:34.134Z"
     }
    },
    ```

  - Route：Istio的route也分为静态配置和动态配置。静态路由配置与静态监听器，以及inbound 动态监听器中设置的**静态路由配置**(`envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager`中的`route_config`)有关。

    下面看一个与Prometheus 9092端口提供的服务有关的动态路由，路由配置名称route_config.name与上面Prometheus outbound监听器`route_config_name`字段指定的值是相同的。

    ```json
        {
         "version_info": "2020-09-16T07:48:42Z/22",
         "route_config": {
          "@type": "type.googleapis.com/envoy.config.route.v3.RouteConfiguration",
          "name": "prometheus-k8s.openshift-monitoring.svc.cluster.local:9092",
          "virtual_hosts": [
           {
            "name": "prometheus-k8s.openshift-monitoring.svc.cluster.local:9092",
            "domains": [
             "prometheus-k8s.openshift-monitoring.svc.cluster.local",
             "prometheus-k8s.openshift-monitoring.svc.cluster.local:9092",
             "prometheus-k8s.openshift-monitoring",
             "prometheus-k8s.openshift-monitoring:9092",
             "prometheus-k8s.openshift-monitoring.svc.cluster",
             "prometheus-k8s.openshift-monitoring.svc.cluster:9092",
             "prometheus-k8s.openshift-monitoring.svc",
             "prometheus-k8s.openshift-monitoring.svc:9092",
             "10.84.30.227",
             "10.84.30.227:9092"
            ],
            "routes": [
             {
              "match": {
               "prefix": "/"
              },
              "route": { /* 路由到的后端cluster */
               "cluster": "outbound|9092||prometheus-k8s.openshift-monitoring.svc.cluster.local",
               "timeout": "0s",
               "retry_policy": {
                "retry_on": "connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes",
                "num_retries": 2,
                "retry_host_predicate": [
                 {
                  "name": "envoy.retry_host_predicates.previous_hosts"
                 }
                ],
                "host_selection_retry_max_attempts": "5",
                "retriable_status_codes": [
                 503
                ]
               },
               "max_grpc_timeout": "0s"
              },
              "decorator": {
               "operation": "prometheus-k8s.openshift-monitoring.svc.cluster.local:9092/*"
              },
              "name": "default"
             }
            ],
            "include_request_attempt_count": true
           }
          ],
          "validate_clusters": false
         },
         "last_updated": "2020-09-16T07:49:52.551Z"
        },
    ```

整个访问流程简单可以视为：

- Inbound请求：

  ```
  +----------+    +-----------------------+    +-----------------+    +----------+
  | iptables +--->+ virtualInbound:15006  +--->+ Inbound Cluster +--->+ endpoint |
  +----------+    +-----------------------+    +-----------------+    +----------+
  ```

- 出站请求：

  ```
  +----------+    +-----------------------+    +-------------------+
  | iptables +--->+ virtualOutbound:105001+--->+ Outbound Listener +---+
  +----------+    +-----------------------+    +-------------------+   |
                                                                       |
                                                                       |
        +--------------------------------------------------------------+
        |
        |     +-------+    +-----------------+    +----------+
        +---->+ route +--->+Outbound Cluster +--->+ endpoint |
              +-------+    +-----------------+    +----------+
  ```

更多内容可以参考Envoy的[官方文档](https://www.envoyproxy.io/docs/envoy/latest/intro/life_of_a_request)

下面是基于Istio官方BookInfo的一个[访问流程图](https://www.servicemesher.com/istio-handbook/concepts/sidecar-traffic-route.html)，可以帮助理解整个流程。

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200918102434466-1282395165.png)



### SDS

下图来自[这篇文章](https://cloud.tencent.com/developer/article/1680602)

![](https://img2020.cnblogs.com/blog/1334952/202009/1334952-20200922132124590-1783723640.png)

SDS会动态下发两个证书：`default`和`ROOTCA`。前者表示本服务使用的证书，证书中的SAN使用了该服务对应的命名空间下的serviceaccount；后者为集群的CA，通过将configmap `istio-ca-root-cert`挂载到服务的pod中，它与`istio-system`命名空间中的secret `istio-ca-secret`相同，可以用于认证各个服务的证书(`default`)。因此不同的服务使用的`default`证书是不同的，但使用的`ROOTCA`证书是相同的。

```json
  {
   "@type": "type.googleapis.com/envoy.admin.v3.SecretsConfigDump",
   "dynamic_active_secrets": [
    {
     "name": "default",
     "version_info": "09-21 20:10:52.178",
     "last_updated": "2020-09-21T20:10:52.446Z",
     "secret": {
      "@type": "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.Secret",
      "name": "default", /* 服务使用的证书 */
      "tls_certificate": {
       "certificate_chain": {
        "inline_bytes": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JS..."
       },
       "private_key": {
        "inline_bytes": "W3JlZGFjdGVkXQ=="
       }
      }
     }
    },
    {
     "name": "ROOTCA",
     "version_info": "2020-09-15 08:05:53.174860205 +0000 UTC m=+1.073140142",
     "last_updated": "2020-09-15T08:05:53.275Z",
     "secret": {
      "@type": "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.Secret",
      "name": "ROOTCA", /* 验证服务证书使用的CA证书 */
      "validation_context": {
       "trusted_ca": {
        "inline_bytes": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t..."
       }
      }
     }
    }
   ]
  }
```

将default证书导出，查看该证书，可以看到其使用的SAN为`spiffe://new-td/ns/default/sa/sleep`，用到了`default`命名空间下的serviceaccount `sleep`，提供了该服务的身份标识。

```shell
# openssl x509 -in ca-chain.crt -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            37:31:36:d4:25:64:18:10:27:47:75:79:6c:ff:21:3a
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: O=cluster.local
        Validity
            Not Before: Sep 21 08:34:33 2020 GMT
            Not After : Sep 22 08:34:33 2020 GMT
        Subject:
        Subject Public Key Info:
            ...

            X509v3 Subject Alternative Name: critical
                URI:spiffe://new-td/ns/default/sa/sleep
    Signature Algorithm: sha256WithRSAEncryption
         ...
```

到处`default`和`ROOTCA`证书，分别对应下面的pod.crt和root-cat.crt，可以看到，能够使用`ROOTCA`来验证`default`。

```shell
# openssl verify -CAfile root-ca.crt pod.crt
ca-chain.crt: OK
```

在 Istio的sidecar配置中，有两处需要通过 SDS 来配置证书：

- Inbound Listener：Inbound Listener用于接收来自下游的连接，在`DownstreamTlsContext`中配置了对下游连接的认证。其指定了用于验证下游连接的ROOTCA证书。`match_subject_alt_names`中指定的通配SAN是因为在安装Istio时通过参数`values.global.trustDomain`指定了信任域。

  ```json
             "common_tls_context": {
              "alpn_protocols": [
               "istio-peer-exchange",
               "h2",
               "http/1.1"
              ],
              "tls_certificate_sds_secret_configs": [
               {
                "name": "default", /* 服务器使用的证书名称，由SDS下发 */
                "sds_config": {
                 "api_config_source": {
                  "api_type": "GRPC",
                  "grpc_services": [
                   {
                    "envoy_grpc": {
                     "cluster_name": "sds-grpc"
                    }
                   }
                  ],
                  "transport_api_version": "V3"
                 },
                 "initial_fetch_timeout": "0s",
                 "resource_api_version": "V3"
                }
               }
              ],
              "combined_validation_context": {
               "default_validation_context": {
                "match_subject_alt_names": [ /* 指定的信任域 */
                 {
                  "prefix": "spiffe://new-td/"
                 },
                 {
                  "prefix": "spiffe://old-td/"
                 }
                ]
               },
               "validation_context_sds_secret_config": {
                "name": "ROOTCA", /* 用于认证下游连接的CA证书 */
                "sds_config": {
                 "api_config_source": {
                  "api_type": "GRPC",
                  "grpc_services": [
                   {
                    "envoy_grpc": {
                     "cluster_name": "sds-grpc"
                    }
                   }
                  ],
                  "transport_api_version": "V3"
                 },
                 "initial_fetch_timeout": "0s",
                 "resource_api_version": "V3"
                }
               }
              }
             },
  ```

- Outbound Cluster：outbound cluster(下例为`outbound|80||sleep.default.svc.cluster.local`)作为下游配置，也需要对服务端的证书进行验证。其`UpstreamTlsContext`中的配置如下。由于上有服务为`sleep.default.svc.cluster.local`，因此在`match_subject_alt_names`字段中指定了验证的服务端的SAN。

  ```json
            "common_tls_context": {
             "alpn_protocols": [
              "istio-peer-exchange",
              "istio"
             ],
             "tls_certificate_sds_secret_configs": [
              {
               "name": "default",
               "sds_config": {
                "api_config_source": {
                 "api_type": "GRPC",
                 "grpc_services": [
                  {
                   "envoy_grpc": {
                    "cluster_name": "sds-grpc"
                   }
                  }
                 ],
                 "transport_api_version": "V3"
                },
                "initial_fetch_timeout": "0s",
                "resource_api_version": "V3"
               }
              }
             ],
             "combined_validation_context": {
              "default_validation_context": {
               "match_subject_alt_names": [ /* 上游服务使用的证书中的SAN */
                {
                 "exact": "spiffe://new-td/ns/default/sa/sleep"
                }
               ]
              },
              "validation_context_sds_secret_config": {
               "name": "ROOTCA",
               "sds_config": {
                "api_config_source": {
                 "api_type": "GRPC",
                 "grpc_services": [
                  {
                   "envoy_grpc": {
                    "cluster_name": "sds-grpc"
                   }
                  }
                 ],
                 "transport_api_version": "V3"
                },
                "initial_fetch_timeout": "0s",
                "resource_api_version": "V3"
               }
              }
             }
            },
  ```

可以看到，作为服务端，仅需要使用ROOTCA证书对客户端进行证书校验即可(如果没有指定信任域)；作为客户端，需要使用ROOTCA证书对服务端证书进行校验，也需要对服务端使用的证书中的SAN进行校验。

gateway上，如果需要针对服务网格外部的服务进行TLS双向认证，可以参考[Traffic Management](https://preliminary.istio.io/latest/docs/tasks/traffic-management/)。

### 参考

- [Sidecar 流量路由机制分析](https://www.servicemesher.com/istio-handbook/concepts/sidecar-traffic-route.html)
- [WebAssembly在Envoy与Istio中的应用](https://www.servicemesher.com/blog/redefining-extensibility-in-proxies/)
- [Istio1.5 & Envoy 数据面 WASM 实践](https://www.servicemesher.com/blog/202004-istio-envoy-wasm/)
- [How to write WASM filters for Envoy and deploy it with Istio](https://banzaicloud.com/blog/envoy-wasm-filter/)
- [Implementing Filters in Envoy](https://medium.com/@alishananda/implementing-filters-in-envoy-dcd8fc2d8fda)
- [一文带你彻底厘清 Isito 中的证书工作机制](https://cloud.tencent.com/developer/article/1680602)

