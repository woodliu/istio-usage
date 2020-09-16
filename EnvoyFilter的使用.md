EnvoyFilter的使用

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

  - clusters：

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
            "transport_socket": { /* 设置与上游连接的传输socket */
              "name": "envoy.transport_sockets.tls", /* 需要实例化的传输socket名称 */
              "typed_config": {
                "@type": "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext",
                "sni": "istiod.istio-system.svc", /* 创建TLS后端(即SDS服务器)连接时要使用的SNI字符串 */
                "common_tls_context": { /* 配置client和server使用的TLS上下文 */
                  "alpn_protocols": [ /* listener暴露的ALPN协议列表 */
                    "h2"
                  ],
                  "tls_certificate_sds_secret_configs": [ /* 通过SDS API获取TLS证书的配置 */
                    {
                      "name": "default",
                      "sds_config": { /* 配置sds_config时将会从静态资源加载secret */
                        "resource_api_version": "V3", /* xDS的API版本 */
                        "initial_fetch_timeout": "0s",
                        "api_config_source": {
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
                    "typed_config": {
                      "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                      "codec_type": "AUTO", /* 由连接管理器判断使用哪种编解码器 */
                      "stat_prefix": "stats",
                      "route_config": { /* 连接管理器的静态路由表 */
                        "virtual_hosts": [ /* 构成路由表的虚拟主机数组 */
                          {
                            "name": "backend", /* 在发送某些统计信息时使用的逻辑名称 */
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
                      "http_filters": [{ /* 构成filter链的filter，用于处理请求 */
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
                      "route_config": {
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
          "filters": [
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
          ]
         },
    ```

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

  - PassthroughCluster：

    ```json
         "cluster": {
          "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster",
          "name": "PassthroughCluster",
          "type": "ORIGINAL_DST",
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
            "name": "istio.metadata_exchange",
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

  - inbound cluster：处理入站请求的cluster，对于下面的sleep应用来说，其只有一个本地后端`127.0.0.1:80`

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

  - outbound cluster：这类cluster为Envoy节点外的服务，配置如何连接上游。下面的EDS表示该cluster的endpoint来自EDS服务发现。下面给出的outbound cluster是istiod的15012端口上的服务。

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
      "filters": [ /* 过滤器设置，设置Node Metadata交互使用的协议为istio-peer-exchange */
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
      "transport_socket_matches": [ /* 指定匹配的后端使用的传输socket */
       {
        "name": "tlsMode-istio", /* match的名称 */
        "match": { /* 匹配后端的条件，注入istio sidecar的pod会打上标签：security.istio.io/tlsMode=istio */
         "tlsMode": "istio"
        },
        "transport_socket": { /* 匹配的后端使用的传输socket的配置 */
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
               "exact": "spiffe://new-td/ns/istio-system/sa/istiod-service-account" /* 与Istio配置的serviceaccount授权策略相同 */
              }
             ]
            },
            "validation_context_sds_secret_config": { /* SDS配置，也是通过静态的cluster sds-grpc提供SDS API服务 */
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

  - virtualOutbound Listener：Istio在注入sidecar时，会通过init容器来设置iptables规则，将所有对外的TCP流量拦截到本地的15001端口：

    ```
    -A ISTIO_REDIRECT -p tcp -j REDIRECT --to-ports 15001
    ```

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
             "filters": [ /* 与该监听器建立连接时使用的过滤器 */
              {
               "name": "istio.stats",
               "typed_config": {
                "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                "type_url": "type.googleapis.com/envoy.extensions.filters.network.wasm.v3.Wasm",
                "value": {
                 "config": {
                  "root_id": "stats_outbound",
                  "vm_config": {
                   "vm_id": "tcp_stats_outbound",
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
               "name": "envoy.tcp_proxy",
               "typed_config": {
                "@type": "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy",
                "stat_prefix": "PassthroughCluster",
                "cluster": "PassthroughCluster",
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

    type.googleapis.com/envoy.extensions.filters.network.wasm.v3.Wasm  参见：https://www.tetrate.io/blog/introducing-getenvoy-extension-toolkit-for-webassembly-based-envoy-extensions/

  - VirtualInbound Listener：

    ```
    -A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-ports 15006
    ```

    

  - 

  



### 参考

- [Sidecar 流量路由机制分析](https://www.servicemesher.com/istio-handbook/concepts/sidecar-traffic-route.html)